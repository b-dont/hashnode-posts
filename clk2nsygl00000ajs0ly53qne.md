---
title: "Shell, Yes!"
datePublished: Fri Jul 14 2023 14:13:27 GMT+0000 (Coordinated Universal Time)
cuid: clk2nsygl00000ajs0ly53qne
slug: shell-yes
canonical: https://brandont.dev/blog/shell-yes/
tags: linux, bash, zsh, shell

---

How to write shell configs that work for you, at any experience level.

For any Linux enthusiast, administrator, developer, systems engineer, or overall neck-beard, the terminal is the meat and potatoes of a good chunk of day-to-day work and interaction with Linux. Your terminal's shell is a powerful tool to make your work easier and more fun to work with.

Configuring your shell to work for your needs is a useful skill to have and is at the heart of automation in your terminal environment. We computer nerds are lazy and love tweaking computer environments to do work for us.

## Anatomy

I'm going to break down shell configurations into several different files (more details in [this awesome Unix StackExchange answer](https://unix.stackexchange.com/a/71258)):

* `rc` - controls our shell's interactive behavior.
    
* `env` - contains exported environment variables.
    
* `aliases` - contains—you guessed it—our aliases.
    

There are a few other files, but I'll mention them later.

I use `zsh` as my shell, so this post will focus on it. The principles should stay the same for your shell of choice.

### The `rc` file

An `rc` file (or a [`RUNCOM` file](https://www.baeldung.com/linux/rc-files)) is a file or directory designated to hold configurations for an interactive shell. Modern shells usually come preinstalled with an `rc` file with several example configuration lines and inline comments explaining them, and these example files can vary between Linux distributions.

This is the best place to start your configurations, and there are plenty of experienced users out there who run their shell with all of its configurations in this file alone. The `rc` is arguably the most important config file for your shell. (The second most important is the file that makes it look cool.)

Let's start configuring this thing. We'll continue using `zsh` as our example here, and will pull the important lines from the generated `zshrc` that comes with a `zsh` install, as well as add a few of our own.

```bash
# .zshrc

HIST_STAMPS="mm/dd/yyyy"
ENABLE_CORRECTION="false"

if [ -f ~/.aliases ]; then
    . ~/.aliases
fi

eval "$(dircolors -p | \
    sed 's/ 4[0-9];/ 01;/; s/;4[0-9];/;01;/g; s/;4[0-9] /;01 /' | \
    dircolors /dev/stdin)"

function precmd {
    if ! typeset -f deactivate >/dev/null; then
        for activate in ./venv/bin/activate ./.venv/bin/activate; do
            if [ -e "$activate" ]; then
                . "$activate"
            fi
        done
    fi
}

if ! [[ -n $SSH_CONNECTION ]]; then
  if [ "$TMUX" = "" ]; then tmux; fi
fi
```

We'll break this down line-by-line:

* `HIST_STAMPS="mm/dd/yyyy"` describes the timestamp formatting for our shell history.
    
* `ENABLE_CORRECTION="false"` sets command-line autocorrection.
    
* The `if` block checks for our `.aliases` config file, and makes sure `zsh` sees it.
    
* The `eval` statement removes the background colors of directories when running `ls` commands.
    
* The `function precmd` will activate a venv when navigating to a directory that holds a venv.
    
* Lastly, the `if` block at the bottom will activate `tmux` when the shell is launched, as long as it's not over a `ssh` connection.
    

### The `env` file

The `env` (environment) file contains our exported variables, we want to put variables that other programs will see or use in here. Some examples (from the [previous StackExchange answer](https://unix.stackexchange.com/a/71258)) include your `$PATH`, `$EDITOR` and `$PAGER` environment variables. The zsh `.zshenv` file is always [sourced](https://ss64.com/bash/source.html).

Here are some lines from my `.zshenv` file:

```bash
# .zshenv

export DOTFILES=$HOME/.dotfiles
export DOCKER=podman
export PATH=$HOME/bin:/usr/local/bin:$PATH
export LANG=en_US.UTF-8

# Preferred editor for local and remote sessions
if [[ -n $SSH_CONNECTION ]]; then
  export EDITOR='vim'
else
  export EDITOR='hx'
fi
```

First, we create a `DOTFILES` variable that describes the location of our dotfiles (I'll go over this later, but this is where we'll store our configs for backups and version control).

Next, I prefer to use Podman over Docker, so for convenience, I have the `$DOCKER` env variable point to `podman` instead.

Then, a line that adds `$HOME/bin` and `/usr/local/bin` to `$PATH`, then we set `$LANG` to `en_US.UTF-8`.

In the `if` block here at the end, we set `$EDITOR` to Helix if we're not on a `ssh` connection, and Vim if we are connected via `ssh`.

Now we'll take a look at the contents of the `.aliases` file we dropped into our `.zshrc` earlier.

### The `aliases` file

This one will be brief, as everyone's aliases are very different from everyone else's. Most of my aliases are `ssh` commands to quickly jump to other hosts, but there's a few others that I use as well.

```bash
# .aliases

# grep
alias grep='grep --color=auto'
alias fgrep='fgrep --color=auto'
alias egrep='egrep --color=auto'

# cat
alias cat='bat --theme Nord -p'

# ls 
alias ll='ls -Al'
alias la='ls -A'
alias ls='ls --color=auto'
alias l='ls -CF'

# dir
alias dir='dir --color=auto'
alias vdir='vdir --color=auto'

# opening files in your $EDITOR
alias ali='$EDITOR ~/.aliases'
alias zrc='$EDITOR ~/.zshrc'

# source config files
alias arc='source ~/.aliases'
alias src='source ~/.zshrc'

# system updates (this is for rpm-based systems like Fedora)
alias upd='sudo dnf upgrade -y && flatpak update'
```

These are pretty straightforward. We use the `alias` keyword followed by a key-value pair describing our alias and the command it represents. The `ls`, `grep`, and `dir` aliases here can be copied and pasted without issue; the `cat` alias will require [`bat`](https://github.com/sharkdp/bat), a fancy `cat` alternative. The rest are shortcuts to open or source config files in the set `$EDITOR` (from the `env` file), plus one to run system updates on Fedora.

Zsh will read these as though they were directly in the `.zshrc`. I like to keep the aliases separate, it feels more organized to me, but there's no issue at all with dropping them straight into the `rc` if you like everything in one place. They will both work the same.

These examples work independently from one another; when you come across examples like these, try experimenting with them line-by-line, and work them into your own configuration, instead of dropping them all blindly in your config.

## How Your Shell Loads Its Configs

If you want to segment out your config options, aliases, environment variables, functions, etc. into these separate files, then you should understand the order in which your shell loads them, to avoid any conflicts and redundancies.

These config files are really just scripts, like any other shell script, that are called when the shell is run or exited.

Here's how `zsh` will load its config files, from [this great post](https://shreevatsa.wordpress.com/2008/03/30/zshbash-startup-files-loading-order-bashrc-zshrc-etc/):

```bash
+----------------+-----------+-----------+------+
|                |Interactive|Interactive|Script|
|                |login      |non-login  |      |
+----------------+-----------+-----------+------+
|/etc/zshenv     |    A      |    A      |  A   |
+----------------+-----------+-----------+------+
|~/.zshenv       |    B      |    B      |  B   |
+----------------+-----------+-----------+------+
|/etc/zprofile   |    C      |           |      |
+----------------+-----------+-----------+------+
|~/.zprofile     |    D      |           |      |
+----------------+-----------+-----------+------+
|/etc/zshrc      |    E      |    C      |      |
+----------------+-----------+-----------+------+
|~/.zshrc        |    F      |    D      |      |
+----------------+-----------+-----------+------+
|/etc/zlogin     |    G      |           |      |
+----------------+-----------+-----------+------+
|~/.zlogin       |    H      |           |      |
+----------------+-----------+-----------+------+
|                |           |           |      |
+----------------+-----------+-----------+------+
|                |           |           |      |
+----------------+-----------+-----------+------+
|~/.zlogout      |    I      |           |      |
+----------------+-----------+-----------+------+
|/etc/zlogout    |    J      |           |      |
+----------------+-----------+-----------+------+
```

If you use `bash`, here's a table describing the equivalent files and their load order in `bash` from the [same post](https://shreevatsa.wordpress.com/2008/03/30/zshbash-startup-files-loading-order-bashrc-zshrc-etc/).

```bash
+----------------+-----------+-----------+------+
|                |Interactive|Interactive|Script|
|                |login      |non-login  |      |
+----------------+-----------+-----------+------+
|/etc/profile    |   A       |           |      |
+----------------+-----------+-----------+------+
|/etc/bash.bashrc|           |    A      |      |
+----------------+-----------+-----------+------+
|~/.bashrc       |           |    B      |      |
+----------------+-----------+-----------+------+
|~/.bash_profile |   B1      |           |      |
+----------------+-----------+-----------+------+
|~/.bash_login   |   B2      |           |      |
+----------------+-----------+-----------+------+
|~/.profile      |   B3      |           |      |
+----------------+-----------+-----------+------+
|BASH_ENV        |           |           |  A   |
+----------------+-----------+-----------+------+
|                |           |           |      |
+----------------+-----------+-----------+------+
|                |           |           |      |
+----------------+-----------+-----------+------+
|~/.bash_logout  |    C      |           |      |
+----------------+-----------+-----------+------+
```

Assuming you've launched an interactive `zsh` login, the shell will check for config files in `/etc` first, load the contents from those files, then check your home directory.

The `zshenv` file is loaded first, then your `zprofile` (Here, we want to place our configs for login shells. This is an alternative to the `login` file (or `.zlogin`), which is sourced *after* the `rc`). Next is our `zshrc`, and finally, the `zlogin`. The `zlogout` file is executed when you exit the interactive shell.

Notice that on an interactive non-login instance, our `zshenv` and `zshrc` files are the only files that are sourced, and the `zshenv` file is the only file sourced from scripts.

## Bonus: Managing Your Dotfiles with GNU Stow

So you've got all these files now, but how do you organize and manage them? If you've consolidated everything to just the `.zshrc`, then there are no issues with just dropping it in your home directory and leaving it at that, but if you expect your configs to grow and want a way to organize them, keep backups, and use some version control then you can place them all in a git repository and use a symbolic link manager like [GNU Stow](https://www.gnu.org/software/stow/).

We'll start by taking a look at `stow` and building out a directory structure that it can use to manage our dotfiles. Create a directory in your `$HOME` called `.dotfiles` and create a `zsh` subdirectory inside it, then move (`mv`, not `cp`) the dotfiles we've worked on to that directory, like so:

```bash
$ tree -a ~/.dotfiles 
/home/btp/.dotfiles/
└── zsh
    ├── .aliases
    ├── .zshenv
    └── .zshrc

1 directory, 3 files
```

This `.dotfiles` directory will be our root directory when using `stow`, and inside this directory we'll be storing our `stow` "packages", which we'll be calling `stow` on to install.

In this example, we'll be creating a `zsh` package which will install the three config files we've created to the target directory we give to `stow`.

By default, the target directory is the directory above the `stow` root directory, which is our home directory in this case. If you want to change the target directory, add the `-T /path/to/target` flag and argument.

First, let's install this `zsh` package with `stow`. Run the following command in your `.dotfiles` directory.

```bash
~/.dotfiles] $ stow --stow zsh
```

This will call stow on our `zsh` package, and install our dotfiles in the directory above our `.dotfiles` directory, which is `$HOME`. The `--stow` flag can also be `-S`. Stow uses symbolic links, and "installs" by creating the symbolic link in the target directory.

This is the same as if you would run `ln -s /home/user/.dotfiles/zsh/.zshrc /home/user/` for each of these files.

Now consider the following `stow` packages:

```bash
$ tree -a ~/.dotfiles
/home/btp/.dotfiles
├── alacritty
│   └── .config
│       └── alacritty
│           └── alacritty.toml
└── zsh
    ├── .aliases
    ├── .zshenv
    └── .zshrc

4 directories, 4 files
```

In this case, we've added a config package for the terminal Alacritty. Stow will check the full path under the package, and mirror it to our target location. When running the `stow` command on this package, it will add the full config path `~/.config/alacritty/alacritty.toml`, which is exactly where we want it.

You can layer your Stow root directory however you want, but when you install packages, you'll need to navigate to the directory that holds the package itself; Stow will not accept a path to a package:

\# this will NOT work.

```bash
~] $ stow -S .dotfiles/zsh
```

I keep my `.dotfiles` in [a git repository](https://codeberg.org/btp/dotfiles) and hold configs for multiple operating systems there, then navigate to the relevant OS directory for installs, like so.

```bash
tree ~/.dotfiles/
└── dots
    ├── linux
    │   ├── alacritty
    │   ├── awesome
    │   ├── helix
    │   ├── nushell
    │   ├── nvim
    │   ├── p10k
    │   ├── powershell
    │   ├── starship
    │   ├── tmux
    │   └── zsh
    ├── macos
    │   ├── alacritty
    │   ├── helix
    │   ├── nushell
    │   ├── nvim
    │   ├── p10k
    │   ├── powershell
    │   ├── starship
    │   ├── tmux
    │   └── zsh
    └── wsl
        ├── helix
        ├── nushell
        ├── p10k
        ├── powershell
        ├── starship
        ├── tmux
        └── zsh
```

## Conclusion

I tried to limit the scope of this article to avoid confusion, and also to avoid a massive post. This topic is very subjective, so take these examples as just that, examples. I've found what works for me when building and managing my config files in a scaleable way, and I hope this has helped you as well. Enjoy!