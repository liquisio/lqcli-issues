# LQ CLI Reference

Complete documentation for LQ CLI (`@liquisio/git-cli`).

## Table of Contents

- [Installation](#installation)
- [Configuration](#configuration)
- [Commands](#commands)
- [How It Works](#how-it-works)
- [Safety Features](#safety-features)
- [After Reconnection](#after-reconnection)
- [Limitations](#limitations)
- [Exit Codes](#exit-codes)
- [Troubleshooting](#troubleshooting)

## Installation

```bash
npm install @liquisio/git-cli --cache=/tmp/.npm && export PATH="$PATH:./node_modules/.bin"
```

**Why this command?** Wix Blocks runs in a read-only container where the default npm cache isn't writable. The `--cache=/tmp/.npm` redirects to a writable location, and `export PATH=...` makes `lqcli` available directly.

**Paste tip:** Wix Blocks terminal requires `Ctrl+Shift+V` (Windows/Linux) or `Cmd+Shift+V` (Mac).

**One-time use:** After running `lqcli push` or `lqcli pull`, your `.git` folder is restored. Use normal git commands (`git add`, `git commit`, `git push`) for the rest of your session.

## Configuration

### Config File

Create `lqcli.config.mjs` anywhere inside `src/` — it will be auto-detected.

```javascript
export default {
  // === REQUIRED ===
  git: {
    name: 'Your Name',           // Git commit author name
    email: 'you@example.com',    // Git commit author email
    user: 'github-username',     // GitHub username
    repo: 'repo-name',           // GitHub repository name
  },

  // === OPTIONAL: Git Settings ===
  // git: {
  //   remote: 'origin',           // Git remote name (default: 'origin')
  //   branch: 'main',             // Git branch name (default: 'main')
  //   tempBranch: 'fresh-start',  // Temp branch for clean push
  // },

  // === OPTIONAL: Paths ===
  // paths: {
  //   src: 'src',                    // Directory to sync (default: 'src')
  //   backup: '/tmp/lqcli-backup',   // Backup directory
  // },

  // === OPTIONAL: Backup Settings ===
  // backup: {
  //   prefix: 'src_',             // Backup folder prefix (default: 'src_')
  // },

  // === OPTIONAL: Timeouts ===
  // timeouts: {
  //   git: 10000,                 // Git operations timeout in ms (default: 10s)
  // },

  // === OPTIONAL: Prefixes ===
  // prefixes: {
  //   commitMessage: 'lq:',       // Auto-commit message prefix
  //   historyBranch: 'history_',  // History branch prefix for clean push
  // },

  // === OPTIONAL: Skip Directories ===
  // skipDirectories: ['site', 'node_modules', 'generated'],
};
```

### Configuration Options

| Option | Default | Description |
|--------|---------|-------------|
| `git.name` | (required) | Git commit author name |
| `git.email` | (required) | Git commit author email |
| `git.user` | (required) | GitHub username |
| `git.repo` | (required) | GitHub repository name |
| `git.remote` | `'origin'` | Git remote name |
| `git.branch` | `'main'` | Git branch name |
| `paths.src` | `'src'` | Directory to sync (relative to root) |
| `paths.backup` | `/tmp/lqcli-backup` | Backup directory |
| `backup.prefix` | `'src_'` | Backup folder prefix |
| `timeouts.git` | `10000` | Git operation timeout (ms) |
| `skipDirectories` | `['site', 'node_modules']` | Directories to exclude from sync |

### GitHub Token

Create `.env` in the same directory as your config:

```
LQCLI_TOKEN=github_pat_xxxx
```

The `.env` file is automatically ignored by git and will not be tracked.

**Getting a token:**

1. Visit [github.com/settings/personal-access-tokens](https://github.com/settings/personal-access-tokens)
2. Click "Generate new token"
3. Give it a name (e.g., "LQ CLI" or "Wix Blocks")
4. Repository access: Select **only** the repository you need
5. Permissions: **Repository permissions → Contents: Read and write**
6. Generate and copy to `.env`

**Security:**
- Token is stored in `.env` file (gitignored)
- Config file (`lqcli.config.mjs`) can be committed (no secrets)
- Never commit your `.env` file to version control
- Give token access to specific repos only

### Directory Structure

```
/user-code/                  ← Run lqcli from here (project root)
├── .git/                    ← Created by lqcli
├── .gitignore               ← Created by lqcli
├── src/                     ← Only this directory is synced (sparse checkout)
│   ├── backend/
│   │   ├── lqcli.config.mjs ← Config file (can be anywhere in src/)
│   │   ├── .env             ← Token (gitignored)
│   │   └── *.js
│   └── public/
│       └── *.js
└── (other files ignored)

/tmp/lqcli-backup/           ← Backups stored here
└── src_2026-01-31_14-23-45/

Remote repo (GitHub):
├── docs/                    ← Exists on remote, not checked out locally
├── tests/                   ← Exists on remote, not checked out locally
└── src/                     ← Only this syncs with Wix Blocks
```

**Key points:**
- Run `lqcli` from `/user-code` (project root where `.git` lives)
- Only `src/` directory is synced via **sparse checkout**
- Remote repo can have other folders (`docs/`, `tests/`) — they won't appear locally
- Config file is searched recursively (finds it in `src/backend/`)
- Backups stored in `/tmp/lqcli-backup/` (auto-cleaned on reboot)

## Commands

### Push

Push local code to GitHub:

```bash
lqcli push              # Auto-generated commit message
lqcli push -m "message" # Custom commit message
lqcli push --clean      # Replace remote history entirely
lqcli push -c           # Short form of --clean
lqcli push --logs       # Show detailed output
```

**Normal push** (preserves history):
1. Creates backup of `src/` files
2. Fetches remote history (preserves commit history)
3. Copies your local `src/` files on top
4. Commits with auto-generated message: `lq: push 2026-01-31 14:23:45`
5. Pushes to GitHub

**Clean push** (`--clean` or `-c` flag):
1. Creates backup of `src/` files
2. Saves existing remote history to `history_<timestamp>` branch
3. Replaces remote with local files (fresh commit, no history)
4. Force pushes to GitHub

Use clean push when you want to start fresh or when normal push fails due to diverged histories.

### Pull

Pull GitHub code to local:

```bash
lqcli pull           # Fails if conflicts exist
lqcli pull --ours    # Keep local files on conflict
lqcli pull --theirs  # Keep remote files on conflict
```

**What happens:**
1. Creates backup of `src/` files
2. Checks for conflicts with remote
3. If conflicts exist: **fails with error** (use `--ours` or `--theirs`)
4. Resets to remote HEAD (sparse checkout — only `src/` affected)
5. Preserves local-only files (files not on GitHub)

**Conflict example:**
```
Error: Cannot pull - 3 file(s) would be overwritten:
   src/backend/config.js
   src/backend/utils.js
   src/backend/api.js

Use 'lqcli push' to save local changes first, or use '--ours' / '--theirs'.
```

### Status

Show differences between local and remote (read-only, doesn't modify anything):

```bash
lqcli status         # Show file differences
lqcli status --logs  # Show detailed diff information
```

**Note:** `status` requires git to already be initialized. Run `push` or `pull` first.

### Restore

Restore files from the latest backup:

```bash
lqcli restore
```

Backups are stored in `/tmp/lqcli-backup/src_YYYY-MM-DD_HH-MM-SS/`.

### Config

Manage configuration:

```bash
lqcli config         # Show current configuration (default)
lqcli config show    # Show current configuration
lqcli config init    # Generate config file templates
lqcli config path    # Show path to loaded config file
```

### Global Options

```bash
lqcli --logs     # Verbose/debug output
lqcli --version  # Show version
lqcli --help     # Show help
```

## How It Works

### Sparse Checkout

Only `src/` is synced. Your remote repository can have other directories (`docs/`, `tests/`, etc.) that won't appear locally.

```bash
# Under the hood:
git sparse-checkout init --cone
git sparse-checkout set src/
```

### Automatic Backups

Every operation creates a timestamped backup:

```
/tmp/lqcli-backup/src_2026-01-31_14-23-45/
```

Backups live in `/tmp` and are auto-cleaned on system reboot.

### Conflict Detection

Pull compares local files against remote:
- **Identical files** — no conflict
- **Different content** — conflict (pull fails)
- **Local-only files** — preserved after pull
- **Remote-only files** — added to local

## Safety Features

- **Automatic Backups:** Creates timestamped backup in `/tmp` before any operation
- **Atomic Operations:** Push uses temp directory snapshot; restores on failure
- **Token Security:** Stored in gitignored `.env` file; fully redacted in logs
- **Conflict Detection:** Pull fails safely on conflicts (no silent overwrites)
- **Conflict Resolution:** Use `--ours` or `--theirs` flags to resolve conflicts
- **Read-only Status:** `status` command never modifies files or initializes git
- **Sparse Checkout:** Only syncs `src/` — remote docs/tests stay on remote only
- **Backup Restore:** Easy recovery with `lqcli restore`
- **Site Folder Preserved:** `site/` folder is never touched (configurable via skipDirectories)
- **Corrupted Git Recovery:** Automatically detects and fixes corrupted `.git` directories
- **UTF-16 Support:** Properly handles UTF-16 encoded files (not falsely detected as binary)

## After Reconnection

Once you run `lqcli push` or `lqcli pull`, your repository is fully connected. Use standard git commands for the rest of your session:

```bash
git status                        # Check changes
git add . && git commit -m "msg"  # Commit changes
git push                          # Push to GitHub
git pull                          # Pull from GitHub
git log                           # View history
git branch                        # Manage branches
git checkout -b feature           # Create branches
git merge feature                 # Merge branches
```

lqcli is only needed again when your container resets and `.git` disappears.

## Limitations

> **Note:** These limitations apply to lqcli commands only. After reconnection, all standard git commands work normally.

### Git & Workflow

| Limitation | Description |
|------------|-------------|
| **Single branch only** | Works with one branch (default: `main`). No branching workflows or branch switching |
| **GitHub only** | Only supports GitHub repositories. No GitLab, Bitbucket, or self-hosted Git servers |
| **No merge support** | Uses reset/overwrite strategy, not proper git merge |
| **No stash** | Can't temporarily stash changes |
| **No commit history** | Can't view commit logs or history |
| **Single remote** | Only supports one remote (`origin`) |
| **No submodules** | Git submodules not supported |
| **No LFS** | Git Large File Storage not supported |

### Sync & Conflicts

| Limitation | Description |
|------------|-------------|
| **Single directory sync** | Only syncs one directory (`src/` by default). Cannot sync multiple directories independently |
| **All-or-nothing conflicts** | `--ours` or `--theirs` applies to all files. No per-file conflict resolution |
| **No diff viewing** | `status` shows which files differ, but not the actual content differences |
| **Remote structure** | Sparse checkout expects `src/` directory at remote root level |

### Environment

| Limitation | Description |
|------------|-------------|
| **Wix Blocks specific** | Designed for Wix Blocks VS Code web containers; may work elsewhere but untested |
| **Ephemeral backups** | Backups in `/tmp` are lost on system reboot |
| **10-second timeout** | Default git timeout may be insufficient for large repositories or slow networks |
| **Single developer** | No multi-user collaboration features (no pull requests, code review, etc.) |

### Authentication

| Limitation | Description |
|------------|-------------|
| **PAT tokens only** | No SSH key support, no OAuth, no GitHub App authentication |
| **Fine-grained tokens recommended** | Classic tokens work but fine-grained are more secure |

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | General error |
| 10 | Config file missing |
| 11 | Config file invalid |
| 12 | Config error (e.g., running inside lqcli source) |
| 20 | Git not installed |
| 21 | Git operation error |
| 30 | Conflict detected |
| 40 | Authentication error |
| 50 | Backup error |

## Troubleshooting

### "Config file not found"

Ensure `lqcli.config.mjs` exists somewhere in `src/`. Run `lqcli config init` to generate a template.

### "Authentication failed"

- Check your token in `.env`
- Ensure token has **Contents: Read and write** permission
- Verify token has access to the specific repository

### "Conflicts detected" on pull

Your local files differ from remote. Options:
- `lqcli push` — save your changes first
- `lqcli pull --ours` — keep local versions
- `lqcli pull --theirs` — accept remote versions

### Status shows "Git not initialized"

Run `lqcli push` or `lqcli pull` first to set up the repository.

## Requirements

- Node.js >= 18
- Git installed on system
