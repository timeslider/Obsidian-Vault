Installing git
If winget is already installed, you can use the command
`winget install --id Git.Git -e --source winget`
in the command prompt.

Configuring git
git config --global user.name "My name"
git config --global user.email timeslider86@gmail.com
git config --core.autocrlf `[true]` and `[input]` if on MacOS or Linux

Git commands
Google "Git config"
or
`git config --help`
Space to go to the next page and Q to exit

`git config -h`
Give a shorter summary

Creating snapshots
1. Create a directory for the project
2. `mkdir Moon`
3. `cd Moon`
4. `get init`
5. `ls` to list all the files except hidden files. It should be empty
6. `ls -a` to list all files including hidden files.
7. `open .git`

Staging Area
Add modfied files to staging area. If everything is good, then we create a snapshot

git status
Gets the status of the tracked files and the staging area