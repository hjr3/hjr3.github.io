+++
title = "Manage Dotfiles with Jujutsu"

[taxonomies]
tags=["vcs", "jj", "git"]
+++

My dotfiles setup is very simple: I version control my `$HOME` directory using git. I ignore everything by default to avoid accidentally adding files I do not want to track. This avoids the need for any scripts or frameworks. Lately, I have been using [Jujutsu](https://github.com/martinvonz/jj) for more personal projects and decided to use `jj` to manage my dotfiles.

<!-- more -->

I am using jj version `0.21.0` at the time of this writing.

## Migration

The migration was fairly straight-forward. Before doing anything, I made sure all my changes were tracked and push to github. I also made sure I had a working backup.

For my work laptop, I chose to colocate git and jj together. This will allow me to fall back to git commands in a pinch.

Steps:

0. `cd $HOME`
1. Colocate git and jj - `jj git init --colocate` 
2. Track my branch on github - `jj branch track main@origin`

For my personal laptop, I decided to force myself to only use jj.

Steps:

0. `cd $HOME`
1. Remove the `.git` directory - `rm -fr .git`
2. Initialize Jujutsu - `jj git init`
    - This will create an empty commit based on root. We will fix this in a minute.
3. Add github as a remote - `jj git remote add origin git@github.com:hjr3/dotfiles.git`
4. Fetch the latest changes - `jj git fetch` or `jj git fetch --remote origin`
5. Track my branch on github - `jj branch track main@origin`
6. Move the empty commit - `jj rebase -d main@origin`

Moving the empty commit will change the repo from this:

```
◆  ppxsuksz herman@hermanradtke.com 2024-09-28 09:48:15 main 7962ecfe
│  Add instructions to set up jj
~  (elided revisions)
│ @  porzoswr herman@hermanradtke.com 2024-09-28 10:19:32 79747d01
├─╯  (no description set)
◆  zzzzzzzz root() 00000000
```

To this:

```
@  porzoswr herman@hermanradtke.com 2024-09-28 10:22:39 d396a016
│  (empty) (no description set)
◆  ppxsuksz herman@hermanradtke.com 2024-09-28 09:48:15 main 7962ecfe
│  Add instructions to set up jj
~
```
  
## No More Forcibly Adding Changes

My workflow did slightly change. My `.gitignore` file used to be:

```
# Ignore everything
*
```

and I would use forcibly add changes using`git add -f <pathspec>...`. There is no concept of `git add` in `jj` though.

Instead, I explicitly added my files to `.gitignore`. I ran `git ls-tree -r main --name-only` to find all files being tracked by git in my `main` branch. Then I used the `!` operator in `.gitignore` to opt them in. Now, my `.gitignore` file looks something like this:

```
# Ignore everything
*

# Manually add files we want to track
!.gitignore
!.ssh/config
```

These changes are compatible with git and I can easily revert back to my old workflow if I decided to stop using jj.

## Docs

I primarily used two sources of documentation to figure this all out:

- The excellent [Jujutsu Tutorial](https://steveklabnik.github.io/jujutsu-tutorial/) by Steve Klabnik
- The Jujutsu Git Comparison [Command equivalance table](https://martinvonz.github.io/jj/latest/git-comparison/#command-equivalence-table)
