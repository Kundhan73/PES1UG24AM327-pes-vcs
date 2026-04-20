# Building PES-VCS — A Version Control System from Scratch

**Objective:** Build a local version control system that tracks file changes, stores snapshots efficiently, and supports commit history. Every component maps directly to operating system and filesystem concepts.

**Platform:** Ubuntu 22.04

---

## Getting Started

### Prerequisites

```bash
sudo apt update && sudo apt install -y gcc build-essential libssl-dev
```

### Building

```bash
make          # Build the pes binary
make all      # Build pes + test binaries
make clean    # Remove all build artifacts
```

### File Inventory

| File               | Role                                 | Your Task                                          |
| ------------------ | ------------------------------------ | -------------------------------------------------- |
| `pes.h`            | Core data structures and constants   | Do not modify                                      |
| `object.c`         | Content-addressable object store     | Implement `object_write`, `object_read`            |
| `tree.h`           | Tree object interface                | Do not modify                                      |
| `tree.c`           | Tree serialization and construction  | Implement `tree_from_index`                        |
| `index.h`          | Staging area interface               | Do not modify                                      |
| `index.c`          | Staging area (text-based index file) | Implement `index_load`, `index_save`, `index_add`  |
| `commit.h`         | Commit object interface              | Do not modify                                      |
| `commit.c`         | Commit creation and history          | Implement `commit_create`                          |
| `pes.c`            | CLI entry point and command dispatch | Do not modify                                      |
| `test_objects.c`   | Phase 1 test program                 | Do not modify                                      |
| `test_tree.c`      | Phase 2 test program                 | Do not modify                                      |
| `test_sequence.sh` | End-to-end integration test          | Do not modify                                      |
| `Makefile`         | Build system                         | Do not modify                                      |

---

## Understanding Git: What You're Building

Before writing code, understand how Git works under the hood. Git is a content-addressable filesystem with a few clever data structures on top. Everything in this lab is based on Git's real design.

### The Big Picture

When you run `git commit`, Git doesn't store "changes" or "diffs." It stores **complete snapshots** of your entire project. Git uses two tricks to make this efficient:

1. **Content-addressable storage:** Every file is stored by the SHA hash of its contents. Same content = same hash = stored only once.
2. **Tree structures:** Directories are stored as "tree" objects that point to file contents, so unchanged files are just pointers to existing data.

```
Your project at commit A:          Your project at commit B:
                                   (only README changed)

    root/                              root/
    ├── README.md  ─────┐              ├── README.md  ─────┐
    ├── src/            │              ├── src/            │
    │   └── main.c ─────┼─┐            │   └── main.c ─────┼─┐
    └── Makefile ───────┼─┼─┐          └── Makefile ───────┼─┼─┐
                        │ │ │                              │ │ │
                        ▼ ▼ ▼                              ▼ ▼ ▼
    Object Store:       ┌─────────────────────────────────────────────┐
                        │  a1b2c3 (README v1)    ← only this is new   │
                        │  d4e5f6 (README v2)                         │
                        │  789abc (main.c)       ← shared by both!    │
                        │  fedcba (Makefile)     ← shared by both!    │
                        └─────────────────────────────────────────────┘
```

### The Three Object Types

#### 1. Blob (Binary Large Object)

A blob is just file contents. No filename, no permissions — just the raw bytes.

```
blob 16\0Hello, World!\n
     ↑    ↑
     │    └── The actual file content
     └─────── Size in bytes
```

The blob is stored at a path determined by its SHA-256 hash. If two files have identical contents, they share one blob.

#### 2. Tree

A tree represents a directory. It's a list of entries, each pointing to a blob (file) or another tree (subdirectory).

```
100644 blob a1b2c3d4... README.md
100755 blob e5f6a7b8... build.sh        ← executable file
040000 tree 9c0d1e2f... src             ← subdirectory
       ↑    ↑           ↑
       │    │           └── name
       │    └── hash of the object
       └─────── mode (permissions + type)
```

Mode values:
- `100644` — regular file, not executable
- `100755` — regular file, executable
- `040000` — directory (tree)

#### 3. Commit

A commit ties everything together. It points to a tree (the project snapshot) and contains metadata.

```
tree 9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d
parent a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0
author Alice <alice@example.com> 1699900000
committer Alice <alice@example.com> 1699900000

Add new feature
```

The parent pointer creates a linked list of history:

```
    C3 ──────► C2 ──────► C1 ──────► (no parent)
    │          │          │
    ▼          ▼          ▼
  Tree3      Tree2      Tree1
```

### How Objects Connect

```
                    ┌─────────────────────────────────┐
                    │           COMMIT                │
                    │  tree: 7a3f...                  │
                    │  parent: 4b2e...                │
                    │  author: Alice                  │
                    │  message: "Add feature"         │
                    └─────────────┬───────────────────┘
                                  │
                                  ▼
                    ┌─────────────────────────────────┐
                    │         TREE (root)             │
                    │  100644 blob f1a2... README.md  │
                    │  040000 tree 8b3c... src        │
                    │  100644 blob 9d4e... Makefile   │
                    └──────┬──────────┬───────────────┘
                           │          │
              ┌────────────┘          └────────────┐
              ▼                                    ▼
┌─────────────────────────┐          ┌─────────────────────────┐
│      TREE (src)         │          │     BLOB (README.md)    │
│ 100644 blob a5f6 main.c │          │  # My Project           │
└───────────┬─────────────┘          └─────────────────────────┘
            ▼
       ┌────────┐
       │ BLOB   │
       │main.c  │
       └────────┘
```

### References and HEAD

References are files that map human-readable names to commit hashes:

```
.pes/
├── HEAD                    # "ref: refs/heads/main"
└── refs/
    └── heads/
        └── main            # Contains: a1b2c3d4e5f6...
```

**HEAD** points to a branch name. The branch file contains the latest commit hash. When you commit:

1. Git creates the new commit object (pointing to parent)
2. Updates the branch file to contain the new commit's hash
3. HEAD still points to the branch, so it "follows" automatically

```
Before commit:                    After commit:

HEAD ─► main ─► C2 ─► C1         HEAD ─► main ─► C3 ─► C2 ─► C1
```

### The Index (Staging Area)

The index is the "preparation area" for the next commit. It tracks which files are staged.

```
Working Directory          Index               Repository (HEAD)
─────────────────         ─────────           ─────────────────
README.md (modified) ──── pes add ──► README.md (staged)
src/main.c                            src/main.c          ──► Last commit's
Makefile                               Makefile                snapshot
```

The workflow:

1. `pes add file.txt` → computes blob hash, stores blob, updates index
2. `pes commit -m "msg"` → builds tree from index, creates commit, updates branch ref

### Content-Addressable Storage

Objects are named by their content's hash:

```python
# Pseudocode
def store_object(content):
    hash = sha256(content)
    path = f".pes/objects/{hash[0:2]}/{hash[2:]}"
    write_file(path, content)
    return hash
```

This gives us:
- **Deduplication:** Identical files stored once
- **Integrity:** Hash verifies data isn't corrupted
- **Immutability:** Changing content = different hash = different object

Objects are sharded by the first two hex characters to avoid huge directories:

```
.pes/objects/
├── 2f/
│   └── 8a3b5c7d9e...
├── a1/
│   ├── 9c4e6f8a0b...
│   └── b2d4f6a8c0...
└── ff/
    └── 1234567890...
```

### Exploring a Real Git Repository

You can inspect Git's internals yourself:

```bash
mkdir test-repo && cd test-repo && git init
echo "Hello" > hello.txt
git add hello.txt && git commit -m "First commit"

find .git/objects -type f          # See stored objects
git cat-file -t <hash>            # Show type: blob, tree, or commit
git cat-file -p <hash>            # Show contents
cat .git/HEAD                     # See what HEAD points to
cat .git/refs/heads/main          # See branch pointer
```

---

### Architecture Overview

```
┌───────────────────────────────────────────────────────────────┐
│                      WORKING DIRECTORY                        │
│                  (actual files you edit)                       │
└───────────────────────────────────────────────────────────────┘
                              │
                        pes add <file>
                              ▼
┌───────────────────────────────────────────────────────────────┐
│                           INDEX                               │
│                (staged changes, ready to commit)              │
│                100644 a1b2c3... src/main.c                    │
└───────────────────────────────────────────────────────────────┘
                              │
                       pes commit -m "msg"
                              ▼
┌───────────────────────────────────────────────────────────────┐
│                       OBJECT STORE                            │
│  ┌───────┐    ┌───────┐    ┌────────┐                         │
│  │ BLOB  │◄───│ TREE  │◄───│ COMMIT │                         │
│  │(file) │    │(dir)  │    │(snap)  │                         │
│  └───────┘    └───────┘    └────────┘                         │
│  Stored at: .pes/objects/XX/YYY...                            │
└───────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────┐
│                           REFS                                │
│       .pes/refs/heads/main  →  commit hash                    │
│       .pes/HEAD             →  "ref: refs/heads/main"         │
└───────────────────────────────────────────────────────────────┘
```

---
**Student:** Kundhan  
**SRN:** PES1UG24AM327  
**Course:** Operating Systems — Unit 4 Lab  
**Repository:** PES1UG24AM327-pes-vcs

---

## Overview

PES-VCS is a minimal content-addressable version control system built in C, modelled on Git's internal design. It stores all data (blobs, trees, commits) as SHA-256-hashed objects in a sharded on-disk store, supports atomic index staging, and maintains a linked commit chain via a HEAD reference.

The four core components implemented in this lab are:

| File | What was implemented |
|---|---|
| `object.c` | `object_write` — hash, shard, atomic write; `object_read` — integrity-verified read |
| `tree.c` | `tree_from_index` — recursive subtree builder from the staged index |
| `index.c` | `index_load`, `index_save` (atomic + sorted), `index_add` (blob staging) |
| `commit.c` | `commit_create` — tree snapshot + parent chain + HEAD update |

---

## Building and Running

```bash
# Install dependencies (Ubuntu/Debian)
sudo apt update && sudo apt install -y gcc build-essential libssl-dev

# Build everything
make all

# Run unit tests
./test_objects       # Phase 1 — object store
./test_tree          # Phase 2 — tree serialization

# Run integration test
bash test_sequence.sh

# Use the CLI
./pes init
./pes add <file>
./pes status
./pes commit -m "message"
./pes log
```

Set your author identity before committing:
```bash
export PES_AUTHOR="Your Name <PESXUG24CSYYY>"
```

---

## Phase 1 — Object Store (`object.c`)

### Design

Every object (blob, tree, or commit) is stored as:

```
"<type> <size>\0<raw data>"
```

The SHA-256 hash of this full byte sequence is the object's identity. Objects are sharded into two-character prefix directories under `.pes/objects/XX/` to avoid filesystem limits on directory entry counts — the same approach used by real Git.

**Write path (`object_write`):**
1. Build the header string `"blob 42\0"` and prepend it to the data
2. Compute SHA-256 of the combined buffer
3. Return early if the object already exists (deduplication)
4. `mkdir` the shard directory
5. Write to a `.tmp` file, `fsync`, then `rename` atomically into place
6. `fsync` the directory so the rename is durable

**Read path (`object_read`):**
1. Locate the object file from its hash
2. Read the entire file
3. Re-compute SHA-256 and reject the file if it doesn't match (integrity check)
4. Parse the header to recover the type and data length

### Phase 1 Test — `./test_objects`

<img width="976" height="180" alt="undhan kundhan-Apple-Virtwaltzatton-Genertc•PLatfornPES1UG24AM327-pes-vcss test_objects" src="https://github.com/user-attachments/assets/e89a78d9-cc8c-417c-bd30-ad8fd837c5fb" />

All three checks pass: blob storage, deduplication (writing the same content twice produces one file), and integrity verification (tampered object is rejected).

### Phase 1 — Object Store After `find`

<img width="1058" height="117" alt="kondhangkondhas-Apple-victvsltzattes-Cenertc-Platfern-PES10C244X127-pes-xcisflod pesobjects-typef" src="https://github.com/user-attachments/assets/0b4481d8-3814-4a47-9669-cf11565b7f8a" />


The shard prefix (`25/`, `2a/`, `d5/`) is the first two hex characters of the hash. Three objects correspond to one blob (the file content), one tree, and one commit.

---

## Phase 2 — Tree Objects (`tree.c`)

### Design

A tree object is the VCS equivalent of a directory snapshot. It records a sorted list of `(mode, name, hash)` entries, where each entry points either to a blob (file) or to another tree (subdirectory).

`tree_from_index` works recursively:

1. Load all staged index entries (already sorted by path)
2. Walk entries in order; for each entry whose relative path contains a `/`, group all entries sharing that subdirectory prefix
3. Recursively call `write_tree_level` on the grouped subset, producing a sub-tree object
4. For entries with no `/` in their relative path, create a blob `TreeEntry` directly
5. Serialize the resulting `Tree` struct and call `object_write(OBJ_TREE, ...)`

This bottom-up approach means every subdirectory tree is written before its parent, matching Git's behaviour.

### Phase 2 Test — `./test_tree`

<img width="392" height="94" alt="PASS tree serializeparse roundtrip" src="https://github.com/user-attachments/assets/06cb4fd7-f301-4e21-88fc-9503bd300109" />


Both checks pass: the tree serialises and parses back to identical data (round-trip), and serialising the same tree twice produces the same bytes (determinism, required for content-addressing).

### Phase 2 — Raw Object Inspection with `xxd`

<img width="1470" height="316" alt="Pasted Graphic 4" src="https://github.com/user-attachments/assets/df7781ea-d74d-4e97-b2bd-12f88e545614" />


The `xxd` dump shows the raw on-disk format: `blob 6.hello.` — the header (`blob 6\0`) immediately followed by the file content (`hello\n`). The `.` in the dump represents the `\0` null byte separator between header and data.

---

## Phase 3 — Index / Staging Area (`index.c`)

### Design

The index is a plain-text file at `.pes/index`, one entry per line:

```
<octal-mode> <hex-hash> <mtime-seconds> <size-bytes> <path>
```

**`index_load`** — parses this file with `fscanf`, initialising an empty index if the file doesn't exist yet (first `add` on a fresh repo).

**`index_save`** — sorts entries by path via a pointer array (sorting pointers rather than copying the full `Index` struct avoids a 5.6 MB stack allocation), writes to a `.tmp` file, calls `fsync` + `fflush`, then `rename`s atomically over the old index.

**`index_add`** — reads the target file's bytes, calls `object_write(OBJ_BLOB, ...)` to store them, calls `lstat` to get `mtime` and `size`, then upserts the entry (updates existing path if already staged, appends otherwise). Calls `index_save` to persist.

### Phase 3 — `./pes init` + `./pes add` + `./pes status`

<img width="428" height="560" alt="Initialized empty PES repository in •pes" src="https://github.com/user-attachments/assets/ce0e3199-9505-43d9-9a1b-8f6cc0285dfd" />


After `./pes add file1.txt file2.txt`, both files appear as `staged` in the status output. Untracked files (the source files of the project itself) are also listed correctly.

### Phase 3 — Index File Contents

<img width="967" height="97" alt="pes-yess cat ipesindex" src="https://github.com/user-attachments/assets/19f16ceb-3e27-4fa1-b9c7-18468659e506" />


Each line shows: `100644` (regular file mode), SHA-256 hash, mtime epoch timestamp, file size in bytes, and the file path. The two files are sorted alphabetically.

---

## Phase 4 — Commits (`commit.c`)

### Design

`commit_create` ties everything together:

1. Call `tree_from_index` to snapshot all staged files into a tree object
2. Attempt `head_read` to get the current HEAD commit as the parent; first commit has no parent
3. Populate a `Commit` struct with tree hash, optional parent hash, author string (from `PES_AUTHOR` env var), Unix timestamp, and message
4. Serialize and write as `OBJ_COMMIT` via `object_write`
5. Call `head_update` to atomically write the new commit hash to `.pes/refs/heads/main` and update `.pes/HEAD`

### Phase 4 — Three Commits + `./pes log`

<img width="1231" height="721" alt="hat-Aole-wirtwal(zatten-Gesertc-Platfeent-PES1UC2444)27-pes-ycssexportPES AuTiORe Kandhan dFS1U02444327" src="https://github.com/user-attachments/assets/bc5145f6-3de8-4b6f-9585-95c3b0736d86" />


Three commits are shown in reverse chronological order, each with a full 64-character SHA-256 hash, author identity (`Kundhan <PES1UG24AM327>`), Unix timestamp, and message.

### Phase 4 — Reference Chain

<img width="1042" height="347" alt="kutdhankunchan-Apple-Virtualizatton-Generic-PLatTorn-PES100244027-pes-yc55 Ftnd -pes «Lype F l sore" src="https://github.com/user-attachments/assets/8a9f74ed-4fee-42d4-a266-54481f717dfc" />


`cat .pes/refs/heads/main` shows the latest commit hash. `cat .pes/HEAD` shows `ref: refs/heads/main`, confirming the indirection chain: HEAD → branch ref → commit hash.

### Phase 4 — Full Object Store

<img width="1038" height="123" alt="dhanakundhan-Apple-Virtualizatton-Generic-Platforn-PES1UG24A327-pes-vcss cat  pesretsheadsnatn" src="https://github.com/user-attachments/assets/9067e101-99ad-48f7-a2d6-6410e3f136f9" />


After three commits to three files, the store contains 12 objects: 3 commits + 3 trees (one per commit snapshot) + 3 blobs for unique file versions (unchanged files deduplicate). `.pes/HEAD`, `.pes/index`, and `.pes/refs/heads/main` are the only non-object files.

---

## Integration Test — `bash test_sequence.sh`

The integration test exercises the full workflow end-to-end in an isolated temp directory: `init` → `add` → `status` → three commits → `log` → reference chain inspection → object store count.

### Integration Test — Part 1 (init → first commit)

<img width="725" height="802" alt="== Running integration tests ass" src="https://github.com/user-attachments/assets/4366840a-807f-4c11-8b9b-124580ca1e97" />


Repository initialisation passes all three structure checks. Staging and status work correctly. First commit is created and appears correctly in `log`.

### Integration Test — Part 2 (full history + reference chain)

<img width="725" height="642" alt="••• Third Cornit " src="https://github.com/user-attachments/assets/1f449963-3186-49fc-abc6-7fddbc41247c" />


Full three-commit history is shown in `./pes log`. The reference chain is correct: `HEAD` points to `refs/heads/main`, which contains the latest commit hash.

### Integration Test — Part 3 (object store + completion)

<img width="790" height="371" alt="••• Object Store •" src="https://github.com/user-attachments/assets/450a2c70-4b63-467e-92b3-8f1df5c03c67" />


10 objects were created across three commits (some blobs are shared/deduplicated between commits). The test completes with `=== All integration tests completed ===`.

---

## Analysis Questions

### Q5.1 — How would `pes checkout <branch>` work?

A checkout reads the tip commit hash from `.pes/refs/heads/<branch>`, then follows the commit's tree hash to reconstruct the file tree, and writes every blob file to the working directory. In `.pes/`, the operation updates `HEAD` to point to the new branch (`ref: refs/heads/<branch>`). The complexity comes from three sources. First, the working directory must be updated file-by-file: files present in the new tree but not the old must be created, files present in the old tree but not the new must be deleted, and files that differ must be overwritten. Second, the index must be updated to match the new tree (every entry's hash, mtime, and size must reflect the checked-out version). Third, a "dirty working directory" — files modified since the last `add` — must be detected and either blocked or stashed before the switch, otherwise those changes would be silently overwritten or left as phantom modifications against the new branch.

### Q5.2 — Detecting a dirty working directory using only the index and object store

At checkout time, walk every entry in `.pes/index`. For each entry, call `lstat` on the working-directory path and compare the result against the stored `mtime_sec` and `size`. If either differs, `stat` suggests a modification; to confirm, read the file, hash it as a blob, and compare against the stored hash. If any entry's hash doesn't match, the file is "dirty" and the checkout should be blocked (or require `--force`). Additionally, any file present in the working directory that matches a path in the current tree but is absent from the index counts as an untracked conflict and should be flagged. This approach requires only `lstat`, `read`, and `SHA-256` — no additional metadata beyond what the index already records.

### Q5.3 — Detached HEAD

Detached HEAD is the state where `HEAD` contains a raw commit hash directly instead of an indirect reference like `ref: refs/heads/main`. It occurs when you check out a specific commit by hash, or check out a tag. If you make a new commit in detached HEAD state, the commit is written to the object store and `HEAD` is updated to the new hash — but no branch ref is updated. The commit is effectively orphaned: it exists in the object store but is not reachable from any named branch. It can be lost as soon as HEAD moves again. Recovery involves noting the orphan commit's hash (visible in the shell prompt if your tool prints it), then running `pes checkout -b <new-branch>` (or the equivalent) to create a branch pointing at that hash before moving away. If the hash is lost, the commit can still be found via a brute-force scan of the object store until garbage collection removes it.

---

## Garbage Collection Analysis (Q6)

### Q6.1 — Garbage Collection Algorithm

A GC algorithm for PES-VCS works in two phases: mark and sweep.

In the mark phase, start from every named ref (all files under `.pes/refs/`) and `HEAD`. For each starting commit hash, read the commit object to get its tree hash and parent hash; add all three to a reachable set (a hash set, implemented as a hash table mapping `ObjectID → bool`). Recurse on the parent commit; recursively walk the tree object to collect all blob and sub-tree hashes. Repeat until all reachable objects are marked.

In the sweep phase, enumerate every file under `.pes/objects/`. For each file, reconstruct its hash from its path (strip the shard prefix). If the hash is not in the reachable set, delete the file.

For 100,000 commits across 50 branches with an average tree of 200 files: each commit touches 1 commit object + 1 tree object + ~200 blob objects. If blobs are heavily deduplicated (typical for source trees), the reachable set might contain ~100,000 commits + ~100,000 trees + ~500,000 unique blobs ≈ 700,000 objects visited. The hash set lookup is O(1), so the total cost is O(total reachable objects + total objects on disk).

### Q6.2 — Race condition between GC and commit

The race: suppose GC runs concurrently with a `commit`. The commit calls `tree_from_index`, which calls `object_write` to store blob objects. At the moment GC runs its mark phase, those freshly written blobs are not yet referenced by any commit (the commit object hasn't been written yet), so GC's traversal from named refs does not reach them. GC then sweeps and deletes the blobs. When the commit finally calls `object_write(OBJ_COMMIT, ...)` and `head_update`, the commit object now references blob hashes that no longer exist on disk. The repository is silently corrupted.

Real Git avoids this with a two-part strategy. First, it uses "loose object" grace periods — objects newer than a configurable threshold (default 14 days) are never swept, giving any in-flight write time to be referenced. Second, Git's `gc` is designed to be run manually or via a heuristic auto-trigger, never truly concurrently with a write; it acquires a repository lock (`gc.pid`) at the start and aborts if another writer holds a lock. Together, the grace period handles the time gap between writing a child object and writing its parent reference, while the lock prevents two concurrent GC runs from racing each other.

---

## Repository Structure

```
.
├── object.c          # Phase 1 — content-addressable object store
├── tree.c            # Phase 2 — tree object builder
├── index.c           # Phase 3 — staging area (index)
├── commit.c          # Phase 4 — commit creation
├── pes.c             # CLI entry point (provided)
├── pes.h             # PES-VCS public API (provided)
├── object.h / tree.h / index.h / commit.h
├── test_objects.c    # Phase 1 unit tests
├── test_tree.c       # Phase 2 unit tests
├── test_sequence.sh  # End-to-end integration test
├── Makefile
└── docs/
    └── screenshots/  # All terminal output screenshots
```
