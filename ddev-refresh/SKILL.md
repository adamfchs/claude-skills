---
name: ddev-refresh
description: Refresh a local DDEV Drupal install: pull latest code, optionally sync a remote database, run updates, and restore your working branch.
---

# DDEV Local Refresh

Interactive workflow for refreshing a local DDEV Drupal environment to match the latest codebase and optionally a remote database.

## When This Skill Activates

- User invokes `/ddev-refresh`
- User says "refresh local", "refresh my local", "refresh the local"
- User says "sync local", "sync my local", "sync the local"
- User says "pull latest", "pull latest and update", "update local"
- User says "let's do a refresh", "time to refresh", "need to refresh"
- User says "grab a new database", "get the latest database", "pull the database"
- User says "get in sync", "get up to date", "get current"

---

## Pre-flight: Read project context

Before starting, check the following files if they exist in the project root:
- `CLAUDE.md` — look for DDEV project name, Pantheon/Acquia site name, drush alias format
- `EC-INSTALL.md` — look for drush alias examples (e.g. `@pantheon.{project}.{dev|test|live}`)

You need to know:
1. The DDEV project name (e.g. `csbweb`)
2. The hosting platform (Pantheon, Acquia, other)
3. The drush alias format for remote DB dumps

---

## Step 1 — Note current branch, switch to main

```bash
git branch --show-current
```

Record the current branch name. If it is not `main`, note it — you will ask about switching back at the end.

```bash
git checkout main
```

---

## Step 2 — Pull latest code

```bash
git pull
```

If the pull fails (merge conflicts, diverged history), stop and report to the user. Do not proceed until resolved.

---

## Step 3 — Install Composer dependencies

```bash
ddev composer install
```

If there are any errors, stop and debug before continuing. Do not proceed to Step 4 until Composer reports a clean install.

---

## Step 4 — Optionally sync remote database

Ask the user: **"Do you want to sync a database from a remote environment?"**

If **no**: skip to Step 7.

If **yes**: ask **"Which environment — dev, test, or live?"**

### Determine the drush alias

Use the project context gathered in the pre-flight step. For Pantheon projects the alias format is:
```
@pantheon.{project-name}.{env}
```
For example: `@pantheon.csbweb.live`

If you cannot determine the alias format from project files, ask the user to confirm it before continuing.

### Remove any existing database.sql

```bash
rm -f database.sql
```

### Dump the remote database

```bash
ddev drush @pantheon.{project}.{env} sql-dump > database.sql
```

(Substitute the correct alias.) Confirm the file was created and is non-empty before proceeding.

---

## Step 5 — Import the database

```bash
ddev import-db --file=database.sql
```

---

## Step 6 — Handle import errors

If there are any import errors, stop and debug before proceeding to Step 7. Do not run cache clear or updb against a broken database state. Common issues:
- File was empty or truncated (check remote alias / SSH auth)
- `ddev auth ssh` may need to be run if SSH agent keys are missing

---

## Step 7 — Clear cache

Run this regardless of whether a database was imported.

```bash
ddev drush cr
```

---

## Step 8 — Run database updates

```bash
ddev drush updb -y
```

Review and report any output indicating failed updates.

---

## Step 9 — Generate a login link and open it in Chrome

```bash
ddev drush uli
```

Capture the URL and open it in Chrome:

```bash
open -a "Google Chrome" "<url>"
```

Also present the URL to the user in case they need it.

---

## Step 10 — Clean up database file

If a database was downloaded in Step 4, delete it now:

```bash
rm -f database.sql
```

---

## Step 11 — Offer to return to original branch

If the user started on a branch other than `main`, ask:
**"You started on branch `{branch-name}`. Do you want to switch back to it?"**

If yes:
```bash
git checkout {branch-name}
```

---

## Step 12 — Offer to merge main into working branch

After switching back to the original branch, ask:
**"Do you want me to merge main into `{branch-name}`?"**

If yes:
```bash
git merge main
```

Report any merge conflicts and stop for the user to resolve them.

---

## Related Skills

- @drupal-ddev - General DDEV patterns and commands
- @drupal-contrib-mgmt - Composer and module update workflows
