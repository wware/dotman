# Private Repository Management

This repository manages private, local-only files for any work-related project that should not be committed to the shared repository. It uses the `dotman` tool to automatically discover and manage two types of files:

1. **Private files** - Files that don't belong in the shared repo at all (debugging scripts, notes, local configs)
2. **Local modifications** - Temporary changes to tracked files that should stay local

The `dotman` tool automatically discovers what files to manage by scanning the directory structure and checking `.gitignore` patterns - no manual configuration needed!

## Repository Structure

```
private/
├── .vscode/settings.json          # VSCode settings
└── BrainDump.md                   # Private notes
```

## Setup

### Configuration

`dotman` needs to know two paths:
- **SRC**: Your private repo with dotfiles (e.g., `~/dotfiles/private`)
- **DST**: Your working directory (e.g., `~/work-project`)

Configuration is loaded in this order (first found wins):

1. **`.dotmanrc` file** (recommended for per-project config):
   ```bash
   SRC=/home/wware/dotfiles/private
   DST=/home/wware/work-project
   ```

2. **Environment variables**:
   ```bash
   export DOTMAN_SRC=~/dotfiles/private
   export DOTMAN_DST=~/work-project
   ```

3. **`.env` file**:
   ```bash
   DOTMAN_SRC=/home/wware/dotfiles/private
   DOTMAN_DST=/home/wware/work-project
   ```

### Initial Setup

1. Clone this private repo:
   ```bash
   cd ~
   git clone <private-repo-url> dotfiles
   ```

2. Create configuration (choose one method):
   ```bash
   # Option 1: .dotmanrc file (recommended)
   cd ~/dotfiles
   cat > .dotmanrc << EOF
   SRC=$HOME/dotfiles/private
   DST=$HOME/work-project
   EOF

   # Option 2: Environment variables in your shell rc
   echo 'export DOTMAN_SRC=~/dotfiles/private' >> ~/.bashrc
   echo 'export DOTMAN_DST=~/work-project' >> ~/.bashrc
   ```

3. Run the setup to create symlinks and apply local modifications:
   ```bash
   cd ~/dotfiles
   ./dotman setup
   ```

### Using from PATH

Once configured, you can put `dotman` on your PATH for use anywhere:

```bash
# Copy or symlink to a directory in your PATH
sudo ln -s ~/dotfiles/dotman /usr/local/bin/dotman

# Now use from any directory (if using .dotmanrc in that directory)
cd ~/work-project
dotman status
```

That's it! `dotman` will automatically:
- Discover all files in `private/`
- Create symlinks for files that match `.gitignore` patterns (private files)
- Apply local modifications from `backups/` directory (overlay files)
- Handle all git skip-worktree operations transparently

## Working with Private Files

### Adding New Private Files

1. Create the file in `private/` with the same relative path
2. Add the file to `work-project/.gitignore` (so it's treated as a symlink)
3. Run `./dotman setup` to create the symlink

The tool will automatically discover the new file and create the symlink based on the `.gitignore` pattern.

### Editing Private Files

Just edit them normally - they're symlinked, so changes automatically go to the private repo. Commit and push to preserve your changes.

## Working with Local Modifications

These are files that **are** tracked in the shared repo, but you want to make local changes that shouldn't be committed.

### Viewing Your Local Changes

```bash
./dotman diff
```

Shows a clean diff of all your local modifications.

### Viewing Current Status

```bash
./dotman status
```

Shows which files are symlinked, which have local modifications, and their protection status.

### Reverting to Original State

```bash
./dotman reset           # Reset to clean state
./dotman reset --restore # Reset and immediately restore your changes
```

### Restoring Your Changes

```bash
./dotman restore
```

Reapplies all your local modifications after a reset.

### Adding New Local Modifications

1. Make your changes to a tracked file in `work-project`
2. Copy it to `backups/` with the same relative path in this repo
3. Run `./dotman setup` or `./dotman restore`

The tool automatically detects files in the `backups/` directory and applies them with proper git protection.

### How Local Modifications Work

Local modifications are automatically protected using git's `--skip-worktree` feature, which:
- Keeps files tracked by git but ignores local changes
- Prevents accidental commits of your private modifications
- Allows free editing without git noticing

**All skip-worktree operations are handled automatically by `dotman`** - you never need to run git commands manually.

**Important**: If upstream changes these files (someone else commits to them), you'll need to:
1. `./dotman reset` (to accept upstream changes)
2. `git pull` (to get the latest changes)
3. `./dotman restore` (to reapply your modifications)

## dotman Command Reference

### dotman setup
Sets up symlinks and applies local modifications. Run after cloning or to restore your local changes.

```bash
./dotman setup
```

### dotman status
Shows current status of all symlinks and local modifications.

```bash
./dotman status
```

### dotman diff
Shows diffs for files with local modifications.

```bash
./dotman diff
```

### dotman reset
Reverts local modifications to original state. Use `--restore` to automatically reapply afterwards.

```bash
./dotman reset           # Reset to clean state
./dotman reset --restore # Reset and restore in one step
```

### dotman restore
Reapplies your local modifications after a reset.

```bash
./dotman restore
```

## Quick Reference

```bash
# Fresh setup on new machine
cd ~
git clone <private-repo-url> dotfiles
cd dotfiles
./dotman setup

# See current status
./dotman status

# See what you've changed
./dotman diff

# Reset to clean state and restore changes
./dotman reset --restore

# Just restore your changes
./dotman restore
```

## Benefits

- **Auto-discovery**: No manual file lists to maintain - just add files and they're detected
- **Simple commands**: Single tool with intuitive commands replaces multiple complex scripts
- **Transparent git operations**: All skip-worktree complexity hidden from users
- **Version control for private files**: Track your debugging scripts, notes, and local configs
- **Separate change history**: Your private tweaks don't clutter the shared repo's history
- **Easy reset**: Return to clean state for testing, restore changes instantly
- **Portable setup**: Clone on a new machine and run one command to get working
- **Clear separation**: Physical separation between public and private files

## How It Works

The `dotman` tool provides a much simpler user experience by:

1. **Auto-discovering files** by scanning `private/` directory structure
2. **Using .gitignore patterns** to automatically determine:
   - Files matching `.gitignore` → Create symlinks (private files)
   - Files in `backups/` directory → Apply as overlays (local modifications)
3. **Handling all git operations** transparently (no more manual skip-worktree commands)
4. **Providing clear status** with ✓, ⚠, ✗ indicators for easy troubleshooting
