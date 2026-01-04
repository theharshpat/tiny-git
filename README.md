# Tiny Git - CLI File Versioning System

A minimal file versioning system using content-addressed snapshots, inspired by Git.

## Tech Stack

- **clap**: CLI argument parsing (subcommands, flags)
- **sha1**: Content hashing
- **flate2**: Zlib compression
- **serde/serde_json**: Object serialization
- **chrono**: Timestamps
- **anyhow**: Error handling

## Implementation Phases

**Phase 1: Objects & Hashing** (Working: Store & retrieve data)
- Implement Blob, Tree, Commit structs
- SHA-1 hashing
- Zlib compression
- Write objects to `.git/objects/`
- Read objects back (deserialize)
- **Test**: Create a blob, hash it, compress it, write it, read it back

**Phase 2: Repository & HEAD** (Working: Track current state)
- Initialize `.git` directory structure
- Create HEAD file pointing to master
- Implement Repository struct managing all objects
- **Test**: Init repo, verify .git files exist

**Phase 3: Basic Commands** (Working: Functional git operations)
- `git init` - initialize repository
- `git add` - update index with file hashes
- `git commit` - create commit from index
- **Test**: Add file, commit, verify objects created

**Phase 4: Status & History** (Working: Introspection)
- `git status` - compare working dir vs HEAD
- `git log` - display commit history
- **Test**: Make changes, see status, view history

**Phase 5: Checkout & Branching** (Complete implementation)
- `git checkout` - load specific commit into working dir
- `git branch` - create/list branches
- Additional commands as needed

---

## Core Concepts

### 1. **Hash** (SHA-1)
Fingerprint of content. Same content = same hash.

### 2. **Object** (Blob, Tree, Commit)
**Blob**: File content
**Tree**: Directory (maps filenames to blobs/trees)
**Commit**: Snapshot metadata (tree, parent, author, message)

### 3. **Head**
Pointer to current commit.

### 4. **Index** (Staging Area)
Buffer between working directory and repository.

## Rust Entities

```rust
struct Blob {
    content: Vec<u8>,
}

struct TreeEntry {
    name: String,
    hash: String,    // SHA-1
    is_dir: bool,
}

struct Tree {
    entries: HashMap<String, TreeEntry>,
}

struct Commit {
    tree_hash: String,
    parent_hash: Option<String>,
    author: String,
    timestamp: u64,
    message: String,
}

struct Repository {
    objects: HashMap<String, Object>,
    head_ref: String,  // path to current branch
}

enum Object {
    Blob(Blob),
    Tree(Tree),
    Commit(Commit),
}
```

## Visualizations

### Objects & Storage

```
Blob: File Content
────────────────
hash: 8ab2f
content: "Hello World"

Tree: Directory
───────────────
hash: 4f1a2
entries:
  main.rs → blob:8ab2f
  README.md → blob:c3d9e

Commit: Snapshot
────────────────
hash: 3a7b2
tree: 4f1a2
parent: (none)
author: alice
message: "Initial commit"
```

### Commit Chain

```
3a7b2 → 5f8e1 → 7d2c9
  ↑
 HEAD

(linked list of commits)
Each commit points to previous via parent_hash
```

## Storage: Inside .git

```
.git/
├── HEAD                          # ref: refs/heads/master
├── objects/
│   ├── XX/                       # 2-char hash prefix (sharding)
│   │   └── YYYYYY...             # 38-char hash suffix
│   └── ...
└── refs/heads/
    └── master                    # commit hash
```

**HEAD**: Text file pointing to current branch
```
ref: refs/heads/master
```

**refs/heads/master**: Text file with commit hash
```
dc4345380bff1a15dadfc54da09f6f59a8aa0c27
```

**objects/XX/YYYYYY**: Zlib-compressed object
- Type + data → hash → compress → store by first 2 chars

Example from your repo:
```
.git/HEAD
  → refs/heads/master (text pointer)
  → dc434... (commit hash)
     ↓
.git/objects/dc/434... (zlib file)
  [decompressed content]
  tree: 5fa9bc...
  parent: (none)
  author: theharshpat
  message: "init"
```

## Workflow

1. **Add**: File → Index (mark for commit)
2. **Commit**: Index → create Tree/Blob objects → create Commit → move Head
3. **Status**: Compare working directory vs Index vs Head
4. **Checkout**: Load Tree from a Commit → populate working directory
