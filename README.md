# Tiny Git - Snapshot-Based File Versioning

A minimal file versioning system using content-addressed snapshots, inspired by Git.

## Core Concepts

### 1. **Hash** (`SHA-1`)
Unique fingerprint of content. Same content = same hash. Different content = different hash.
- Used to identify objects uniquely
- Enables deduplication (no storing identical content twice)

### 2. **Object** 
The fundamental unit of storage. Types:
- **Blob**: Stores file content (the actual data)
- **Tree**: Stores a directory snapshot (filename → blob/tree mapping)
- **Commit**: Stores metadata about a snapshot (tree pointer, parent commit, message, timestamp)

Objects are immutable and identified by their hash.

### 3. **Snapshot**
A complete state of the project at one point in time.
- Represented by a **Commit** object
- Points to a **Tree** object (root directory structure)
- Contains metadata: author, timestamp, message

### 4. **Head**
Pointer to the current snapshot (which commit we're on).
- Allows checking out and tracking which version is active
- Can point to different commits when switching versions

### 5. **Index** (Staging Area)
Workspace between working directory and repository.
- Tracks which files have changed
- Allows selecting specific changes to commit

### 6. **Content Addressing**
Objects stored by their hash, not by name.
- Deduplication automatic: identical content referenced once
- Corruption detection: hash mismatch reveals tampering
- Enables efficient storage and retrieval

## Rust Entities

```rust
// Hash identifier for objects (SHA-1)
type Hash = String;  // 40-char hex string

// File content
struct Blob {
    hash: Hash,
    content: Vec<u8>,
}

// Directory entry mapping names to objects
struct TreeEntry {
    name: String,
    hash: Hash,
    is_dir: bool,  // true = tree, false = blob
}

// Directory snapshot
struct Tree {
    hash: Hash,
    entries: HashMap<String, TreeEntry>,
}

// Version snapshot
struct Commit {
    hash: Hash,
    tree_hash: Hash,         // points to root tree
    parent_hash: Option<Hash>, // previous commit (None for initial)
    message: String,
    author: String,
    timestamp: u64,
}

// Current active snapshot
struct Head {
    current_commit: Hash,
}

// Working area tracking
struct Index {
    entries: HashMap<String, FileState>,
}

enum FileState {
    Modified(Hash),  // changed, hash of current content
    Staged(Hash),    // ready to commit
    Untracked,       // not in repo
}

// The repository
struct Repository {
    objects: HashMap<Hash, Object>,  // all blobs, trees, commits
    head: Head,
    index: Index,
}

enum Object {
    Blob(Blob),
    Tree(Tree),
    Commit(Commit),
}
```

## Visualizations

### 1. Hash - Content Addressing

```
File: "hello.txt"
Content: "Hello World"
         ↓
     [SHA-1 Hash]
         ↓
Hash: a0c3b9c1d2e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8

Same Content = Same Hash (Deduplication!)
─────────────────────────────────────────────
File A: "Hello World" → a0c3b9c1d2e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8
File B: "Hello World" → a0c3b9c1d2e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8
                       (Same hash = store once!)

Different Content = Different Hash
──────────────────────────────────
File A: "Hello World"  → a0c3b9c1d2e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8
File B: "Hello World!" → b1d4c0a2e3f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0
```

### 2. Objects - The Building Blocks

```
BLOB (File Content)
────────────────────────────────
Hash: 8ab2f   Content: "README.md file data..."

BLOB (File Content)
────────────────────────────────
Hash: c3d9e   Content: "src code here..."

TREE (Directory)
────────────────────────────────
Hash: 4f1a2
Entries:
  README.md → blob:8ab2f
  src/      → tree:7e4b3

TREE (src/ directory)
────────────────────────────────
Hash: 7e4b3
Entries:
  main.rs → blob:c3d9e
```

### 3. Deduplication Through Content Addressing

```
Project with 2 identical files:
────────────────────────────────

Working Directory:
  readme.txt (content: "Installation steps")
  docs/guide.txt (content: "Installation steps")

Hash Both → Same hash: 9x9f2

Repository (single blob stored):
  blob:9x9f2 ← "Installation steps"
     ↑         ↑
     └─────────┘
      (referenced by both files)

No duplication! Both files point to same blob.
```

### 4. Snapshot Timeline - Commits

```
Initial State: Empty

Commit 1
────────────────────────────────
hash: 3a7b2 (parent: none)
tree: root_v1
├── main.rs (blob: 8ab2f)
├── README.md (blob: c3d9e)
message: "Initial commit"
author: alice
time: 2024-01-01 10:00

        ↓ (user modifies main.rs)

Commit 2
────────────────────────────────
hash: 5f8e1 (parent: 3a7b2)
tree: root_v2
├── main.rs (blob: 9x9f2) ← NEW VERSION
├── README.md (blob: c3d9e) ← SAME (reused!)
message: "Fix bug in main"
author: alice
time: 2024-01-01 11:30

        ↓ (user adds config.yaml)

Commit 3
────────────────────────────────
hash: 7d2c9 (parent: 5f8e1)
tree: root_v3
├── main.rs (blob: 9x9f2) ← SAME
├── README.md (blob: c3d9e) ← SAME
└── config.yaml (blob: 1a4b5) ← NEW
message: "Add config"
author: alice
time: 2024-01-01 12:15

Legend: [parent] → [child]
3a7b2 → 5f8e1 → 7d2c9
(linked list of commits)
```

### 5. Head Pointer - Where We Are

```
Commit Timeline:
─────────────────────────────────────────

3a7b2 → 5f8e1 → 7d2c9 → 9f4a1
(Initial) (Fix) (Config) (Latest)
                          ↑
                        HEAD
                    (We are here)

When you add/commit:
     HEAD moves →
        ↓
3a7b2 → 5f8e1 → 7d2c9 → 9f4a1 → 2b5c3 (NEW)
                                     ↑
                                   HEAD
```

### 6. Index (Staging Area) - The Buffer

```
State at different stages:

Working Directory       Index           Repository (HEAD)
─────────────────────────────────────────────────────────

main.rs (v1)          (empty)         3a7b2 [tree]
README.md             README.md       ├── main.rs (blob)
                                      └── README.md (blob)


Step 1: Edit main.rs
───────────────────

main.rs (v2) ✎       (empty)         (unchanged)
README.md             README.md


Step 2: Add main.rs to Index
──────────────────────────────

main.rs (v2)          main.rs (v2)    (unchanged)
README.md             README.md


Step 3: Commit
──────────────

main.rs (v2)          (cleared)       5f8e1 [tree]
README.md             README.md       ├── main.rs (v2)
                                      └── README.md


Legend:
✎ = modified in working dir
(empty) = nothing staged yet
```

### 7. Full Workflow Example

```
SCENARIO: Developer makes changes and commits

┌─────────────────────────────────────────────────────┐
│ Initial State                                        │
├─────────────────────────────────────────────────────┤
│ Working Dir:    main.rs (Hello World)               │
│ Index:          (empty)                             │
│ HEAD:           3a7b2 [Commit 1]                    │
│                 └─ tree: a2f8b                      │
│                    └─ main.rs: blob:9abc1           │
└─────────────────────────────────────────────────────┘

$ git add main.rs
       ↓
┌─────────────────────────────────────────────────────┐
│ After Add                                            │
├─────────────────────────────────────────────────────┤
│ Working Dir:    main.rs (Hello World)               │
│ Index:          main.rs: blob:9abc1                 │
│ HEAD:           (unchanged)                         │
└─────────────────────────────────────────────────────┘

$ git commit -m "Update greeting"
       ↓
┌─────────────────────────────────────────────────────┐
│ After Commit                                         │
├─────────────────────────────────────────────────────┤
│ Working Dir:    main.rs (Hello World)               │
│ Index:          (cleared)                           │
│ HEAD:           5f8e1 [Commit 2] ← NEW              │
│                 ├─ parent: 3a7b2                    │
│                 ├─ tree: c4d7e                      │
│                 │  └─ main.rs: blob:9abc1           │
│                 └─ message: "Update greeting"       │
└─────────────────────────────────────────────────────┘

Repository now has:
  blob:9abc1 (file content)
  tree:c4d7e (directory snapshot)
  commit:5f8e1 (version record)
```

### 8. Multiple File Changes & Deduplication

```
Initial Commit (3a7b2)
├── main.rs (blob:aaa1)
├── utils.rs (blob:bbb2)
└── README.md (blob:ccc3)

User Changes: main.rs + utils.rs
↓

New Commit (5f8e1)
├── main.rs (blob:xxx9) ← CHANGED, new hash
├── utils.rs (blob:yyy8) ← CHANGED, new hash
└── README.md (blob:ccc3) ← UNCHANGED, reused!

Repository Storage:
  aaa1 (old main.rs) - kept for history
  bbb2 (old utils.rs) - kept for history
  ccc3 (README.md) - used by both commits
  xxx9 (new main.rs)
  yyy8 (new utils.rs)

Only 5 blobs needed, not 6!
(Commit 1 can be recreated from: aaa1, bbb2, ccc3)
(Commit 2 created from: xxx9, yyy8, ccc3)
```

## Workflow

1. **Add**: File → Index (mark for commit)
2. **Commit**: Index → create Tree/Blob objects → create Commit → move Head
3. **Status**: Compare working directory vs Index vs Head
4. **Checkout**: Load Tree from a Commit → populate working directory
