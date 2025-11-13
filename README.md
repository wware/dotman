# dotman

Track private files and local modifications that don't belong in your shared git repository.

## What Problem Does This Solve?

When working on shared projects, you often need files that shouldn't be committed. While `.gitignore` handles build artifacts (`.o`, `.so`, `.pyc`, `__pycache__/`) and secrets (`.env`), you also need to manage:
- Private debugging scripts and notes
- Local IDE settings (`.vscode/settings.json`)
- Temporary changes to tracked files for your development environment
- Personal configuration tweaks

`dotman` manages these files in a separate private git repository (which can be located anywhere) while keeping them accessible in your working directory.

## How Files Are Managed

dotman automatically discovers and manages all files in your private repository. It decides how to handle each file based on whether it matches a `.gitignore` pattern:

### 1. Private Files (Symlinked)
Files that match patterns in working repository's `.gitignore` are symlinked.

**How it works:**
- Put the file in your private repository (same relative path as it should appear in working repo)
- There should be a matching pattern in `.gitignore` in your working repo
- dotman creates a symlink from the working directory to your private repo

**Example:**
```
$SRC/.vscode/settings.json  →  (symlinked to) $DST/.vscode/settings.json
```

### 2. Local Modifications (Overlaid with Skip-Worktree)
Files that DON'T match `.gitignore` patterns are treated as overlays.

**How it works:**
- Put the file in your private repo
- dotman auto-creates a backup copy in private repo `backups/` (if it doesn't exist)
- The file is applied to your working repo
- If the file is git-tracked, dotman uses `--skip-worktree` to prevent accidental commits
- Your local changes won't show up in `git status` or `git diff`

**Example:**
```
$SRC/config/database.yml  →  (overlays) $DST/config/database.yml
                          →  (backup stored in) $SRC/backups/config/database.yml
```

## Setup

### 1. Create Your Private Repo

```bash
git clone <your-private-repo-url> <path-to-private-repo>
cd <path-to-private-repo>
```

### 2. Configure Paths

Create a `.dotmanrc` file (recommended) or use environment variables:

**Option A: .dotmanrc file**
```bash
cat > .dotmanrc << EOF
SRC=/path/to/your/private-repo
DST=/path/to/your/work-project
EOF
```

**Option B: Environment variables**
```bash
export DOTMAN_SRC=/path/to/your/private-repo
export DOTMAN_DST=/path/to/your/work-project
```

### 3. Apply Your Configuration

```bash
./dotman dirty
```

This creates all symlinks and applies local modifications.

### 4. Optional: Add to PATH

Make sure dotman appears on your path, either copy it (e.g. to /usr/local/bin or ~/.local/bin) or make a symlink.

## Commands

### `dotman dirty`
Apply all private files (symlinks) and local modifications. Run this:
- After initial clone
- After `git pull` on your private repo
- To restore your local environment

### `dotman status`
Show what files are managed:
- ✓ Symlinks in place
- ⚠ Local modifications applied
- ✗ Problems detected

### `dotman diff`
Show differences for files with local modifications.

### `dotman clean`
Remove everything dotman has done:
- Delete all symlinks
- Revert all local modifications to HEAD
- Clear all skip-worktree flags

Use before `git pull` on the shared repo if upstream changed files you've modified locally:
```bash
dotman clean
git pull
dotman dirty
```

## Workflows

### Adding a New Private File

1. Create the file in `$SRC/` with the same relative path:
   ```bash
   mkdir -p $DOTMAN_SRC/scripts
   echo "#!/bin/bash" > $DOTMAN_SRC/scripts/debug.sh
   ```

2. Make sure it matches a `.gitignore` pattern (usually already there):
   ```bash
   # Verify it's in .gitignore
   grep "scripts/debug.sh" $DOTMAN_DST/.gitignore
   ```

3. Run dirty:
   ```bash
   dotman dirty
   ```

The symlink is created automatically.

### Adding a Local Modification

1. Create the file in `$SRC/` with the same relative path:
   ```bash
   mkdir -p $DOTMAN_SRC/config
   cp $DOTMAN_DST/config/database.yml $DOTMAN_SRC/config/database.yml
   # Edit $DOTMAN_SRC/config/database.yml with your changes
   ```

2. Run dirty:
   ```bash
   dotman dirty
   ```

dotman auto-creates the backup and applies your changes with skip-worktree protection.

### Editing Existing Files

**Private files (symlinked):** Just edit them directly - they're symlinked, so changes go straight to your private repo.

**Local modifications (overlaid):**
- Edit in the working directory OR edit in `$SRC/` directory
- If you edit in working directory, copy changes back:
  ```bash
  cp $DOTMAN_DST/config/database.yml $DOTMAN_SRC/config/database.yml
  ```
- Commit to your private repo

### Handling Upstream Changes

If upstream changes a file you've modified locally:

```bash
dotman clean           # Clean slate
git pull               # Get upstream changes
dotman dirty           # Reapply your modifications
```

If there are conflicts, manually merge your `$SRC/` file with the new upstream version.

## How Skip-Worktree Protection Works

For local modifications, dotman uses git's `--skip-worktree` feature:
- The file remains tracked by git
- Local changes are invisible to git (won't show in `git status`)
- Prevents accidental commits of your private tweaks
- All skip-worktree operations are automatic - you never run git commands manually

## Why dotman?

**Before dotman:**
- Manual file lists to maintain
- Complex shell scripts for each project
- Forgetting which files are "special"
- Accidentally committing private changes

**With dotman:**
- Auto-discovery based on directory structure
- Single tool, intuitive commands
- Clear status at a glance
- Impossible to accidentally commit protected files
- Version control for private development files
- Portable setup across machines

## Configuration Files

dotman looks for configuration in this order (first found wins):
1. `.dotmanrc` in current directory
2. Environment variables `DOTMAN_SRC` and `DOTMAN_DST`
3. `.env` file in current directory

## Example Structure

```
<private-repo>/                # Your private repo (SRC points to <private-repo>)
├── .dotmanrc                  # Configuration
├── .vscode/
│   └── settings.json      # In .gitignore → will be symlinked
├── BrainDump.md           # In .gitignore → will be symlinked
├── scripts/
│   └── debug.sh           # In .gitignore → will be symlinked
├── config/
│   └── database.yml       # NOT in .gitignore → will be overlaid
├── backups/               # Auto-created backups for overlaid files
│   └── config/
│       └── database.yml   # Backup of overlay file

<work-project>/                # Your working directory (DST)
├── .gitignore                 # Patterns determine symlink vs overlay
├── .vscode/
│   └── settings.json          # → symlink to <private-repo>/vscode/settings.json
├── BrainDump.md               # → symlink to <private-repo>/BrainDump.md
├── scripts/
│   └── debug.sh               # → symlink to <private-repo>/scripts/debug.sh
└── config/
    └── database.yml           # Modified version applied, skip-worktree enabled
```

## Quick Reference

```bash
# Fresh setup on new machine
git clone <private-repo-url> <path-to-private-repo>
cd <path-to-private-repo>
./dotman dirty

# Daily use
dotman status              # What's managed?
dotman diff                # What did I change?

# Before pulling shared repo changes
dotman clean
git pull
dotman dirty

# View this help
dotman --help
```
