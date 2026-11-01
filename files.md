# Proposal: files, modes, links, and the process

2026-11-01 · Anton · Draft

## Summary

Close the gaps a shell script meets when ported to Ant's `File` and `Os` namespaces:

- **Octal literals.** `0o600`, so a mode reads as it does in every shell and manual.
- **Temporary files.** `File.tempFile`, created empty with mode `0o600`. `File.tempDir` promises
  `0o700`. Both take a pattern and a directory.
- **Modes.** `File.chmod`, and `mode` in `File.Info`. New files and directories follow the umask as
  the shell's do.
- **Copies.** `File.copy` for one file and `File.copyAll` for a tree.
- **Symbolic links.** `File.symlink`, `File.readLink`, `File.realPath`, `stat` that can stop at a
  link, and `isSymlink` in `File.Info`. Every existing function says what it does with a link.
- **Times.** `File.touch`.
- **The process.** `Os.pid`, `Os.uid`, `Os.home`, `Os.hostname`.
- **Open:** exclusive creation and locks (recommended: `File.createNew` and `File.lock`), `copyAll`
  onto an existing empty directory, and `Os.configDir`, `Os.cacheDir` and `File.isTerminal`.

```ant
// On Unix, a secret is never readable by others, not even for a moment.
var tmp = try File.tempFile(".token-*", dir: Path.dir(path))
defer { File.remove(tmp) }      // Err after the rename, and discarded
try File.write(tmp, token)      // still 0o600: a write keeps an existing file's mode
try File.rename(tmp, path)      // atomic on Unix: readers see the old file or the new one

try File.chmod("deploy.ant", 0o755)
var info = try File.stat("deploy.ant")
println(String.fromInt(info.mode, base: 8))   // 755
```

Globbing is left to its own entry in `ant/TODO.md`.

## Motivation

Ant is meant to replace shell scripts. Today these lines have no Ant equivalent:

| Shell                                | Ant today                                                         |
| ------------------------------------ | ----------------------------------------------------------------- |
| `tmp=$(mktemp)`                      | `File.tempDir` and a file inside it: two names, two cleanups      |
| `install -m 600 /dev/null secret`    | none: every file is created `0o644`                               |
| `chmod +x run.sh`                    | none                                                              |
| `[[ -x run.sh ]]`, `stat -c %a`      | none: `File.Info` has no mode                                     |
| `cp a b`, `cp -R src dst`            | `File.readBytes` and `File.writeBytes`; a tree by hand; mode lost |
| `ln -s target link`, `readlink link` | `Cmd.run(Cmd.new("ln", ...))`                                     |
| `[[ -L path ]]`, `realpath path`     | none: `File.stat` follows links                                   |
| `touch stamp`, `touch -d @0 file`    | none                                                              |
| `$$`, `$EUID`, `$HOME`, `hostname`   | `Os.env("HOME")`, which is unset on Windows; nothing for the rest |
| `set -o noclobber`, `mkdir lock`     | none: `File.write` replaces, `File.mkdir` is `Ok` when it exists  |

The secret case is the sharpest. A script that writes an API token with `File.write` leaves it
`0o644`, readable by every user on the machine, and has no way to fix that.

## Prior art

### Octal literals

| Language   | Literal        | Legacy `0644`                         |
| ---------- | -------------- | ------------------------------------- |
| Go         | `0o644` (1.13) | still octal                           |
| Rust, Zig  | `0o644`        | decimal 644                           |
| Python 3   | `0o644`        | syntax error                          |
| JavaScript | `0o644`        | octal in sloppy mode, error in strict |
| Swift      | `0o644`        | decimal 644                           |
| Java, C    | `0644`         | that is the octal form                |
| Kotlin, C# | none           | —                                     |

Some libraries use strings instead: Java's `PosixFilePermissions.fromString("rw-------")`, Ruby's
`FileUtils.chmod("u=rw,go=", path)`. Those are checked only when the program runs. `0o` is the form
the newer languages agree on, and it sits beside Ant's `0x` and `0b`.

### Temporary files

| Language | Function                                                                 | Returns            | Mode              |
| -------- | ------------------------------------------------------------------------ | ------------------ | ----------------- |
| Go       | `os.CreateTemp(dir, "report-*.csv")`, `os.MkdirTemp`                     | open file / path   | `0o600` / `0o700` |
| Python   | `tempfile.mkstemp(suffix, prefix, dir)`, `mkdtemp`, `NamedTemporaryFile` | fd and path / path | `0o600` / `0o700` |
| Deno     | `Deno.makeTempFile({ prefix, suffix, dir })`, `makeTempDir`              | path               | `0o600` / `0o700` |
| Node     | `fs.mkdtemp(prefix)` only                                                | path               | `0o700`           |
| Java     | `Files.createTempFile(dir, prefix, suffix)`                              | path               | `0o600` on POSIX  |
| C#       | `Path.GetTempFileName()`                                                 | path               | `0o600` (.NET 7+) |
| Rust     | none in `std`; the `tempfile` crate                                      | handle             | `0o600`           |
| Shell    | `mktemp [-d] [-p dir] name.XXXXXX`                                       | path               | `0o600` / `0o700` |

Every one creates the file. None only invents a name, since a name that is free now can be taken
before the program uses it. Every one uses `0o600` and `0o700`. Most return a path. Go's pattern,
whose last `*` is replaced by the random part, handles a suffix without a second parameter. Python's
`NamedTemporaryFile` and the `tempfile` crate delete the file when it is closed or dropped. Go,
Deno, Java and the shell leave that to the program.

### A mode when a file is written

| Language | Mode at creation                                              | Existing file      |
| -------- | ------------------------------------------------------------- | ------------------ |
| Go       | `os.WriteFile(name, data, perm)`, `os.OpenFile(..., perm)`    | keeps its mode     |
| Node     | `fs.writeFile(path, data, { mode })`                          | keeps its mode     |
| Deno     | `Deno.writeTextFile(path, text, { mode })`                    | keeps its mode¹    |
| Python   | `os.open(path, flags, mode)`; `open()` has none               | keeps its mode     |
| Rust     | `OpenOptionsExt::mode` on Unix; `fs::write` has none          | keeps its mode     |
| Java     | `Files.createFile(path, attrs)`; `Files.writeString` has none | —                  |
| Shell    | `install -m 600`; `umask 077` before `>`                      | `>` keeps its mode |

¹ Deno documents `mode` as applying "when creating"; its `chmod` is separate.

Everywhere, the mode applies only when the file is created, and the umask is subtracted from it. So
`write(path, secret, mode: 0o600)` over an existing `0o644` file leaves it `0o644`. That is the
classic mistake a mode parameter invites. Writing a temporary file and renaming it over the target
is the pattern that gets the mode right whether the target exists or not.

### Copying

| Language | One file                                               | A tree                                | Replaces the target           | Keeps                        |
| -------- | ------------------------------------------------------ | ------------------------------------- | ----------------------------- | ---------------------------- |
| Go       | none: `io.Copy` between two files                      | `os.CopyFS(dir, fsys)` (1.23)         | no: fails on an existing file | execute bits                 |
| Python   | `shutil.copyfile`, `copy`, `copy2`; `Path.copy` (3.14) | `shutil.copytree`; `Path.copy` (3.14) | file yes, tree no             | `copy`: mode; `copy2`: times |
| Node     | `fs.copyFile(src, dst, flags)`                         | `fs.cp(src, dst, { recursive })`      | yes, unless `COPYFILE_EXCL`   | mode; times on request       |
| Deno     | `Deno.copyFile`                                        | none                                  | yes                           | mode                         |
| Rust     | `fs::copy`                                             | none                                  | yes                           | mode                         |
| Java     | `Files.copy(src, dst, options)`                        | none                                  | no, unless `REPLACE_EXISTING` | `COPY_ATTRIBUTES`            |
| C#       | `File.Copy(src, dst, overwrite = false)`               | none                                  | no by default                 | —                            |
| Shell    | `cp`                                                   | `cp -R`                               | yes                           | mode; `-p` adds times        |

Two things vary: whether the target is replaced, and what metadata travels. The scripting languages
and the shell replace a file. Java and C# refuse unless asked. Copying the mode is nearly universal.
Copying times is opt-in everywhere it exists. `cp -R` recreates links as links, while `copytree`
follows them by default.

### Copying a tree onto an existing directory

| Tool                | `to` does not exist | `to` is an existing directory                                                    |
| ------------------- | ------------------- | -------------------------------------------------------------------------------- |
| `cp -R src dst`     | `dst` becomes `src` | copies **into** it, as `dst/src`; `cp -R src/. dst` copies the contents          |
| BSD `cp -R src/`    | same                | the trailing slash copies the contents; GNU `cp` ignores it                      |
| `rsync -a src/ dst` | same                | merges, replacing files; old files stay unless `--delete`                        |
| Ruby `cp_r`         | same                | into, as `cp -R`                                                                 |
| Python `copytree`   | same                | `FileExistsError`; `dirs_exist_ok=True` (3.8) merges, replacing files            |
| Node `fs.cp`        | same                | merges; replaces files unless `force: false`, `errorOnExist: true` makes it fail |
| Go `os.CopyFS`      | same                | merges directories; an existing file is `fs.ErrExist`, after a partial copy      |
| Rust `fs_extra`     | same                | into, or the contents with `content_only`; `overwrite` and `skip_exist` options  |

There are three behaviours. **Into** (`cp`, Ruby) changes meaning with what is on disk: the same
command makes `dst` on the first run and `dst/src` on the second, and the trailing-slash rule
differs between BSD and GNU. **Merge** (rsync, Node, Go, Python's opt-in) keeps files that the
source no longer has, so the result is neither tree. Getting a real mirror needs rsync's `--delete`.
Go's version can also stop halfway through on the first file that exists. **Refuse** (Python's
default) is the only one where the result never depends on what was there before.

The common case that refusing gets in the way of is a directory the program has just made: a
`tempDir`, a `Test.tempDir`, or a `mkdir` for an output directory. That directory is empty, and
copying into an empty directory gives the same tree as copying to a new path. The only difference is
the root directory's mode.

### Exclusive creation

| Language | Spelling                                              | How the program tells "exists" |
| -------- | ----------------------------------------------------- | ------------------------------ |
| POSIX    | `open(path, O_CREAT \| O_EXCL)`                       | `EEXIST`                       |
| Go       | `os.OpenFile(path, O_CREATE\|O_EXCL\|O_WRONLY, perm)` | `errors.Is(err, fs.ErrExist)`  |
| Python   | `open(path, "x")`                                     | `FileExistsError`              |
| Node     | `{ flag: "wx" }`                                      | `err.code === "EEXIST"`        |
| Deno     | `{ createNew: true }`                                 | `Deno.errors.AlreadyExists`    |
| Rust     | `File::create_new` (1.77), `OpenOptions::create_new`  | `ErrorKind::AlreadyExists`     |
| Java     | `StandardOpenOption.CREATE_NEW`, `Files.createFile`   | `FileAlreadyExistsException`   |
| C#       | `FileMode.CreateNew`                                  | `IOException` with an HResult  |
| Shell    | `set -o noclobber`, then `>`; `mkdir` as a lock       | the exit status                |

`O_EXCL` also fails when the path is a symbolic link, even one whose target does not exist. That is
why it is the safe way to create a file in a shared directory such as `/tmp`. It is atomic on local
file systems and on NFS version 3 and later. Windows has the same thing as `CREATE_NEW`. Every
language pairs it with an error the program can tell apart from "permission denied".

Exclusive creation has a well-known limit: a file outlives the process that made it. A pid file left
behind by a crash, a `kill -9` or a power cut makes every later run report "already running" until
someone removes it. The usual fixes are to read the pid and check that the process is still alive,
which races with pid reuse, or to use a lock that the kernel releases when the process dies:

| Language | Advisory lock on an open file                                               | Released                    |
| -------- | --------------------------------------------------------------------------- | --------------------------- |
| POSIX    | `flock(fd, LOCK_EX \| LOCK_NB)`; `fcntl(F_SETLK)` for byte ranges           | on close and at exit        |
| Rust     | `File::lock`, `lock_shared`, `try_lock`, `try_lock_shared`, `unlock` (1.89) | on close                    |
| Deno     | `FsFile.lock(exclusive)`, `unlock`                                          | on close                    |
| Java     | `FileChannel.lock`, `tryLock`, returning a `FileLock`                       | on release or channel close |
| C#       | `FileStream.Lock(position, length)`; `FileShare.None` when opening          | on unlock or close          |
| Python   | `fcntl.flock` on Unix; `msvcrt.locking` on Windows                          | on close                    |
| Go       | none in `std`: `syscall.Flock`; `cmd/go` keeps a private `lockedfile`       | on close                    |
| Node     | none: the `proper-lockfile` package uses `mkdir` and stale-lock timeouts    | —                           |
| Shell    | `flock(1)` from util-linux, absent on macOS                                 | when the command ends       |

Locks have subtleties of their own. `fcntl` locks belong to the process and are dropped when _any_
descriptor for the file is closed, which is why Rust and most libraries use `flock`, whose lock
belongs to the open file. Locks are advisory on Unix, so only programs that ask for them are kept
out. On Windows `LockFileEx` is mandatory: other processes' reads and writes of the locked range
fail. Linux emulates `flock` on NFS with `fcntl` locks.

A third need is to publish a whole file under a name only if the name is free, so readers never see
it half-written. `O_EXCL` cannot do this, since the file exists and is empty while it is being
written. Linux has `renameat2(RENAME_NOREPLACE)` (3.15) and macOS `renamex_np(RENAME_EXCL)`.
Windows' `MoveFileEx` without `MOVEFILE_REPLACE_EXISTING` refuses an existing target. The portable
Unix trick is `link(tmp, target)`, which fails with `EEXIST`, followed by `unlink(tmp)`. No
mainstream standard library offers it as one call.

### Setting times

| Language | Function                                                      | Creates when missing |
| -------- | ------------------------------------------------------------- | -------------------- |
| Go       | `os.Chtimes(name, atime, mtime)`                              | no                   |
| Python   | `os.utime(path, (atime, mtime))`; `Path.touch(exist_ok=True)` | `touch` yes          |
| Node     | `fs.utimes(path, atime, mtime)`, `fs.lutimes`                 | no                   |
| Deno     | `Deno.utime(path, atime, mtime)`                              | no                   |
| Rust     | `File::set_modified`, `File::set_times` (1.75), on a handle   | no                   |
| Java     | `Files.setLastModifiedTime(path, time)`                       | no                   |
| C#       | `File.SetLastWriteTime(path, time)`                           | no                   |
| Ruby     | `FileUtils.touch(path, mtime:)`, `File.utime`                 | `touch` yes          |
| Shell    | `touch [-d time] path`                                        | yes, unless `-c`     |

Two shapes: a setter for both times that requires the file, and `touch`, which creates the file when
it is missing and never truncates it. Scripts reach for `touch`: stamp files, invalidating a build,
making sure a log exists.

### Symbolic links

| Language | Create                                                          | Read                     | Stat without following                          | Resolve                            |
| -------- | --------------------------------------------------------------- | ------------------------ | ----------------------------------------------- | ---------------------------------- |
| Go       | `os.Symlink(old, new)`                                          | `os.Readlink`            | `os.Lstat`                                      | `filepath.EvalSymlinks`            |
| Python   | `os.symlink(src, dst)`                                          | `os.readlink`            | `os.stat(p, follow_symlinks=False)`, `os.lstat` | `os.path.realpath`, `Path.resolve` |
| Node     | `fs.symlink(target, path, type)`                                | `fs.readlink`            | `fs.lstat`                                      | `fs.realpath`                      |
| Deno     | `Deno.symlink(old, new, { type })`                              | `Deno.readLink`          | `Deno.lstat`                                    | `Deno.realPath`                    |
| Rust     | `unix::fs::symlink`; `windows::fs::symlink_file`, `symlink_dir` | `fs::read_link`          | `fs::symlink_metadata`                          | `fs::canonicalize`                 |
| Java     | `Files.createSymbolicLink(link, target)`                        | `Files.readSymbolicLink` | `LinkOption.NOFOLLOW_LINKS`                     | `Path.toRealPath`                  |
| Shell    | `ln -s target link`                                             | `readlink`               | `[[ -L ]]`, `stat` without `-L`                 | `realpath`                         |

Everyone but Java puts the target first, as `ln -s` does. Windows needs to know whether a link
points at a file or a directory: Node, Deno and Rust ask, while Go decides from the target as it
exists when the link is made. Creating a link on Windows needs Developer Mode or an administrator.

### The process

| Need           | Go                            | Python                    | Node                           | Deno                          | Rust              |
| -------------- | ----------------------------- | ------------------------- | ------------------------------ | ----------------------------- | ----------------- |
| process id     | `os.Getpid()`                 | `os.getpid()`             | `process.pid`                  | `Deno.pid`                    | `process::id()`   |
| user id        | `os.Geteuid()`, −1 on Windows | `os.geteuid()`, Unix only | `process.geteuid()`, Unix only | `Deno.uid()`, null on Windows | `libc`            |
| home directory | `os.UserHomeDir()`            | `Path.home()`             | `os.homedir()`                 | none                          | `env::home_dir()` |
| host name      | `os.Hostname()`               | `socket.gethostname()`    | `os.hostname()`                | `Deno.hostname()`             | none in `std`     |

## Design

### Octal literals

An integer literal may be `0o` octal: `0o644`, `0o7_7_7`. The prefix is lowercase, as `0x` and `0b`
are, so `0O644` is an error. `007` stays an error. `String.fromInt(m, base: 8)` prints a mode and
`String.toInt(s, base: 8)` parses one, as they already can.

### Modes

```
type Info = { name: string, size: int, isDir: bool, isSymlink: bool, modified: Time.Instant, mode: int }

File.chmod(path: string, mode: int): Result[void, Error]   // follows a link
```

- **What `mode` holds.** The permission bits and the setuid, setgid and sticky bits: `0o4000`,
  `0o2000` and `0o1000`, at most `0o7777`. The file's type is in `isDir` and `isSymlink`, not in
  `mode`. `chmod` with a mode outside `0 ..= 0o7777` is `Err`.
- **New files and directories** are created as the shell creates them. A file gets `0o666` and a
  directory `0o777`, less the process umask. Under the usual umask `022` that is `0o644` and
  `0o755`, so nothing changes for most users. A user whose umask is `002`, or `077`, gets the files
  they asked for. This covers `write`, `writeBytes`, `create`, `append`, `mkdir` and `touch`.
  `chmod`, `copy` and `tempFile` set their modes exactly and ignore the umask.
- **An existing file keeps its mode** when `write`, `writeBytes`, `create` or `append` replaces or
  extends it.
- **There is no mode parameter** on `write` or `create`. It would apply only when the file is new,
  which is the mistake the prior art shows. A file that must never be readable by others is written
  as a temporary file in the same directory and renamed, as in the Summary. A mode that may be
  briefly wider is `write` followed by `chmod`.
- **The guarantees are Go's, and so Unix's.** `0o600` keeps a file private on Unix. On Windows modes
  do not control access at all; an access control list does, and a new file inherits its
  directory's. Like Go's `os.CreateTemp`, Ant sets no ACL. The default `%TEMP%`, under the user's
  profile, is private to the user, so a `tempFile` there is too. A `tempFile` made elsewhere is as
  private as that directory. `rename` replaces the target atomically on Unix. Go's `os.Rename`
  promises no atomicity on other systems, even within one directory, and neither does Ant.
- **On Windows**, only the read-only attribute exists. `stat` gives a file `0o444` when it is
  read-only and `0o666` otherwise, and a directory has `0o111` added. `chmod` sets the read-only
  attribute when `0o200` is clear and ignores the other bits. These are Go's rules.
- **Writable?** Ant has no `access()`. The mode cannot answer `[[ -w dir ]]`: the owner, the group,
  root, ACLs and read-only mounts all decide. A script that asks tries instead, with
  `File.tempFile("probe-*", dir: dir)`, so the answer is true when it is used.

### Temporary files

```
File.tempFile(pattern: string, dir: string | null = null): Result[string, Error]   // created empty, mode 0o600
File.tempDir(pattern: string, dir: string | null = null): Result[string, Error]    // created empty, mode 0o700
```

- The name is `pattern` with its last `*` replaced by a random string, or with the string appended
  when there is no `*`. So `"report-*.csv"` keeps its extension and `"build"` works as `tempDir`'s
  `prefix` does today. A pattern that contains a path separator is `Err`.
- `dir` is the host's temporary directory when null: `$TMPDIR` or `/tmp` on Unix, `%TEMP%` on
  Windows.
- Both return the path. The program opens the file if it wants a handle, and removes the file or
  directory when it is done with it: `defer { File.remove(tmp) }` or
  `defer { File.removeAll(dir) }`. Nothing is removed for the program, not at exit and not when a
  handle closes. `Os.exit` runs no deferred block, so a script that exits early with temporary files
  removes them first.
- `tempDir`'s parameter is renamed from `prefix` to `pattern`. A call that named it, as
  `tempDir(prefix: "x")`, has to be updated.

### Copies

```
File.copy(from: string, to: string): Result[void, Error]      // one file: its contents and mode
File.copyAll(from: string, to: string): Result[void, Error]   // a file or a tree; Err when to exists
```

- `copy` copies one file's contents and sets the copy's mode to the source's. It replaces `to` when
  `to` is a file, as `write` and `rename` do, and is `Err` when `to` is a directory: there is no
  "into" form, so `Path.join` says where. A link at `from` is followed, so the target's contents are
  copied, as `cp` does. A link at `to` is followed too, as `write` follows it and as Go's
  `os.OpenFile(to, O_TRUNC)` does: the link's target is replaced and gets the mode, and a dangling
  link's target is created. To replace the link itself, `remove` it first. Times are not copied. The
  copy is not atomic: a reader can see `to` half-written, and copying to a temporary file and
  renaming it fixes that.
- **Never onto itself.** Copying a file over itself would truncate the source before reading it. So
  `copy` is `Err` before anything changes when `from` and `to` are the same file, which is decided
  by identity (device and inode, as Go's `os.SameFile`), not by text. That catches
  `copy("a", "./a")` and a link at `to` that points at `from`. `copyAll` is `Err` in the same way.
  It is also `Err` when `to`, with its links resolved, lies inside `from`, as
  `copyAll("src", "src/backup")` does, since the copy would copy itself.
- `copyAll` is the recursive form, named like `removeAll` so the bigger operation has the longer
  name. It is `Err` when `to` exists, unless `to` is an empty directory (see below), so two trees
  are never merged by accident. Directories and files keep their modes. A link inside the tree is
  recreated as a link with the same target, as `cp -R` does, which also means a cycle cannot loop. A
  link at `from` itself is followed. On an error partway through, what was copied stays.
- The implementation may use the file system's copy-on-write clone, `copy_file_range` or similar.
  The result is the same.

### Symbolic links

```
File.symlink(target: string, link: string): Result[void, Error]   // link is created and points at target
File.readLink(path: string): Result[string, Error]                // the target as stored, not resolved
File.realPath(path: string): Result[string, Error]                // absolute and clean, with every link resolved
File.stat(path: string, follow: bool = true): Result[File.Info, Error]
```

- The order is `ln -s`'s: target first. `target` is stored as given, so a relative target is
  relative to the link's directory, not to the working directory. `symlink` is `Err` when `link`
  exists.
- `File.stat(path, follow: false)` describes the link itself. `isSymlink` is true only then: with
  `follow` true, the result describes the target. A named parameter is Python's choice. It keeps one
  `stat` instead of adding `lstat`, and reads well at the call: `File.stat(p, follow: false)`.
- `realPath` is `Err` when the path or a link along it does not exist. `Path.abs` stays a function
  of the text alone.
- On Windows, `symlink` makes a directory link when `target` exists and is a directory, and a file
  link otherwise. Without Developer Mode or administrator rights it is `Err`.

What the existing functions do with a link:

| Function                                                                           | At a link                                                                   |
| ---------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| `read`, `write`, `open`, `create`, `append`, `chmod`, `touch`, both ends of `copy` | follow it                                                                   |
| `exists`                                                                           | follows it, so a dangling link is `false`; `stat(p, follow: false)` sees it |
| `stat`                                                                             | follows it unless `follow: false`                                           |
| `remove`, `rename`                                                                 | act on the link, not the target                                             |
| `removeAll`                                                                        | removes the link, never what it points at                                   |
| `list`                                                                             | lists the link's name                                                       |
| `walk`                                                                             | does not descend through it; reports it as a file                           |
| `copyAll`                                                                          | recreates it inside the tree; follows it at the root                        |

### Times

```
File.touch(path: string, time: Time.Instant | null = null): Result[void, Error]
```

`touch` creates an empty file when there is nothing at `path`. It sets the modified and access times
to `time`, or to now when `time` is null. It never truncates, and it follows a link. The file system
may round the time: FAT to 2 seconds, HFS+ to 1, ext4 and APFS to the nanosecond. So `stat` can
return a slightly earlier `modified` than the one set. Access time is not in `File.Info`: many
systems mount with `noatime` or `relatime`, so it rarely means what it says.

### The process

```
Os.pid(): int                          // this process's id
Os.uid(): int | null                   // the effective user id; 0 is root; null on Windows
Os.home(): Result[string, Error]       // $HOME on Unix, %USERPROFILE% on Windows; Err when unset
Os.hostname(): Result[string, Error]
```

`Os.pid` matches `Cmd.pid`, which gives a child's id. `Os.uid` is the effective id, the one that
decides what the process may touch, so `Os.uid() == 0` is `[[ $EUID -eq 0 ]]`. On Windows it is null
rather than Go's −1, so the type says that the answer can be missing.

### `copyAll` onto an existing directory (open)

Three choices, from the prior art:

- **Refuse any existing `to`.** This is the simplest rule. A program that has just made `dir` copies
  to `Path.join([dir, "src"])`, or removes `dir` first.
- **Accept an empty directory (recommended).** An existing directory with no entries is used as the
  root of the copy. Since it is empty, the result is the tree a fresh `to` would get, so nothing is
  merged and nothing is lost. The directory keeps its own mode, so a `tempDir` stays `0o700` instead
  of taking the source's `0o755`. A directory with any entry, a file, or a link at `to` is `Err`
  before anything is copied. A file at `from` still needs a `to` that does not exist.
- **Merge, as `rsync`, `fs.cp` and `dirs_exist_ok` do.** Rejected: the result is neither tree,
  replacing files silently loses data, and stopping halfway on a conflict as Go does leaves a mix. A
  program that wants a merge says so with a loop over `File.list(from)`.

"Into", as `cp -R` does it, is rejected for the reason `copy` has no "into" form.

### More of the process (open)

Each candidate, with what it is for and a recommendation:

| Candidate                         | Shell                           | Prior art                                                                                               | Recommendation            |
| --------------------------------- | ------------------------------- | ------------------------------------------------------------------------------------------------------- | ------------------------- |
| `Os.configDir()`, `Os.cacheDir()` | `${XDG_CONFIG_HOME:-~/.config}` | Go `os.UserConfigDir`, `UserCacheDir`; Python `platformdirs`, Rust `dirs`, Node `env-paths` as packages | **add**                   |
| `File.isTerminal(f)`              | `[[ -t 1 ]]`                    | Rust `IsTerminal` (1.70), Deno `isTerminal()`, Node `isTTY`, Python `isatty()`; Go only in `x/term`     | **add**                   |
| the user name                     | `whoami`, `$USER`               | Go `os/user.Current`, Node `os.userInfo`, Python `getpass.getuser`                                      | not now: `Os.env("USER")` |
| the temporary directory's path    | `$TMPDIR`                       | Go `os.TempDir`, Python `tempfile.gettempdir`, Node `os.tmpdir`                                         | no                        |
| `Os.gid`, groups                  | `id -g`, `id -G`                | Go `os.Getgid`, `Getgroups`                                                                             | no                        |
| `Os.setenv`, `Os.chdir`           | `export`, `cd`                  | everywhere                                                                                              | no                        |

- **Config and cache directories.** A script that writes a token at `0o600` needs somewhere to put
  it, and the right place differs on each system: `$XDG_CONFIG_HOME` or `~/.config` on Linux,
  `~/Library/Application Support` and `~/Library/Caches` on macOS, `%AppData%` and `%LocalAppData%`
  on Windows. Writing that by hand is a dozen lines that most scripts get wrong on at least one
  system. Go's rules, as `os.UserConfigDir` and `os.UserCacheDir` implement them, are
  `Result[string, Error]`, `Err` when `$HOME` is unset or an XDG variable holds a relative path.
  Like Go, they return the root and the program adds its own name under it. The data and state
  directories of the XDG spec are left out, as Go leaves them out.
- **`File.isTerminal(f: File.Handle): bool`.** It is the check behind colours, progress bars and
  prompts: `File.isTerminal(Os.stdout())`. It belongs to `File` because it is a question about a
  handle, and it is false for files, pipes and connections. The runtime already makes this test for
  `Cmd.run` and for standard input.
- **The user name.** The honest answer needs the system's user database. Go's `os/user` uses libc
  through cgo, and without cgo it parses `/etc/passwd`, which misses directory users such as LDAP
  and macOS accounts. `$USER`, which is what shell scripts use, is already `Os.env("USER")`, and
  `Os.uid()` covers the "am I root" question that most uses of `whoami` ask.
- **The temporary directory's path.** The only safe use of a shared temporary directory is a random
  name, which `tempFile` and `tempDir` already give. A fixed name such as `/tmp/app.sock` is open to
  squatting and symlink attacks, and a function that hands out the path invites it. Fixed paths
  belong in `Os.cacheDir()`.
- **`gid`.** It is useful only with the supplementary groups and file owners, which are non-goals.
- **`setenv` and `chdir`.** Both are process-wide state that races with every goroutine. `Cmd` takes
  `env` and `dir` for a child, and `ant test` rejected `Test.setenv` and `Test.chdir` for the same
  reason.

## Exclusive creation (open)

Ant has no atomic "create unless it exists" today. `write`, `create` and `rename` replace, and
`mkdir` is `Ok` when the directory exists, so the shell's `mkdir lock` idiom does not work either.

### What it is for

| Use                                         | Needs                                         | `O_EXCL` enough?                                     |
| ------------------------------------------- | --------------------------------------------- | ---------------------------------------------------- |
| claim a name: `out-1.txt`, `out-2.txt`, ... | atomic create                                 | yes                                                  |
| never clobber: refuse to overwrite output   | atomic create                                 | yes                                                  |
| initialize once: write a config if absent   | atomic create, ideally the whole file at once | mostly                                               |
| single instance: a pid or lock file         | survives nothing: freed when the process dies | **no**: a crash leaves it                            |
| publish a finished file under a free name   | whole file, atomic, no replace                | **no**: readers see it empty                         |
| a secret, created exclusively               | exclusive, and never readable by others       | **no**: another process can open it before a `chmod` |

So exclusive creation covers the first three uses. Single instance needs a lock. Publishing and
secrets need a rename that will not replace.

### How Ant reports "exists"

Ant's library errors are a message only, and section 8 says that an error code branches on should be
a type of its own. The library already handles expected outcomes another way: it puts them in the
`Ok` value. `File.readLine` is `Ok(null)` at the end of the file, and `Os.which` and `Os.env` are
null when there is nothing. A path that is taken is the same kind of outcome. It is the case the
program asked about, not a failure.

### Options

**A. A named parameter.** `exclusive: bool = false` on `write`, `writeBytes` and `create`.

```ant
var written = File.write("out.txt", report, exclusive: true)
if written is Err {
  if File.exists("out.txt") {
    eprintln("out.txt exists")
    Os.exit(1)
  }
  return written
}
```

This is one parameter, the spelling Deno, Rust and Node use as an option. But a parameter cannot
change the result's type, so "exists" is an `Err` that the program tells apart with a second check.
That check races, though harmlessly: a file that vanished in between means trying again.

**B. `File.createNew`, whose result says it.**

```
File.createNew(path: string): Result[File.Handle | null, Error]   // null when anything is at path
```

```ant
var f = try File.createNew("out.txt")
if f == null {
  eprintln("out.txt exists")
  Os.exit(1)
}
try File.writeText(f, report)
try File.close(f)
```

The type carries the case the program must handle, as `readLine` does, and no second check is
needed. A link at `path` counts as taken, even a dangling one, as `O_EXCL` has it. The file is
created as `create` creates one: `0o666` less the umask. Writing a whole file exclusively is
`createNew`, `writeText`, `close`. A `writeNew(path, text): Result[bool, Error]` could be added
later if that turns out to be common.

**C. Kinds on library errors.** For example, `Error` gains `kind: string`, or `File`'s errors become
a union with `File.Exists`, `File.NotFound` and `File.Denied`. This would also answer "not found"
for every `File` call. But it changes the error type of every namespace, touches section 8, and goes
against the rule that the library's `Error` is text. It deserves its own proposal, and B does not
depend on it.

**D. Locks, for the single-instance use.**

```
File.lock(f: File.Handle, shared: bool = false, ctx: Context | null = null): Result[void, Error]   // waits
File.tryLock(f: File.Handle, shared: bool = false): Result[bool, Error]                             // false when another holds it
```

```ant
try File.touch("app.lock")            // created if missing, never truncated
var f = try File.open("app.lock")     // a read handle can be locked, on Windows too
if !(try File.tryLock(f)) {
  eprintln("already running")
  Os.exit(1)
}
// the lock is held until f is closed or the process ends, however it ends
```

The semantics are Rust's: `flock` on Unix and `LockFileEx` on Windows, the whole file, and the lock
belongs to the handle. A second handle on the same file conflicts with the first, even in the same
process, so it also works between goroutines. Closing the handle releases the lock, and the kernel
releases it when the process dies, so nothing is left stale.

There is no `unlock`. Writes to a handle are buffered (File), so an `unlock` before `close` could
release the lock while the data is still in the buffer, and another process would then write over
it. The lock lasts until `File.close`, which flushes first and reports what the flush reports. A
handle from `append` cannot be locked: Go opens it with append-only access on Windows, and
`LockFileEx` needs read or write access, as Rust's documentation warns. A handle from `open`,
`create` or `createNew` can.

D is the one part of this proposal that goes beyond Go's standard library. Go has `O_EXCL` but no
portable lock: `syscall.Flock` on Unix and the private `cmd/go/internal/lockedfile`. The lock is
advisory on Unix and mandatory on Windows, and the spec says both. A context ends a wait in `lock`.
Since `flock` cannot be interrupted, the implementation retries the non-blocking form.

**E. A rename that refuses to replace,** for publishing a finished file and for secrets:
`File.rename(from, to, replace: false)`, `Err` or `Ok(false)` when `to` exists. It is
`RENAME_NOREPLACE` on Linux, `RENAME_EXCL` on macOS, `MoveFileEx` on Windows, and `link` plus
`unlink` elsewhere. But `RENAME_NOREPLACE` is missing on some Linux file systems, the `link`
fallback fails on FAT, and no standard library in the prior art offers it. It is worth doing only
when someone needs it.

### Recommendation

**B and D**, in this proposal. B covers claiming a name, never clobbering and initializing once. D
covers single instance, which B cannot do safely and which is the most common reason scripts reach
for a pid file. C and E stay out: C needs its own proposal, and E is rare enough to wait for a real
use.

## Considered and rejected

- **A mode parameter on `write` and `create`.** It applies only at creation, see Prior art. The
  temporary file plus rename covers the case that matters, and `chmod` covers the rest.
- **Modes as strings** (`"rw-------"`, `"u+x"`). They are checked only when the program runs and are
  a second notation to learn. `0o` is in every manual.
- **`File.access` or `File.writable`.** It answers for a moment that is already gone (a
  time-of-check race), and it means nothing on Windows. Trying is the answer.
- **Removing temporary files automatically** at exit or when a handle closes. Ant prefers explicit
  cleanup with `defer`, and `Os.exit` already promises that nothing runs.
- **`File.lstat`.** `stat(path, follow: false)` is the same thing without a second name.
- **`copy` into a directory** when `to` is one, as `cp` and `shutil.copy` do. One call means one
  thing whatever is on disk.
- **Copying times by default.** Nobody does that, and a fresh `modified` is what makes a build tool
  see the copy.
- **`Os.umask`.** It is process-wide, so it races with every goroutine that creates a file, and
  Windows has none. A script sets its umask in the shell that starts it, or uses `chmod`.

## Spec changes

- `ant-lang.md` section 2, Literals: `0o` octal integer literals.
- `stdlib.md`, File: the new functions and the new `Info` fields. Paragraphs on modes and the umask,
  temporary files, copies, links (the table above) and times. `tempDir` gains `pattern` and `dir`.
- `stdlib.md`, Os: `pid`, `uid`, `home`, `hostname`.
- Whatever the open questions settle: `createNew`, the locks, `configDir`, `cacheDir`, `isTerminal`.
- `fmt.md`: none. Number literals are printed as written.

Implementation, beyond the spec: the lexer, `internal/lib/file.go` and `os.go`, `prelude.d.ant`
docs, tests in `tests/run`. Syntax highlighting needs `0o` in the TextMate grammar (vscode-ant,
intellij-ant), the Monaco language in ant-web, and nvim-ant's syntax.

## Non-goals

- Globbing (`*.txt`). It has its own entry in `ant/TODO.md`.
- Owners and groups: `chown`, and `uid` and `gid` in `File.Info`.
- Hard links, ACLs, extended attributes, Windows file attributes beyond read-only.
- Kinds on library errors (option C), except as a follow-up.
- A rename that refuses to replace (option E), until a use appears.

## Decisions

From the review of 2026-11-01:

1. `0o` octal literals.
2. No `access()` or `writable()`. A script tries with `tempFile` instead.
3. No mode parameter on `write` or `create`. Secrets use a temporary file and `rename`.
4. `tempDir` promises `0o700`.
5. `tempFile` returns the path.
6. `mode` holds the bits in `0o7777`, and the umask applies to new files and directories.
7. Go's rules for modes on Windows.
8. Nothing removes temporary files for the program.
9. `File.copy` and `File.copyAll`, as designed above.
10. Symbolic links, as designed above.
11. Globbing goes to `ant/TODO.md`.
12. `File.touch(path, time)`, not a setter that requires the file.
13. `Os.pid`, `Os.uid`, `Os.home` and `Os.hostname`.

## Open questions

1. Exclusive creation and locks: which of A to E. Recommended: B (`File.createNew`) and D
   (`File.lock`, `tryLock`; released by `close`). D goes beyond Go's standard library.
2. `copyAll` onto an existing directory. Recommended: accept an empty one and keep its mode.
3. More of the process. Recommended: add `Os.configDir`, `Os.cacheDir` and `File.isTerminal`, and
   nothing else.
