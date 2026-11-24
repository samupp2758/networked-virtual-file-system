# Networked Virtual File System

A Unix-style, inode-based file system implemented from scratch in C++. The file system lives inside a single disk-image file, is served over TCP by a dedicated file server, and is operated through an interactive client shell that supports familiar commands such as `ls`, `cp`, `mv`, `ln` and `chown`.

Originally developed as the final project for CSC 351 (Operating Systems).

## Overview

The project is split into three layers, each with a single responsibility:

| Layer | Source | Responsibility |
| --- | --- | --- |
| Storage engine | [FS.h](FS.h), [Read-Write.cpp](Read-Write.cpp) | Block I/O, block and inode allocation, directories, permissions and file data on the disk image |
| File server | [main/fs.cpp](main/fs.cpp), [main/_tcp_fs.h](main/_tcp_fs.h) | Exposes storage-engine operations to clients as JSON requests over TCP |
| Client shell | [main/shell.cpp](main/shell.cpp), [main/shell.h](main/shell.h) | Interactive command line that translates user commands into server requests |

```
┌──────────────┐   JSON over TCP (port 230)    ┌──────────────┐   block I/O   ┌──────────────┐
│ Client shell │ ────────────────────────────▶ │ File server  │ ────────────▶ │  Disk image  │
│   ./shell    │ ◀──────────────────────────── │    ./fs      │ ◀──────────── │  (2 GiB file)│
└──────────────┘   file data   (port 231)      └──────────────┘               └──────────────┘
```

## Key features

- **Inode-based design** — 128-byte inodes holding mode, owner, group, link count, size and access/modify/change timestamps.
- **Large-file support** — 12 direct block addresses per inode plus single, double and triple indirect blocks.
- **Bitmap allocation** — block and inode bitmaps, with a next-fit strategy for allocating data blocks.
- **Hierarchical directories** — absolute and relative path resolution, nested directories and hard links.
- **Unix-style permissions** — read/write/execute bits for owner, group and others, enforced on every shell operation.
- **Client/server architecture** — the shell can run on a different machine from the file server.
- **Host import and export** — copy files between the host machine and the virtual file system in either direction.

## On-disk layout

The disk image is a fixed 2 GiB file divided into 524,288 blocks of 4,096 bytes.

| Blocks | Contents |
| --- | --- |
| 0 | Reserved |
| 1 – 16 | Block allocation bitmap |
| 17 | Inode allocation bitmap (32,768 inodes) |
| 18 onward | Inode table (32 inodes per block) |
| Remainder | Data blocks, directory blocks and indirect address blocks |

The root directory is inode 0 and is created automatically when a new file system is formatted.

## Getting started

### Prerequisites

- A C++11 compiler (`g++` or `clang++`)
- macOS or Linux (the networking layer uses POSIX sockets)
- 2 GiB of free disk space for the disk image

The only third-party dependency, [nlohmann/json](https://github.com/nlohmann/json), is vendored in [main/utils](main/utils).

### Build

From the `main` directory, build the file server and the shell:

```bash
cd main
g++ -std=c++11 ../Read-Write.cpp fs.cpp -o fs -funsigned-char
g++ -std=c++11 shell.cpp -o shell
```

### Run

Start the file server first, in its own terminal. The second argument selects whether to format a new disk image (`1`) or open an existing one (`0`):

```bash
./fs filesystem.dat 1
```

Then start the shell in a second terminal, choosing the user to log in as. An optional second argument sets the server address (defaults to `127.0.0.1`):

```bash
./shell 0
```

The server listens on ports 230 and 231. On Linux these are privileged ports, so the server may need to be run with elevated privileges.

### Users

Three users are predefined:

| User (UID) | Group (GID) |
| --- | --- |
| 0 | 0 |
| 1 | 0 |
| 2 | 1 |

## Shell commands

Every command accepts `-h` or `--help` to print its usage.

| Command | Usage | Description |
| --- | --- | --- |
| `ls` | `ls path/to/directory` | List directory contents with mode, links, owner, group, size and modified time |
| `cd` | `cd path/to/directory` | Change the working directory |
| `mkdir` | `mkdir path/to/directory` | Create a directory |
| `rmdir` | `rmdir path/to/directory` | Remove an empty directory |
| `cat` | `cat path/to/file` | Print the contents of a file |
| `cp` | `cp [sources...] destination` | Copy files within the file system |
| `mv` | `mv [sources...] destination` | Move or rename files |
| `ln` | `ln source destination` | Create a hard link |
| `rm` | `rm [path/to/file] ...` | Remove one or more files |
| `chown` | `chown [UID:GID or UID] path` | Change the owner, and optionally the group, of a file or directory |
| `lcp` | `lcp machine/source filesystem/destination` | Import a file from the host machine into the file system |
| `Lcp` | `Lcp filesystem/source machine/destination_folder` | Export a file from the file system to the host machine |
| `clear` | `clear` | Clear the terminal |
| `exit` | `exit` | Disconnect the shell and leave the server running |
| `shutdown` | `shutdown` | Disconnect the shell and stop the server |

## Client/server protocol

The shell and the server exchange JSON messages over a TCP connection. Each request names the storage-engine operation to perform in a `call` field, followed by its arguments:

```json
{ "call": "my_mkdir", "path": "/documents", "user": { "UID": 0, "GID": 0 } }
```

The server replies with a JSON object containing the result. File contents are transferred as raw bytes in chunks, with uploads sent over a second connection on port 231 so that data and control messages stay separate.

## Repository structure

```
.
├── FS.h                 Storage-engine interface (FileSystem class)
├── Read-Write.cpp       Storage-engine implementation
├── Read-Write-m.cpp     Standalone test driver for the storage engine
├── makefile             Builds the standalone test driver
└── main
    ├── fs.cpp           File server
    ├── _tcp_fs.h        Server request handler interface
    ├── shell.cpp        Client shell
    ├── shell.h          Shell interface
    └── utils            Vendored headers (nlohmann/json)
```

## Limitations

- The server handles one shell connection at a time.
- The disk-image size, block size and port numbers are fixed at compile time.
- Users and groups are predefined in the shell and are not stored in the file system.
