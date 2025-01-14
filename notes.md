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