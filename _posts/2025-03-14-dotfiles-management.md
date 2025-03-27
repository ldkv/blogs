---
layout: post
title: Dotfiles management with Copier
date: 2025-03-14 01:04 +0100
---

## Introduction

Dotfiles are the configuration files to customize the shell and other command-line tools. They are usually hidden in the user's home directory, and they are called "dotfiles" because their filenames start with a dot (`.`).

Dotfiles are present on every Unix-like system, the most popular ones are `.bashrc` or `.zshrc` (for zsh shell), who are loaded when the shell starts. They can contain aliases, functions, environment variables, and other settings that customize the shell environment to the user's preferences.

On Windows they also exist, for example the profile file for `PowerShell`, which is located in the `$PROFILE` environment variable.

As someone who works on multiple machines and operating systems, I find it important to have a consistent environment across all of them, especially when I found and tried random configurations regularly. After some time it's hard to keep track of all the changes and customizations, and even harder to replicate them across machines.

This is why I started to look for a solution to manage my dotfiles. This is not a new problem, and there are many tools and methods readily available, and I will discuss some of them in this post.

But first, let's show my current context and what I want to achieve with a dotfiles management solution.

## Context

Currently I find myself dabbling with multiple machines on a daily basis, each one with a different OS and shell environment.

- My personal PC runs `Windows 11`, with `PowerShell v7` as the default shell.
- I also use `Debian` via `WSL2` on my personal PC, with `zsh` as the default shell.
- I have a mini PC running `Ubuntu`, with `zsh` as the default shell.
- My work laptop runs `macOS`, with `zsh` as the default shell.
- I also work with multiple `Ubuntu` servers, with `bash` as the default shell. Normally I am not allowed to customize these servers, but I include them anyway for the complete picture.

As you can see, the situation is quite complex, and it is hard to switch contexts without slowing down my workflow. A consistent environment across all machines won't solve all the problems, but it will make the transition between them much smoother.

The ideal solution I am looking for should have the following features:

- Backup my current dotfiles to a central repository with version control.
- Install the dotfiles to any machine with minimal effort.
- Support for custom scripts and configurations during the installation process.
- Support for different operating systems and shells, including `Windows` and `PowerShell`.
- Allow to customize profiles on each machine (personal, work, etc.).
- Allow to sync updates across all machines without erasing local customizations.
- Should be able to sync outside of the `$HOME` directory.

## Existing solutions and their limitations

There are many tools and methods available to manage dotfiles, where the most common approaches can be summarized into the following categories:

- [Git bare repository](https://www.atlassian.com/git/tutorials/dotfiles)
- [Symlink](https://en.wikipedia.org/wiki/Symbolic_link): for example [GNU Stow](https://www.gnu.org/software/stow/)
- Template and overwrite target files: for example [Chezmoi](https://www.chezmoi.io/)

These solutions are elegant and should work well for most users. I have read and tried some of them, and almost settled with `chezmoi` for its flexible features.

However, they all share 2 common limitations that doesn't satisfy my requirements:

1. **Unable to manage files outside of the `$HOME` directory**. This is related to my `PowerShell` configuration, where `$PROFILE` is located in `D:\`, not in `$HOME` (`C:\Users\username\`).
2. **Unable to sync updates without erasing local customizations**. This one is quite specific, and kinda defeats the purpose of a homogeneous environment. But let's consider the following scenario: I use Ruby on a specific machine to manage my blogs (this one), so I need to extend my `$PATH` for the Ruby binaries in `~/.zshrc`:

```shell
export PATH="$HOME/gems/bin:$PATH"
```

I don't need this line anywhere else, nor do I want to add it to my dotfiles permanently. However, any current solution will erase this line when syncing updates, which is also not I want. What I want is similar to the `git cherry-pick` feature, where changes are applied selectively, and not just blindly overwritten.

I know my life would be much easier if I just add this line to the central repository and forget about it, but I was stubborn and took it as a challenge to make it work, so here we are.

## Solution using Copier

### What is Copier?

I came across [Copier](https://copier.readthedocs.io/en/stable/) at work, where I use it to create templates to synchronize projects structure across my organization's repositories. It offers a lot of flexibility and customization using templates and task execution, which is very similar to `chezmoi`.

However, one of its most powerful features is to update and synchronize the project to the latest template version, using a [sophisticated workflow](https://copier.readthedocs.io/en/stable/updating/#how-the-update-works) based on `git diff` to **apply the changes selectively**.

This was the missing piece of the puzzle I was looking for when I was using `chezmoi`, where the only way to sync updates was to overwrite local changes.

### Manage dotfiles with Copier

The main idea is a combination of existing solutions, with multiple components:

- A **central repository** to store the dotfiles as template. It is hosted here on my [dotfiles repository](https://github.com/ldkv/dotfiles).
- A **local repository** to render the dotfiles on the target machine from the central template.
- Python scripts to backup then **symlink** the local files to the correct location. The script is executed automatically when the local repository is updated.

A simplified routines for initial setup and updating the dotfiles would look like this:

![routines](/assets/img/posts/2025-03-14-dotfiles-management/dotfiles-routines.png)

### Prerequisites

Copier is not the only required component for this solution, it is combined with another powerful tool, [`uv`](https://docs.astral.sh/uv/). The advantages of using `uv` are numerous:

- It can install Copier independently as a CLI tool.
- It is cross-platform
- It does not require an existing Python installation.

The 2nd point is particularly important. It also allows me to write Python scripts to automate the dotfiles management, instead of writing separate shell scripts for each platform, which is quite a pain (looking at you, [`PowerShell`](https://github.com/ldkv/dotfiles/blob/6caf104d0ef9f9826df058211ed45c6fdcaa4ddd/template/%7B%25if%20_copier_conf.os%20%3D%3D%20'windows'%25%7Dwindows%7B%25endif%25%7D/initial_setup.ps1.j2)). The power of `uv` is not limited to this, but I will not cover it in this post, maybe in a future post dedicated to it.

To install `uv`:

```shell
curl -LsSf https://astral.sh/uv/install.sh | sh
```

For other platforms, please refer to the [official documentation](https://docs.astral.sh/uv/getting-started/installation/).

To install Copier as a CLI tool:

```shell
uv tool install copier
```

This allows me to launch Copier directly with `copier` command, or through uv with `uvx copier` (equivalent to `uv tool copier`).

### Initial setup

The initial setup is straightforward, just clone the central repository using the `copier` command:

```shell
uvx copier copy gh:ldkv/dotfiles.git /path/to/local/repository --trust
```

You will be prompted to answer some questions, such as the user name and email, or any other questions defined in the `copier.yml` file. The template will be then rendered to the folder based on the configurations, and a oneshot script will be executed to install anything I define, such as `oh-my-zsh` and its plugins. The `--trust` flag is required to allow the script to be executed.

The script will also initialize a git repository in the local folder, which is required by `Copier` for its update workflow.

### Update routine

For the update routine to work normally, there are 2 requirements:

- Generate a new tag in Github for each release. This is also a good practice to have a history of changes.
- The local repository doesn't have uncommited changes. This should not be a problem since all local changes are committed automatically by the script.

To execute the update routine:

```shell
uvx copier update /path/to/local/repository -A --trust
```

This will apply the changes selectively to the local repository, you will be able to review and resolve any conflicts. Finally, an update script will be executed to do the following:

1. Detect all managed dotfiles based on local folder structure. The script find the managed dotfiles from 2 sources in the local repository:
   - All files in the `common` folder.
   - All files in `unix` or `windows` folder, based on the OS. If we are on Windows, the `unix` folder won't be rendered, and vice versa.
2. Backup all existing dotfiles to a `backups` subfolder. This allows to restore the previous dotfiles if there were any problems during the update.
3. Create symlinks to the new dotfiles.
4. Commit all local changes to the repository, including the backups.

### Customizations

At this point you will wonder how does this solve my 1st requirement mentioned earlier, about managing files outside of the `$HOME` directory?

In fact, the script takes into account a mappings defined in the `configs.json` file, where I can specify a custom target to any path outside of the `$HOME` directory for a given dotfile.

If no mapping is defined, the dotfile will be linked to the `$HOME` folder by default, based on its relative path in the local repository. For example, a file located in `common/.config/.abc` will be linked to `$HOME/.config/.abc`.

The Python script can manipulate files using native `Pathlib` package, but it can also execute any command using `subprocess.run`. It also allows me to adapt the script to the target platform easily, should the need arise.

With this approach, the possibilities for customizations are endless. For example, in the same configs JSON, I also define a list of external plugins for `oh-my-zsh` to install during the initial setup, but only triggered on Unix systems.

## Final thoughts

This concludes my dotfiles management solution. It is not perfect, but it is a good compromise between flexibility and ease of use. Above all, it fits all my needs, and I enjoyed tackling the challenge with multiple iterations, learning a lot during the process.

I am sure there are many improvements that can be made, such as automating the updates whenever a new version is released, or script to revert the changes using the backups, etc. I will probably work on it in the future, and I will share the progress here.

Finally, as this post is dedicated to the management solution, I didn't go into details about my dotfiles preferences, and I will save that for another post.
