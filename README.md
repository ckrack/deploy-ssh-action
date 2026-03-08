# Deploy via rsync

A composite GitHub Action that deploys a release to a remote server using rsync and SSH. It handles the full release lifecycle: syncing files, symlinking shared resources, running remote commands, activating the release, and cleaning up old releases.

## How it works

1. **Release naming** — Generates a release name based on the git ref (e.g., `prod-v1.2.0`, `staging-2025.03.08-abc1234`, `staging-pr-42-abc1234`)
2. **Rsync** — Syncs the working directory to a new release directory on the server
3. **Shared resources** — Symlinks shared files (e.g., `.env.local`) and directories (e.g., `var/log`) from the project root into the release
4. **Remote commands** — Runs arbitrary shell commands inside the release directory (e.g., `composer install`, cache clearing, migrations)
5. **Activation** — Points the docroot symlink to the new release's `public/` directory
6. **Cleanup** — Optionally removes old releases in the same release group, keeping only the current and previous release

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `project_path` | ✅ | — | Remote project base path (e.g., `/var/www/project`) |
| `docroot` | ✅ | — | Remote docroot symlink path (e.g., `/var/www/html`) |
| `release_group` | ✅ | — | Release group prefix to isolate releases (e.g., `staging`, `prod`) |
| `cleanup` | ❌ | `false` | Remove old releases in the same release group after activation |
| `rsync_switches` | ❌ | — | Flags and options passed to rsync |
| `rsync_debug` | ❌ | `true` | Enable rsync debug output |
| `shared_files` | ❌ | — | Newline-separated list of shared files to symlink into the release |
| `shared_dirs` | ❌ | — | Newline-separated list of shared directories to symlink into the release |
| `remote_commands` | ❌ | — | Shell commands to run inside the release directory after symlinking (before activation) |
| `ssh_host` | ✅ | — | Remote SSH host |
| `ssh_user` | ✅ | — | Remote SSH user |
| `ssh_key` | ✅ | — | SSH private key |
| `ssh_passphrase` | ❌ | — | SSH key passphrase |

## Usage

### Minimal example

```yaml
steps:
  - uses: actions/checkout@v4

  - uses: ckrack/deploy-ssh-action
    with:
      project_path: /var/www/myapp
      docroot: /var/www/html
      release_group: prod
      cleanup: true
      rsync_switches: -avzr --delete
      shared_files: |
        .env
      shared_dirs: |
        var/log
        public/uploads/media
      ssh_host: ${{ secrets.SSH_HOST }}
      ssh_user: ${{ secrets.SSH_USER }}
      ssh_key: ${{ secrets.SSH_KEY }}
```

### Full example with build step and remote commands

```yaml
steps:
  - uses: actions/checkout@v4

  - uses: actions/setup-node@v4
    with:
      node-version-file: '.nvmrc'
      cache: 'yarn'

  - run: |
      yarn install --frozen-lockfile
      yarn build

  - uses: ckrack/deploy-ssh-action
    with:
      project_path: /var/www/myapp
      docroot: /var/www/html
      release_group: prod
      cleanup: true
      rsync_switches: -avzr --delete
      shared_files: |
        .env
      shared_dirs: |
        var/log
        public/uploads/media
      remote_commands: |
        composer install --no-dev --optimize-autoloader --no-interaction --no-progress
        php bin/console cache:clear
        php bin/console doctrine:migrations:migrate --no-interaction
      ssh_host: ${{ secrets.SSH_HOST }}
      ssh_user: ${{ secrets.SSH_USER }}
      ssh_key: ${{ secrets.SSH_KEY }}
      ssh_passphrase: ${{ secrets.SSH_PASS }}
```

### With additional rsync excludes

```yaml
  - uses: ./.github/actions/deploy-rsync
    with:
      project_path: /var/www/myapp
      docroot: /var/www/html
      release_group: prod
      cleanup: true
      rsync_switches: -avzr --delete --exclude='extra-dir/'
      ssh_host: ${{ secrets.SSH_HOST }}
      ssh_user: ${{ secrets.SSH_USER }}
      ssh_key: ${{ secrets.SSH_KEY }}
```

## Release directory structure

```
/var/www/project/
├── .env                        # shared file
├── .env.local                  # shared file
├── var/
│   ├── uploads/media/          # shared directory
│   └── log/                    # shared directory
├── public/
│   └── uploads/media/          # shared directory
└── releases/
    ├── prod-v1.0.0/            # old release (cleaned up)
    ├── prod-v1.1.0/            # previous release (kept)
    └── prod-v1.2.0/            # current release (active)

/var/www/html -> /var/www/project/releases/prod-v1.2.0/public
```
