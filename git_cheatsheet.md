# Git CheatSheet

## How create new branch

1. Refer to https://github.com/ideadevice/calm/blob/master/CONTRIBUTING.md for naming conventions
   > git branch new-branch-name
2. Push it to remote
   > git push -u origin local-branch-name
3. Or create and checkout to new branch using
   > git checkout -b branch-name

## To add git config

> - git config --global user.name "usernae"
> - git config --global user.email "email"
> - git config -l

## To switch to remote branch from local

> 1. git fetch origin or git fetch
> 2. git checkout -t origin/branch-name

## How to merge master changes with current branch

> 1. git fetch origin
> 2. git rebase origin/master
> 3. git push -f origin <branch>
>    > :point_up: In above commands you are actually comparing commit history and pulling all the necessary missing commits from master and putting your commits (code changes made by you) on the top.

## Update the reference

> git update-ref -d refs/remotes/origin/[locked branch name]

## Restore file

> git checkout master _path-to-file_

## Delete branch (use this command from another branch e.g.master)

1. Locally
   > git branch -d _branch_. (use -D for force delete)
   >
   > > e.g.: git branch -d mybranch
2. Remotely
   > git push _remote_ :_branch_
   >
   > > e.g git push origin :mybranch

## Unstage File

> git reset HEAD -- path/to/file

## Rename a branch

1. If you want to rename a branch while pointed to any branch, do:
   > git branch -m oldname newname
2. If you want to rename the current branch, you can do:
   > git branch -m newname
