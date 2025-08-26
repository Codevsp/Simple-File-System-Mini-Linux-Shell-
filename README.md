# SFS — A Simple File System (C) / Mini Linux Shell

Author: **VSP_9196**

This repository contains a minimal, educational file system (“SFS”) and a tiny interactive shell implemented in C.  
SFS stores its state in a single disk image file `sfs.disk` and exposes a handful of commands to create files, list directories, navigate, display file contents, remove files/directories (recursively), and print statistics.

The goal is to demonstrate classic FS building blocks: superblock, bitmaps, an inode table, fixed-size directory entries, and simple allocation.

---

## Table of Contents
- [Design Overview](#design-overview)
- [On-Disk Layout](#on-disk-layout)
- [Core Data Structures](#core-data-structures)
- [Build & Run](#build--run)
- [Interactive Shell](#interactive-shell)
- [Command Reference (in depth)](#command-reference-in-depth)
  - [rd](#rd--reset-to-root)
  - [ls](#ls--list-directory)
  - [cd](#cd-dname--change-directory)
  - [md](#md-dname--make-directory)
  - [display](#display-fname--print-file-contents)
  - [create](#create-fname--create-and-edit-a-file)
  - [rm](#rm-name--remove-file-or-directory)
  - [stats](#stats--free-space)
- [Allocation & Freeing](#allocation--freeing)
- [Limits & Assumptions](#limits--assumptions)
- [Error Messages You’ll See](#error-messages-youll-see)
- [Troubleshooting Notes](#troubleshooting-notes)

---

## Design Overview

- **Disk image**: All state lives in `sfs.disk` (100 blocks of 1024 bytes each; blocks are numbered `0..99`).
- **Metadata blocks** (reserved):
  - `0`: Superblock
  - `1`: Block bitmap
  - `2`: Inode bitmap
  - `3`: Inode table
- **Data blocks**: `4..99` hold either directory pages or file content.
- **Inodes**: 128 entries (index `0..127`). Each inode has:
  - A **type** (`DI` for directory, `FI` for file),
  - Up to **three direct block pointers** (`XX`, `YY`, `ZZ`) stored as 2-digit, zero-padded strings.
- **Directories**: Fixed-size pages. Each directory block holds 4 directory entries (1024 / 256).
- **Allocation**: First-fit using bitmaps, never compacts/defragments.
- **Root**: Inode `0` is the root directory. The shell keeps a variable `CD_INODE_ENTRY` (current directory’s inode index).

---

## On-Disk Layout

```

Block 0  (SUPER)  : "BLB INB \<zeros...>"  (first 3 chars = BLB, next 3 = INB)
Block 1  (BLK-BM) : 1024 ASCII '0'/'1' chars; first BLB entries matter
Block 2  (INO-BM) : 1024 ASCII '0'/'1' chars; first INB entries matter
Block 3  (INODES) : 128 inode entries serialized into 1024 bytes
Blocks 4..99      : Data (directory pages or file content)

````

- **BLB** (Block-bitmap length): typically `100` (blocks 0..99).
- **INB** (Inode count): `128`.
- **Inode entry on disk**: `TTXXYYZZ` (8 chars).
- **Directory entry on disk** (256 bytes):
  - `F` (1 char): `'1'` used, `'0'` free  
  - `fname` (252 chars including `\0`)  
  - `MMM` (3 chars): zero-padded inode index

---

## Core Data Structures (in memory)

```c
typedef struct {
  char TT[2];     // "DI" or "FI"
  char XX[2];     // 2-digit block index or "00"
  char YY[2];
  char ZZ[2];
} _inode_entry;   // 8 bytes serialized, but C struct has no packing guarantees

typedef struct {
  char F;             // '1' used, '0' free
  char fname[252];    // null-terminated name
  char MMM[3];        // 3-digit inode index
} _directory_entry;   // 256 bytes by design
````

**Global state while running**

* `BLB` / `INB`: total blocks tracked and total inodes.
* `_block_bitmap[1024]`, `_inode_bitmap[1024]`: ASCII '0'/'1'.
* `_inode_table[128]`: in-memory copy of the on-disk table.
* `free_disk_blocks`, `free_inode_entries`: cached counters.
* `CD_INODE_ENTRY`: current directory inode index (starts at 0).
* `current_working_directory`: prompt label (not a full path).

---

## Build & Run

> SFS requires a **pre-formatted** `sfs.disk` matching the layout above. If you don’t have one, use the provided sample disk (or write a tiny formatter).

1. **Compile**

```sh
gcc -O2 -Wall -Wextra -o sfs sfs.c
```

2. **Ensure disk exists**

```
./sfs
Disk file sfs.disk not found.
```

If you see the above, place a valid `sfs.disk` next to the binary.

3. **Run**

```sh
./sfs
SFS::/# _
```

The shell waits for commands until you type `exit`.

---

## Interactive Shell

**Prompt**: `SFS::<cwd># ` (shows just the current directory name; root is `/`).

**Available commands**

```
ls
rd
cd <dname>
md <dname>
display <fname>
create <fname>
rm <name>
stats
exit
```

---

## Command Reference (in depth)

### `rd` — Reset to Root

**Synopsis**

```
rd
```

**What it does**

* Sets current directory inode to **0** (root).
* Updates the prompt label to `/`.

**Notes**

* No disk I/O except prompt update; purely in-memory swap of `CD_INODE_ENTRY` and label.

---

### `ls` — List Directory

**Synopsis**

```
ls
```

**Behavior**

* Reads `XX/YY/ZZ` blocks from the current directory’s inode.
* For each non-zero block, reads 4 directory entries.
* Prints file names (plain) and directory names (ANSI **red**).
* Prints a summary line:
  `N file(s) and M director(y|ies).`

**Implementation notes**

* Will `exit(1)` if the current inode is a file (shouldn’t happen in normal use).
* Ignores empty (`F == '0'`) directory entries.

**Example**

```
SFS::/ # ls
notes    src    assets
3 files and 0 directories.
```

---

### `cd <dname>` — Change Directory

**Synopsis**

```
cd <dname>
```

**Behavior**

* Searches the current directory’s entries for a directory named `dname`.
* On success, updates `CD_INODE_ENTRY` and prompt label.
* On failure prints: `<name>: No such directory.`

**Details**

* Only moves **one level** (no `/` or `..` logic; no full path resolution).
* Only matches entries where the inode type is `DI`.

**Examples**

```
SFS::/ # cd docs
SFS::docs# ls
README  specs
SFS::docs# cd specs
specs: No such directory.
```

---

### `md <dname>` — Make Directory

**Synopsis**

```
md <dname>
```

**High-level flow**

1. Validate `dname` and ensure **free inode** exists.

2. Scan the current directory’s 0..2 data blocks:

   * If a block pointer is `0`, remember its **index** as an expandable slot.
   * Within non-zero blocks, remember the **first free directory entry**.
   * If a used entry matches `dname`, abort with `Already exists.`

3. **If** no free dir entry but some pointer is `0`:

   * Allocate a **new data block** (`getBlock()`), zero it, and plug it into `XX/YY/ZZ`.
   * Use its entry slot `0`.

4. Allocate an **inode** for the new directory (`getInode()`), write the new directory entry:

   * `F = '1'`, `fname = dname`, `MMM = inodeIndex`.
   * Initialize that inode in the table as `TT="DI", XX="00", YY="00", ZZ="00"`.
   * Persist the inode table.

**Possible errors**

* `Error: Inode table is full.`
* `Error: Maximum directory entries reached.` (all 3 blocks used, no free slots)
* `Error: Disk is full.` (needed a new dir block but no free blocks)
* `<name>: Already exists.`

**Example**

```
SFS::/ # md projects
SFS::/ # cd projects
SFS::projects# ls

0 files and 0 directories.
```

---

### `display <fname>` — Print File Contents

**Synopsis**

```
display <fname>
```

**Behavior**

* Ensures current inode is a directory (else: `Display Error: CD_INODE is a file.` and program exits).
* Searches the directory for an entry named `fname` and confirms it is a **file** (not a directory).

  * If it’s a directory: `Given File name is a directory. Cannot be open.`
  * If not found: `File not found in the current directory`
* For the file’s inode, reads up to 3 data blocks (`XX/YY/ZZ`) and prints them in order.

**Notes**

* Content is printed with `printf("%s")`. Text is expected to be null-terminated within each 1KB block (the editor used by `create` ensures this at the point you press **Esc**).

**Example**

```
SFS::/ # display notes
Shopping list:
- Eggs
- Milk
- Bread
```

---

### `create <fname>` — Create and Edit a File

**Synopsis**

```
create <fname>
```

**What it does**

* Validates that the current inode is a directory.
* Scans existing directory blocks:

  * Detects **duplicate filename** → `<fname>: Already exists.`
  * Finds a **free directory entry** (or allocates a **new directory block** if possible).
* Allocates a **new inode** for the file and writes a directory entry pointing to it.
* Initializes the file inode `TT="FI"` and clears `XX/YY/ZZ`.

**Editor / input mode**

* Prompts: `<fname> has been created, enter the text.`
* Reads bytes from `stdin`, **up to 3 blocks (3 \* 1024 bytes)**.
* **Press the Esc key** to finish editing early. The code writes a `\0` at the cursor and flushes the block.
* If you fill an entire block, it allocates another and continues (max 3).
* If no more blocks are available:
  `File system full: No such data blocks!` and “Data will be truncated!”
* If you reach 3 blocks:
  `Maximum file size reached! Then data will be truncated!`

**Important**

* The terminator is the **Esc** key (`ASCII 27`), **not** Ctrl-D/Ctrl-C.

**Examples**

```
SFS::/ # create hello
hello has been created, enter the text.
Hello, SFS!
<press Esc>

SFS::/ # display hello
Hello, SFS!
```

---

### `rm <name>` — Remove File or Directory

**Synopsis**

```
rm <name>
```

**Behavior**

* Validates current inode is a directory.

* Scans each directory block and its 4 entries:

  * On name match:

    * If entry’s inode is a **file** → `removeFile(inode)`

      * Frees up to 3 data blocks (if non-zero) and frees the inode
    * If entry’s inode is a **directory** → `removeDirectory(inode)` (**recursive**)

      * Recursively visits, removes child files/dirs, frees directory pages, frees inode
    * Marks the directory entry `F='0'`.

* After processing a block, if **all 4 entries are free**, returns that directory block to free list and zeros the corresponding pointer in the **current directory’s** inode (`XX/YY/ZZ`).

**Possible errors**

* If current is a file → `Remove error: CD_INODE is a file!` (exits)
* Wrong type to `removeFile`/`removeDirectory` (hard errors that call `exit(1)`).
* If not found at all → `<name> not found in current directory!`

**Example (recursive)**

```
SFS::/ # ls
docs   tmp
SFS::/ # rm docs
SFS::/ # ls
tmp
```

---

### `stats` — Free Space

**Synopsis**

```
stats
```

**Behavior**

* Re-counts **free blocks** from the block bitmap and **free inodes** from the inode bitmap.
* Prints:

```
X block(s) free.
Y inode entr(y|ies) free.
```

**Example**

```
SFS::/ # stats
72 blocks free.
101 inode entries free.
```

---

## Allocation & Freeing

* **Block allocation (`getBlock`)**

  * Scans block bitmap for first `'0'`, flips to `'1'`, updates `free_disk_blocks`, writes bitmap to disk.
  * Returns block index or `-1` if none.
  * Blocks `0..3` are **reserved** and never returned.

* **Block free (`returnBlock`)**

  * Only frees indices `> 3` and `<= 99`.
  * Flips bitmap entry to `'0'`, updates `free_disk_blocks`, persists bitmap.

* **Inode allocation (`getInode`)**

  * Similar to blocks, scans inode bitmap for `'0'`.
  * Returns inode index or `-1`.

* **Inode free (`returnInode`)**

  * Frees indices `1..127` (inode `0` is the root).
  * Flips bitmap entry to `'0'`, updates `free_inode_entries`, persists bitmap.

---

## Limits & Assumptions

* **Max disk size**: 100 blocks × 1KB = **100KB** total image.
* **Usable data**: blocks **4..99** (96KB) minus metadata and directory overhead.
* **Max inodes**: **128** (index `0..127`).
* **Max file size**: **3KB** (three direct blocks).
* **Directory capacity**: Up to **3 blocks × 4 entries = 12 entries** per directory.
* **Names**: Up to **251 visible chars** (`fname[252]` includes the trailing `\0`).
* **Navigation**: Single-segment `cd` only (no path, no `..`, no `/`).
* **Case sensitivity**: Exact `strcmp`/`strncmp`.

---

## Error Messages You’ll See

* General

  * `Disk file sfs.disk not found.` (on mount)
  * `Fatal Error! Aborting.` (programming invariant violation)
* Navigation / I/O

  * `Display Error: CD_INODE is a file.`
  * `Given File name is a directory. Cannot be open.`
  * `File not found in the current directory`
  * `<dname>: No such directory.`
  * `<name>: Already exists.`
* Creation / Space

  * `File system is full: There is no empty space in this directory!`
  * `File system is full: No data blocks available!`
  * `File system full: No such data blocks!` (during content entry)
  * `Data will be truncated!`
  * `Maximum file size reached! Then data will be truncated!`
  * `Error: Inode table is full.`
  * `Error: Maximum directory entries reached.`
  * `Error: Disk is full.`
* Removal

  * `Remove error: CD_INODE is a file!`
  * `Remove file error: inode is a directory!`
  * `Remove directory error: inode is a directory!` (string says "directory", but indicates wrong type)
  * `<name> not found in current directory!`

---

## Troubleshooting Notes

* **Esc to finish `create`**: The editor ends when you press **Esc** (`ASCII 27`). This inserts a `\0` and saves the block. (If your terminal intercepts Esc, try pressing it once plainly; do not hold or use modifiers.)
* **Windows consoles** may not show ANSI color for directories in `ls`.
* **Undefined `fflush(stdin)`**: The code calls `fflush(stdin)` in a couple places; on some C libraries this is undefined. The program still works in typical Linux environments.
* **Struct packing**: The in-memory `_inode_entry` structure relies on 8-byte serialized form on disk; the program reads/writes the entire 1024-byte inode table block. Do not change struct packing unless you also change serialization.
* **Formatter**: This repo assumes you already have a valid `sfs.disk`. If you need to create one, write a tiny formatter that:

  1. Writes block 0 as `"100128" + 1018 zeros`,
  2. Initializes bitmaps with `'1'` for blocks 0..3 / inode 0 and `'0'` elsewhere,
  3. Writes an inode table with inode 0 = `DI` and pointers zeroed,
  4. Zeros out blocks 4..99.

