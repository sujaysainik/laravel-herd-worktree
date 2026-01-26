---
name: laravel-herd-worktree
description: Use when setting up a Laravel worktree for local development with Laravel Herd, or when user asks to work on a feature branch in isolation
license: MIT
---

# Laravel Herd Worktree Setup

## Overview

Sets up a git worktree for Laravel projects served by Laravel Herd, ensuring the worktree has its own site URL and properly configured environment.

**Announce at start:** "I'm using the laravel-herd-worktree skill to set up an isolated Laravel workspace with Herd."

## When to Use

- User wants to work on a feature branch in isolation
- User mentions "worktree" and the project uses Laravel Herd
- Starting work on a task that needs isolation from main branch

## Prerequisites

- **Laravel Herd** installed and running on macOS
- **Git** for version control
- **Vite** as the frontend build tool (this skill assumes Vite, not Webpack/Mix)
- **npm** as package manager (adjust commands if using yarn/pnpm)
- **Laravel Sanctum** if using API authentication (optional - skill handles this if present)

## Setup Steps

### 1. Create Worktree (if needed)

```bash
# Check if worktree exists
git worktree list | grep "$BRANCH_NAME"

# If not, create it
git worktree add .worktrees/$BRANCH_NAME -b $BRANCH_NAME
```

### 2. Link with Laravel Herd

```bash
cd /path/to/project/.worktrees/$BRANCH_NAME
herd link $BRANCH_NAME
```

This creates a site at `http://$BRANCH_NAME.test`

**Note:** Do NOT run `herd secure` - the site should use HTTP to match the Vite dev server.

### 3. Copy and Configure .env

```bash
# Copy .env from main project
cp /path/to/main/project/.env /path/to/worktree/.env

# Update APP_URL to match Herd site (use HTTP, not HTTPS)
sed -i '' "s|APP_URL=.*|APP_URL=http://$BRANCH_NAME.test|" /path/to/worktree/.env

# Update SESSION_DOMAIN to match the worktree domain
sed -i '' "s|SESSION_DOMAIN=.*|SESSION_DOMAIN=$BRANCH_NAME.test|" /path/to/worktree/.env

# Add worktree domain to SANCTUM_STATEFUL_DOMAINS (for API auth, if using Sanctum)
sed -i '' "s|SANCTUM_STATEFUL_DOMAINS=\(.*\)|SANCTUM_STATEFUL_DOMAINS=\1,$BRANCH_NAME.test|" /path/to/worktree/.env

# Ensure secure cookies are disabled for HTTP
echo "SESSION_SECURE_COOKIE=false" >> /path/to/worktree/.env
```

### 4. Install Dependencies

**CRITICAL: Worktrees do NOT share vendor/ or node_modules/ with the main project.**

**Before running composer install, ask the user:**

```
question: "Do you need to ignore any platform requirements for composer?"
header: "Composer"
options:
  - label: "No flags needed (Recommended)"
    description: "Run composer install without platform requirement flags"
  - label: "Ignore all platform reqs"
    description: "Use --ignore-platform-reqs to skip all platform checks"
  - label: "Custom flags"
    description: "I'll provide specific flags to ignore"
```

```bash
# Install PHP dependencies (adjust flags based on user response)
composer install --no-interaction  # Add --ignore-platform-reqs if user selected it

# Install Node.js dependencies
npm install

# Clear Laravel caches
php artisan config:clear
php artisan cache:clear
```

**Ask user if composer/npm install fails:** "The dependency installation failed. Would you like me to:
1. Retry with `--ignore-platform-reqs`
2. Skip and continue (not recommended)
3. Show the error for manual debugging"

### 5. Ensure vite.config.js Has CORS Support

**IMPORTANT:** To avoid Cross-Origin Request Blocked errors, ensure vite.config.js includes:

```javascript
export default defineConfig(() => {
    return {
        server: {
            host: 'localhost',
            cors: true,
        },
        plugins: [
            // ... your plugins
        ]
    }
});
```

**Key settings:**
- `host: 'localhost'` - Prevents CORS issues (do NOT use `0.0.0.0`)
- `cors: true` - Enables CORS for cross-origin requests

### 6. Kill Existing Vite Processes & Start Dev Server

**IMPORTANT:** If Vite is already running from the main project, it will occupy port 5173. You must kill it first to ensure the worktree's Vite uses the correct port and serves the correct assets.

```bash
# Kill any existing Vite processes
pkill -f "node.*vite" 2>/dev/null

# Remove stale hot file
rm -f public/hot

# Start Vite dev server (will use APP_URL from .env)
npm run dev
```

**Why this matters:** Vite reads APP_URL from .env to configure hot module replacement. If the wrong Vite instance is running, assets will fail to load or point to the wrong domain.

## Finishing Work (Integrating Back to Main Tree)

When development is complete and you want to integrate the worktree changes:

### 0. Gather Information

**Before proceeding, use AskUserQuestion to ask:**

**Question 1 - Task Identifier:**
```
question: "Do you have a task/issue number for this work?"
header: "Task ID"
options:
  - label: "No task number"
    description: "Skip adding a task identifier to the branch and commit"
  - label: "Enter task number"
    description: "I'll provide a task/issue number to include in the branch name and commit"
```

**Question 2 - PR Description:**
```
question: "How would you like to handle the PR description?"
header: "PR Body"
options:
  - label: "I'll write it"
    description: "Create PR with empty body - I'll fill in the description on GitHub"
  - label: "Generate for me"
    description: "Analyze the changes and generate a PR description automatically"
  - label: "Leave empty"
    description: "Create PR with no description"
```

### 1. Commit All Changes

```bash
cd /path/to/project/.worktrees/$BRANCH_NAME
git add -A
git commit -m "Your commit message (#TASK_NUMBER)"  # Omit (#TASK_NUMBER) if none provided
```

### 2. Rename Branch with Task Number (if provided)

If a task number was provided and the branch doesn't include it:

```bash
git branch -m old-branch-name TASK_NUMBER-descriptive-name
```

### 3. Push and Create PR

**Note:** The base branch defaults to `develop`. Adjust to your project's default branch (e.g., `main` or `master`) if different. You can detect the default branch with:
```bash
git symbolic-ref refs/remotes/origin/HEAD | sed 's@^refs/remotes/origin/@@'
```

```bash
git push -u origin BRANCH_NAME

# Based on user's PR description preference:
# - "I'll write it" or "Leave empty": gh pr create --base $DEFAULT_BRANCH --title "Title" --body ""
# - "Generate for me": Generate description from git diff, then use --body "$DESCRIPTION"
gh pr create --base $DEFAULT_BRANCH --title "Description (#TASK_NUMBER)" --body "$BODY"
```

### 4. Cleanup After Merge

After the PR is merged, clean up the worktree:

```bash
# Stop Vite if running
pkill -f "node.*vite" 2>/dev/null

# From main project directory
cd /path/to/main/project
herd unlink $BRANCH_NAME
git worktree remove .worktrees/$BRANCH_NAME
git branch -d TASK_NUMBER-branch-name  # Delete local branch
```

## Cleanup (When Done)

When abandoning a worktree without merging:

```bash
# Stop Vite if running
pkill -f "node.*vite" 2>/dev/null

# Unlink from Herd
herd unlink $BRANCH_NAME

# Remove worktree
git worktree remove .worktrees/$BRANCH_NAME

# Delete branch if not needed
git branch -D $BRANCH_NAME
```

## Quick Reference

| Step | Command |
|------|---------|
| Create worktree | `git worktree add .worktrees/NAME -b NAME` |
| Link to Herd | `herd link NAME` |
| Copy .env | `cp ../../../.env .env` |
| Update APP_URL | `sed -i '' "s\|APP_URL=.*\|APP_URL=http://NAME.test\|" .env` |
| Install PHP deps | `composer install --no-interaction` |
| Install Node deps | `npm install` |
| Clear cache | `php artisan config:clear && php artisan cache:clear` |
| Kill old Vite | `pkill -f "node.*vite"` |
| Start dev | `npm run dev` |
| Unlink | `herd unlink NAME` |
| Remove worktree | `git worktree remove .worktrees/NAME` |

## Common Issues

### 401 Unauthorized on API routes
- **Cause:** SANCTUM_STATEFUL_DOMAINS doesn't include the worktree domain
- **Symptoms:** Login works but API calls return 401
- **Fix:**
  1. Add worktree domain to SANCTUM_STATEFUL_DOMAINS in .env
  2. Run `php artisan config:clear`

### Cookie rejected for invalid domain
- **Cause:** SESSION_DOMAIN in .env doesn't match the worktree site domain
- **Symptoms:** Console shows "Cookie has been rejected for invalid domain"
- **Fix:**
  1. Update SESSION_DOMAIN in .env: `SESSION_DOMAIN=$BRANCH_NAME.test`
  2. Add `SESSION_SECURE_COOKIE=false` for HTTP sites
  3. Run `php artisan config:clear`
  4. Clear browser cookies for the domain (or use incognito)

### White page / CORS Errors (Cross-Origin Request Blocked)
- **Cause:** Vite using `host: '0.0.0.0'` instead of `localhost`
- **Symptoms:** Console shows "Cross-Origin Request Blocked" for Vite assets
- **Fix:**
  1. Update vite.config.js to use `host: 'localhost'` and `cors: true`
  2. Restart Vite: `npm run dev`

### White page / Mixed Content Error
- **Cause:** HTTPS site trying to load HTTP Vite assets
- **Symptoms:** Console shows "Mixed Content" errors
- **Fix:**
  1. Unsecure the site: `herd unsecure $BRANCH_NAME`
  2. Update APP_URL to use `http://` instead of `https://`
  3. Run `php artisan config:clear`
  4. Restart Vite

### White page / Assets not loading
- **Cause:** Vite running from wrong directory or wrong port
- **Fix:**
  1. Kill all Vite: `pkill -f "node.*vite"`
  2. Remove hot file: `rm -f public/hot`
  3. Restart from worktree: `npm run dev`

### Vite uses wrong URL
- **Cause:** APP_URL in .env not updated
- **Fix:** Update APP_URL to match Herd site name

### Missing vendor directory
- **Cause:** Worktrees don't share vendor/
- **Fix:** Run `composer install` in worktree

### Missing node_modules
- **Cause:** Worktrees don't share node_modules/
- **Fix:** Run `npm install` in worktree

### Neo4j/DB connection errors
- **Cause:** Missing .env file
- **Fix:** Copy .env from main project

### Port already in use
- **Cause:** Another Vite instance running
- **Fix:** Kill existing Vite processes: `pkill -f "node.*vite"`

### Permission issues with Herd
- **Cause:** Herd needs to access worktree
- **Fix:** Ensure worktree is in a Herd-accessible path

## Checklist

Before considering setup complete, verify:

- [ ] Worktree created at `.worktrees/$BRANCH_NAME`
- [ ] Herd link created (`herd link $BRANCH_NAME`)
- [ ] Site is NOT secured (use HTTP, not HTTPS)
- [ ] vite.config.js has `host: 'localhost'` and `cors: true`
- [ ] .env copied and APP_URL updated (use `http://`)
- [ ] SESSION_DOMAIN updated to match worktree domain
- [ ] SESSION_SECURE_COOKIE=false added for HTTP
- [ ] `composer install` completed successfully
- [ ] `npm install` completed successfully
- [ ] Old Vite processes killed
- [ ] `npm run dev` running from worktree directory
- [ ] Site accessible at `http://$BRANCH_NAME.test` (no white page)

## Integration

**Pairs with:**
- Git worktree workflows
- Any Laravel development workflow
