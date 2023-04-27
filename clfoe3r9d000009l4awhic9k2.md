---
title: "Creating Supabase Backups with GitHub workflows"
datePublished: Sat Mar 25 2023 19:54:19 GMT+0000 (Coordinated Universal Time)
cuid: clfoe3r9d000009l4awhic9k2
slug: creating-supabase-backups-with-github-workflows
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1680716023276/3843943e-3625-4933-abb3-77cb76504be3.avif
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1679775794963/061dfaa5-cf30-491e-9a7a-72fdb5fc7a1f.png
tags: github, github-actions-1, supabase

---

This tutorial explains the [Supa-backup](https://github.com/mansueli/Supa-Backup) workflow that automates the process of backing up a Supabase database on a daily basis. This workflow uses GitHub Actions, which is a powerful automation tool that allows developers to build, test, and deploy their applications directly from their GitHub repositories. The Supa-backup workflow creates a daily backup of the Supabase database and commits the backup file to the repository.

## **Workflow Overview**

The Supa-backup workflow has the following steps:

1. **Trigger the Workflow**: The workflow runs automatically every day at midnight using the `schedule` event. It can also be triggered manually from the GitHub Actions tab using the `workflow_dispatch` event.
    
2. **Check out the Repository**: The `actions/checkout@v3` action checks out the repository, so the workflow can access the repository's files and history.
    
3. **Install Postgres 15**: The `Postgres15` job installs PostgreSQL 15 on the Ubuntu virtual machine using the `apt` package manager. It then uses the `pg_dump` command to create a backup of the Supabase database, excluding specific schemas.
    
4. **Tweak the Dump File**: The `Tweaking the dump file` job uses the `sed` command to comment out some of the SQL commands in the dump file that might cause errors when restoring the database. This is necessary because some schemas are excluded from the dump file in the previous step.
    
5. **Commit the Backup File**: The `git-auto-commit-action@v4` action commits the backup file to the repository with a commit message of `Supabase backup`. The action uses the default `GITHUB_TOKEN` to commit the file, so there is no need to create a personal access token.
    

## **Workflow Code**

Here is the full code for the Supa-backup workflow:

```yaml
mdxCopy codename: Supa-backup

# Controls when the workflow will run
on:
  # Allows you to run this workflow manually from the Actions tab
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:
  schedule:
    - cron: '0 0 * * *' # Runs every day at midnight
    
# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  run_db_backup:
    runs-on: ubuntu-latest
    permissions:
      # Give the default GITHUB_TOKEN write permission to commit and push the changed files back to the repository.
      contents: write
    steps:
      - uses: actions/checkout@v3
        with:
          ref: ${{ github.head_ref }}
      - name: Postgres15 
        run: |
          sudo apt-get remove postgresql-client-common
          sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
          wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc &>/dev/null
          sudo apt update
          sudo apt install postgresql-15 postgresql-client-15 -y
          /usr/lib/postgresql/15/bin/pg_dump --clean --if-exists --quote-all-identifiers --schema '*' --exclude-schema 'extensions|graphql|graphql_public|net|pgbouncer|pgsodium|pgsodium_masks|realtime|supabase_functions|storage|pg_*|information_schema' -d postgres://postgres:${{ secrets.SUPABASE_PASSWORD }}@${{ secrets.SUPABASE
```