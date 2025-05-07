---
layout: post
title:  "Git Basics"
date:   2024-03-26 14:22:39 +0300
author:
  name: "Yan Vugenfirer"
  url: "https://github.com/yanvugenfirer"
---

{{toc}}

# Git Basics

## Introduction

Git is a distributed version control system that helps track changes in source code during software development. This article will cover the fundamental concepts and commands that every developer should know.

## Basic Concepts

### Repository
A Git repository is a storage location where your project files and their complete history are stored.

### Commit
A commit is a snapshot of your project at a specific point in time. Each commit has a unique identifier (hash) and contains information about what changed.

### Branch
A branch is a separate line of development. The default branch is usually called `main` or `master`.


# Basic Git usage

### Initial Configuration

If this is the first time you are using Git as a specific user, you should configure it with your details first.

  - Set user credentials:

    ~~~
    git config --global user.name "Your name"
    git config --global user.email your_name@yourdomain.com
    ~~~

      - The global user credentials are saved in the `~/.gitconfig` file. You can open it and see the current status of configuration.

  - You can also configure credentials per repository, by using:

    ~~~
    git config user.name "Your name"
    git config user.email your_name@yourdomain.com
    ~~~

      - These local settings are saved in the `.git/config` file, in the local directory.
  - To use the "`git send-email`" command (which allows sending patches to a mailing list for review) your email server and account details will need to be configured in the `~/.gitconfig` file as well. Use these settings for the Daynix mail server (Obviously, replace the "`from`" and "`smtpuser`" fields with your own data):

    ~~~
    [sendemail]
            from = Your Name <your_name@yourdomain.com>
            smtpserver = smtp.gmail.com
            smtpuser = your_name@yourdomain.com
            smtpencryption = tls
            chainreplyto = false
            smtpserverport = 587
    ~~~


### Getting remote repositories, and creating new ones

#### Clone repository (get it locally)

To clone a repository into a local directory do:

~~~
git clone <repo URL> [optional local directory]
~~~

This operation sometimes called "checkout" in other source control systems.

#### Create repository

In the required directory do:

~~~
git init
~~~

The above command will create a new empty repository. You can also use "--bare" option to create repository without a file tree (can be useful if this is a central repository that no one is actively working on but only pushes to and pulling from).

Next, **assuming that we are working with GitHub** the stages are:

  - Create the remote repository on GitHub:

    ~~~
    curl -u '<USER>' https://api.github.com/user/repos -d '{"name":"<REPOSITORY>"}'
    ~~~

      - In the above command, **ONLY** `<USER>` and `<REPOSITORY>` should be changed to the appropriate values (your username on GitHub and the desired name for the new repository). The "`user/repos`" part should be kept as-is, and used literally.
      - The output of this command will print a lot of useful information on the newly created repository. It may be useful to keep.
  - Next, define the remote "origin" for the new local repository, in the newly created one on GitHub:

    ~~~
    git remote add origin https://github.com/<USER>/<REPOSITORY>.git
    ~~~

      - Again, `<USER>` and `<REPOSITORY>` should be changed here to the same values that were used above.
      - If you had a typo above, or just want to use a different remote origin, use the command below to remove the origin and start over:

        ~~~
        git remote rm origin
        ~~~


### Important commands to see the current status

  - `git status` : Shows the files that were modified since the last commit, and the files that are currently untracked by Git. Very useful to get the current status of things
  - `git log` : Shows the history of the commits, their hash numbers **(very useful)**, and their commit messages.

### Committing

1.  Before beginning to work on a specific issue, create a branch for it, and switch to it. These two actions can be achieved using a single command:

    ~~~
    git checkout -b newFeature
    ~~~


    ...Where `newFeature` is the name of the new branch you're creating. If you forgot to perform this before making your changes, you can do this later. But, it will be best to create the new branch before any changes are made.
2.  Edit the files and make the desired changes to the code.
3.  Perform "`git status`", "`git diff`" or "`git diff <file>`", in order to see the changes from the master branch (in all files or in a specific one).
4.  To stage the desired changes to a commit, use:
    1.  "`git add -i`" or "`git add -i <file>`". This will let you interactively select the wanted changes for committing.
        1.  Press "`5`" (patch) then the number of file to look at (if there are several) and then press Enter.
        2.  Blocks of changed code will be presented to you. To include the whole block press "`y`", to skip the whole block press "`n`", to split it into smaller blocks (if, *e.g.*, it is not continuous, and only one part is required) press "`s`", or, to manually edit the changes, press "`e`".
        3.  After finishing going over a file, select the next file to look at, or quit ("`7`").
        4.  If you would like to include a previously untraccked file, just type "`a`" in the prompt, and choose the number of the file.
    2.  Or "`git add -u`" to add all changes in tracked files (be careful - new files will not be added!).
    3.  Or "`git add .`" to add all changes and untracked files in local directory (be very careful, as this can add intermediate files. Should be used for initial commits).
5.  Run "`git diff --cached`" to see only the changes that will be included in the next commit.
      - In case there were mistakes in staging (running "add") you can always fix it by:

        ~~~
        git reset <file>
        ~~~


        This will unstage the file, and you will need to add it (or parts of it) again. You can also run this command without "`<file>`", to unstage all changes.
6.  Perform the commit:

    ~~~
    git commit -s
    ~~~


    In the editor that will appear, write the commit title (starting with "`PROJECT NAME:`") then, **if** the title is not informative enough, make a one-line space and underneath it write a more detailed description. Save and quit.
7.  Repeat steps 4--6 as many times as necessary, to create commits for all the introduced changes. It is a good practice to make the commits as small and specific as possible.

### Formatting patches and sending them for review

**How to determine the subject prefix of a patch:**
The subject prefix of a patch is determined by the mailing list the patch will be sent to, for example:

  - When a patch is send to a mailing list related to one repository only (like qemu-devel) subject should be prefixed with \[PATCH\]

    ~~~
    git config format.subjectprefix "PATCH"
    ~~~


<!-- end list -->

1.  To format the patches use the command:

    ~~~
    git format-patch master -n -o <desired output directory>
    ~~~


    (If this is the second (or any other number) iteration of patches after review use):

    ~~~
    git format-patch master -n -o <desired output directory> -v<iteration number>
    ~~~


    This command will format the patches (as differences from the "master" branch), and output them as ".patch" files into the specified output directory.
      - Other useful options for this command include:
          - "`--cover-letter`", which will enable you to write a cover letter explaining the patches.
          - "`--subject-prefix=<Desired-Subject-Prefix>`", which will replace the standard subject prefix from the default, which is "`[PATCH]`".
          - It is also possible to replace subject prefix for all patches generated from a specific working copy by running `"git config format.subjectprefix <Desired-Subject-Prefix>"`
          - When renaming files use --minimal or -M in order to make patch size minimal and readable.
          - Use -v option to append version number to \[PATCH\] prefix. Example -v 2 will create "\[PATCH v2\]" prefix.


#### Important note for GMail usage

https://support.google.com/accounts/answer/6010255 - users should allow the usage of "less secure apps" in order to be able to send emails with send-patch

### Editing commits

After peer review, or if you just found an error in a commit, editing the commits may be needed.
TBD

#### [Resolving merge conflict while rebasing](http://tedfelix.com/software/git-conflict-resolution.html#git-rebase)

### Pull Requests

TBD

### Force push a Pull Request

TBD

### Pushing changes to Master

After your patches were peer-reviewed and approved, and no changes are needed to be performed on the commits, you can push them to the production branch (Master) on the remote server.

**\[PLEASE VERIFY AGAIN THAT ALL THE COMMITS ARE CORRECT AND FINAL!!!\]**

**There are several ways to push changes:**

The simplest one assumes that no one was changing the remote repository while you were working on your feature, and you did not modify your local "master" branch while working on the branch dedicated to your new feature. Basically, it means that you are the only one working on the project, and you make one change at a time:

#### [[You are the only one working on the remote repository, and you KNOW it did not change while you were working on your feature]]

TBD

If other people may have changed the remote repository, but you did not change the local "master" branch since starting to work on your separate branch, use the following method:

#### [[Your LOCAL "master" branch didn't change since you started to work on your feature]]

TBD

The longest method does not assume anything. **IF IN ANY DOUBT, USE ONLY THE FOLLOWING METHOD:**

#### [[You are not sure of anything - the longest, but the most reliable way]]
TBD

## Advanced How-To's

### How to revert commit(s) in GIT

Let's say you have the branch history like:

~~~
HashA -> HashB -> HashC ... -> HashPreLast -> HashLast
~~~

Then you can use:

  - to revert `HashLast` commit either:
      - `git revert HEAD`
      - `git revert HashPreLast..HEAD`
  - to revert `HashPreLast` commit:
      - `git revert ^HEAD`
  - to revert `HashB` commit:
      - `git revert HashA..HashB`
  - to revert `HashB` to `HashLast` commits:
      - `git revert HashA..HEAD`

More information can be found [here](http://book.git-scm.com/4_undoing_in_git\_-\_reset,\_checkout_and_revert.html)

### Checking out pull request from GitHub

Sometimes you would like to checkout pull request from Github to do additional tests.
In order to do it follow: https://help.github.com/articles/checking-out-pull-requests-locally/

Fetch pull request by ID and add new branch name:

~~~
git fetch origin pull/ID/head:BRANCHNAME
~~~

Check out the branch:

~~~
 git checkout BRANCHNAME
~~~

### Bisection with GIT (**git bisect**)

Let's say you have a branch history with regression, you know commit that worked and commit that doesn't work, but there are commits between those and you do not know which one introduced a problem. For cases like that GIT provides the bisect mode (**git bisect**) that allows to find a change that introduced a bug by binary search.

**git bisect** command is described in great details at http://git-scm.com/docs/git-bisect

It worth noting that GIT bisect mode may be used for finding both regressions, i.e. commits that broke something (in a natural way) and commits that fixed something. In the latter case **good** and **bad** commits concepts have exactly opposite meaning, i.e. **good** - commit that doesn't work, **bad** - commit that works.

