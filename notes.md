# Notes for making gip (git in python)

### Creating repositories: init
- Almost everytime we run a git command we're trying to do something to a repository, create it, read from it or modify it
- A git repo is made of two things, a `work tree` where files in version control live and a `git directory` where Git stores its own data. Most cases the worktree is a regular director and git directory is a child directory of the worktree called `.git`

- To create a `Repository` object:
    - Verify the directory exists and contains a subdirectory called .git
    - Read its configuration in `.git/config` (just an INI file) and check `core.repositoryformatversion` is 0.

```
def repo_path(repo, *path):
    """Compute path under repo's gitdir."""
    return os.path.join(repo.gitdir, *path)
```
- `*path` makes the function variadic, meaning it can be called with multiple path components as separate arguments. For example `repo_path(repo, "objects", "df", "4ec9fc2ad990cb9da906a95a6eda6627d7b7b0")` is a valid call where function receives path as a list

- `repo_file()` and `repo_dir()` return and optionally create a path to a file or directory. Difference being file version only creates directories up to the last component.
- `mkdir` arguement must be passed explicitly by name due to variadic nature of the functions `repo_file(repo, "objects", mkdir=True)`

- To **create** a new repo, start with directory and create the **git directory** which must not exist already or be empty.
- That directory being `.git` and contains:
    - `.git/objects/` : the object store
    - `.git/refs/` the reference store. It contains two subdirectories, heads and tags.
    - `.git/HEAD`, a reference to the current HEAD 
    - `.git/config`, the repository’s configuration file.
    - `.git/description`, holds a free-form description of this repository’s contents, is rarely used.

- Configuration file is very simple, INI-like file with a single section `[core]` and three fields:
    - `repositoryformatversion = 0` 0 is initial format, 1 the same with extensions, if > 1 git will panic
    - `filemode = false` disable tracking of file modes (permissions) changes in the work tree
    - `bare = false` indicate the repo has a worktree. git supports optional worktree which indicates the location of the worktree

- `repo_find()` helps find the root of the current repo. This will be used a lot since almost all Git functions work on existing repos (except init). Sometimes the root is the current dir, or it could be a parent
- For example your repository’s root may be in `~/Documents/MyProject`, but you may currently be working in `~/Documents/MyProject/src/tui/frames/mainview/`
- repo_find will look for that root starting at current dir recursing back to `/`
- To identify a path as a repo, it will for the presence of .git directory

### What are objects?
- Let's implement `git hash-object` and `git cat-file`
- `hash-object` converts an existing file into a git object and `cat-file` prints an existing object to the standard output.
- Git is a "content-addressed filesystem". Names of files stored by git are mathematically derived from the contents. If a single byte, changes, its internal name also changes. You dont *modify* a file in git, but you create a new file in a different location.
- Objects are just that: files in the git repo, whose paths are determined by their contents.
- Git uses objects to store quite a lot of things, the files it keeps in version control (source code). Commits are objects, so are tags. Almost everything in Git is stored as an object.
- The path where git stores objects is computed by calculating the SHA-1 hash. Git renders the hash as lowercase hexadecimal and splits it into two parts:
    - The first two characters
    - The rest
    - The first part is a directory name, the rest as a filename. Git's method creates 256 possible intermediate directories
- An object starts with a header that specifies its type:
blob, commit, tag or tree. This header is followed by ASCII space (0x20) then the size of the object in bytes as an ASCII number, then null (0x00), then the contents of the object. The first 48 bytes of a commit object could look like:
```
00000000  63 6f 6d 6d 69 74 20 31  30 38 36 00 74 72 65 65  |commit 1086.tree|
00000010  20 32 39 66 66 31 36 63  39 63 31 34 65 32 36 35  | 29ff16c9c14e265|
00000020  32 62 32 32 66 38 62 37  38 62 62 30 38 61 35 61  |2b22f8b78bb08a5a|
```
- In the first line, we see the type header, space, size in ASCII and the null separator. The last four bytes on the first line are the beginning of the object's contents
- The objects (headers and contents) are stored compressed with `zlib`

### A generic object object
- Objects can be multiple types, but they all share same storage/retrieval mechanism and same header format. We need to abstract common features
- Create a generic `GitObject` with two unimplemented methods: `serialize()` and `deserialize()` and a default `init()`