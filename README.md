# Laravel Herd Worktree

A Claude Code skill that automates setting up git worktrees for Laravel projects served by Laravel Herd.

## What It Does

When you need to work on a feature branch in isolation, this skill:

1. **Creates a git worktree** in `.worktrees/<branch-name>`
2. **Links with Laravel Herd** to serve the worktree at `http://<branch-name>.test`
3. **Configures environment** - copies and updates `.env` with correct URLs, session domains, and Sanctum settings
4. **Installs dependencies** - runs `composer install` and `npm install`
5. **Starts development** - kills stale Vite processes and starts fresh

It also handles cleanup and PR creation when you're done.

## Prerequisites

- **Laravel Herd** installed and running on macOS
- **Git** for version control
- **Vite** as the frontend build tool (skill assumes Vite, not Webpack/Mix)
- **npm** as package manager (adjust commands if using yarn/pnpm)
- **Laravel Sanctum** if using API authentication (optional - skill handles this if present)

## Installation

### Via Claude Plugins (Recommended)

```bash
claude plugins add harris21/laravel-herd-worktree
```

### Manual Installation

1. Clone this repository:
   ```bash
   git clone https://github.com/harris21/laravel-herd-worktree.git ~/.claude/plugins/laravel-herd-worktree
   ```

2. Add to your Claude Code configuration:
   ```bash
   claude plugins add ~/.claude/plugins/laravel-herd-worktree
   ```

## Usage

The skill is automatically invoked when you mention worktrees with Laravel Herd projects. Example prompts:

- "Set up a worktree for feature-login"
- "Create an isolated workspace for this branch"
- "I need to work on a feature branch in isolation"

### Manual Invocation

```
/laravel-herd-worktree
```

## Configuration

### Default Branch

The skill defaults to `develop` as the base branch for PRs. If your project uses a different default branch (e.g., `main` or `master`), you can:

1. Let Claude detect it automatically (it will check `git symbolic-ref refs/remotes/origin/HEAD`)
2. Specify it when creating the PR

### Package Manager

The skill assumes npm. If you use yarn or pnpm, the skill will ask before running install commands.

### Composer Flags

If your project requires specific composer flags (like `--ignore-platform-reqs`), the skill will ask during setup.

## What Gets Created

```
your-project/
├── .worktrees/
│   └── feature-branch/      # Your isolated worktree
│       ├── .env             # Configured for http://feature-branch.test
│       ├── vendor/          # Fresh composer install
│       └── node_modules/    # Fresh npm install
```

Plus a Herd site at `http://feature-branch.test`

## Common Issues

### 401 Unauthorized on API routes
- Add worktree domain to `SANCTUM_STATEFUL_DOMAINS` in `.env`
- Run `php artisan config:clear`

### Cookie rejected for invalid domain
- Update `SESSION_DOMAIN` in `.env` to match worktree domain
- Add `SESSION_SECURE_COOKIE=false` for HTTP sites

### CORS Errors / White page
- Ensure `vite.config.js` has `host: 'localhost'` and `cors: true`
- Restart Vite from the worktree directory

### Mixed Content Error
- Don't secure the Herd site (use HTTP, not HTTPS)
- Keep `APP_URL` as `http://` not `https://`

### Assets not loading
- Kill all Vite processes: `pkill -f "node.*vite"`
- Remove hot file: `rm -f public/hot`
- Restart Vite from worktree

See the skill's "Common Issues" section for complete troubleshooting.

## License

MIT - see [LICENSE](LICENSE)
