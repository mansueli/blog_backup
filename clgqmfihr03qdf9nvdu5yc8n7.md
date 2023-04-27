---
title: "How to Use GitHub Actions for Backing Up Supabase Storage Objects"
datePublished: Fri Apr 21 2023 14:02:39 GMT+0000 (Coordinated Universal Time)
cuid: clgqmfihr03qdf9nvdu5yc8n7
slug: how-to-use-github-actions-for-backing-up-supabase-storage-objects
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1682016523953/fd6353ff-d10c-4e81-93e4-4654052a3371.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1682016586132/39a5e964-e3a9-4584-a97c-5a4363665e8f.png
tags: storage, backup, github-actions, ci-cd, supabase

---

GitHub Actions is a feature that allows you to automate your software development workflows right in your GitHub repository. You can create custom workflows that perform any job you'd like, such as building, testing, deploying, or packaging your code. In this blog post, I will show you how to use GitHub Actions to back up your storage objects from Supabase, a platform that provides you with a scalable and secure backend for your web applications.

Supabase storage is a service that lets you store and serve files such as images, videos, or documents. You can create buckets to organize your files and set permissions to control who can access them. However, you may also want to back up your storage objects regularly in case of data loss or corruption. This is where GitHub Actions can help you.

GitHub Actions allows you to run workflows on specific events, such as push, pull requests, schedules, or manual triggers. You can use actions from the GitHub Marketplace or create your own actions using scripts or Docker containers. In this blog post, I will use a Python script that uses the Supabase Python client to download all the files from a Supabase storage bucket and save them in a local directory.

The workflow that I will create has the following steps:

* Checkout the code from the GitHub repository
    
* Set up Python and install the Supabase package
    
* Run the backup script to download the files from Supabase storage
    
* Save the backup as an artifact or commit it to the main branch of the repository
    

In this blog post, I will show you how to use GitHub Actions to back up your storage objects from Supabase to your GitHub repository. There are two options for doing this: backing up into an artifact or backing up into the main repo (which should be private).

The workflow file is named `supastorage-backup.yml` and is located in the `.github/workflows` directory of the repository. The file contains the following code:

```yaml
name: SupaStorage-backup
on:
  # Allows you to run this workflow manually from the Actions tab
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:
  schedule:
  # Runs twice a day. Once at 8:00, then at 20:00.
    - cron: '0 8,20 * * *' 
jobs:
  backup:
    runs-on: ubuntu-latest
    env:
      SUPABASE_URL: https://jpzwvyukydqlzwgvncyj.supabase.co
      SUPABASE_SERVICE_ROLE: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6Impwend2eXVreWRxbHp3Z3ZuY3lqIiwicm9sZSI6InNlcnZpY2Vfcm9sZSIsImlhdCI6MTY3MDg4MTY2MSwiZXhwIjoxOTg2NDU3NjYxfQ.oP0gYfwdWCWQObNC3kxq7yK-LbYOFhkXqQWvJ3-aC1I
    permissions:
      contents: write
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
      with:
         ref: ${{ github.head_ref }}
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.10'

    - name: Install dependencies and perform backup
      run: |
        pip install supabase
        [[ -d supabase_storage_backup ]] || mkdir supabase_storage_backup
        cd supabase_storage_backup
        wget https://raw.githubusercontent.com/mansueli/Supa-Migrate/main/storage-backup.py
        chmod +x storage-backup.py
        python storage-backup.py
        rm storage-backup.py
      shell: bash
      
    - name: Set current date as env variable
      run: echo "NOW=$(date +'%Y-%m-%dT%H:%M:%S')" >> $GITHUB_ENV
# Option A: Storing into the repo (should be private)        
    - uses: stefanzweifel/git-auto-commit-action@v4
      with:
        commit_message: Supabase Storage backup - ${{ env.NOW }}
# Option B: Backup into a GitHub Artifact
    - name: Save backup as artifact
      uses: actions/upload-artifact@v3
      with:
          name: Storage-Backup
          path: ./supabase_storage_backup # <= upload folder
          retention-days: 14
```

### Let's go through each part of the code and explain what it does.

The first part defines the name of the workflow and the triggers for running it. In this case, the workflow will run on push and pull request events to the main branch, on manual triggers from the Actions tab, and on a schedule of twice a day at 8:00 and 20:00.

```yaml
  workflow_dispatch:
  schedule:
  # Runs twice a day. Once at 8:00, then at 20:00.
    - cron: '0 8,20 * * *' 
```

The second part defines the job that will perform the backup. The job runs on an Ubuntu virtual machine and sets two environment variables that contain the Supabase URL and service role key. You need to store these secrets in your repository settings and access them using the syntax `${{ secrets.SECRET_NAME }}`. The job also grants written permission to access the contents of the repository.

## Installing the dependencies and running the backup

First, this part of the script installs the supabase package using pip, which is a package manager for Python. Then, the script checks whether the supabase\_storage\_backup directory exists, and if not, creates it using the `mkdir` command. This directory will be used to store the downloaded files.

The script then changes its working directory to the newly created `supabase_storage_backup` directory using the `cd` command.

After that, the script uses the `wget` command to download the [`storage-backup.py`](http://storage-backup.py) file from the `mansueli/Supa-Migrate` repository on GitHub. This file contains the Python code that is responsible for downloading all the files from the Supabase storage bucket.

The script then runs the [`storage-backup.py`](http://storage-backup.py) script using the `python` command, which downloads all the files from the storage bucket and saves them in the `supabase_storage_backup` directory.

Finally, the `rm` command is used to remove the [`storage-backup.py`](http://storage-backup.py) file from the `supabase_storage_backup` directory.

```yaml
    - name: Install dependencies and perform backup
      run: |
        pip install supabase
        [[ -d supabase_storage_backup ]] || mkdir supabase_storage_backup
        cd supabase_storage_backup
        wget https://raw.githubusercontent.com/mansueli/Supa-Migrate/main/storage-backup.py
        chmod +x storage-backup.py
        python storage-backup.py
        rm storage-backup.py
      shell: bash
```

## Backup into an artifact:

Backing up into an artifact means that you will create a zip file containing all your storage objects and upload it as an artifact to your GitHub workflow. An artifact is a file or collection of files produced by a workflow run that can be downloaded after the run completes. Artifacts are not part of your repository and are only stored for a limited time (by default 90 days).

```yaml
    - name: Save backup as artifact
      uses: actions/upload-artifact@v3
      with:
          name: Storage-Backup
          path: ./supabase_storage_backup # <= upload folder
          retention-days: 14
```

## Backup into the repo:

Backing up into the main repo means that you will copy all your storage objects to a folder in your repository and commit them to the main branch. This way, you will have a version history of your storage objects and you can access them anytime from your repository. However, this option requires that your repository is private, otherwise, anyone can see and download your storage objects.

```yaml
    - uses: stefanzweifel/git-auto-commit-action@v4
      with:
        commit_message: Supabase Storage backup - ${{ env.NOW }}
```

Using a Python script that utilizes the Supabase Python client, we have created a workflow that can download all the files from a Supabase storage bucket and save them in a local directory. The workflow can run on specific events, such as push, pull requests, schedules, or manual triggers.

We have discussed two options for backing up your storage objects from Supabase to your GitHub repository. The first option is to back up into an artifact, which involves creating a zip file containing all your storage objects and uploading it as an artifact to your GitHub workflow. The second option is to back up into the main repo, which involves copying all your storage objects to a folder in your repository and committing them to the main branch.

By following the steps outlined in this blog post, you can use GitHub Actions to automate your backups of Supabase storage objects, ensuring that your data is safe and secure.