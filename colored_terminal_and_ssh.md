# Colored Terminal and SSH
Or, "why do all my colors go away when I SSH into a server?"

I noticed this on my Debian Stretch (9.5) server while connecting from a Windows client using Git Bash + ssh + cmder. Other distros and setups may vary.

## Make sure `~/.bashrc` is getting `source`'d when you log in
Your ~/.bashrc on the server might be specifying colors, but it doesn't run when you log in via SSH. Change that by adding the following to `~/.bash_profile`:

``` bash
if [ -f ~/.bashrc ]; then
  . ~/.bashrc
fi
```

## Force colors
Add/uncomment this line in `~/.bashrc`:

``` bash
force_color_prompt=yes
```

This helped when using Git Bash on Windows since $TERM was set to `cygwin` instead of e.g. `xterm-256color`.

## Bonus: Add current git branch to prompt
https://askubuntu.com/questions/730754/how-do-i-show-the-git-branch-with-colours-in-bash-prompt

References:
* http://www.joshstaiger.org/archives/2005/07/bash_profile_vs.html
* https://stackoverflow.com/questions/820517/bashrc-at-ssh-login
