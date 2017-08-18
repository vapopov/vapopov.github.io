---
title: "Git Gpg Verification"
date: 2017-08-12T19:20:06+02:00
draft: false
tags: ["git", "gpg"]
---

# GPG and git on macOS

## Setup

*No need* for homebrew or anything like that. Works with https://www.git-tower.com and the command line.

- Install https://gpgtools.org -- I'd suggest to do a **customized install** and deselect **GPGMail**.
- Create or import a key -- see below for https://keybase.io
- Run `gpg --list-secret-keys` and look for `sec`, use the key ID for the next step
- Configure `git` to use GPG -- replace the key with the one from `gpg --list-secret-keys`

```
git config --global gpg.program /usr/local/MacGPG2/bin/gpg2
git config --global user.signingkey A6B167E1 
git config --global commit.gpgsign true 
```
Add this line to `~/.gnupg/gpg-agent.conf`
```
pinentry-program /usr/local/MacGPG2/libexec/pinentry-mac.app/Contents/MacOS/pinentry-mac
```
Add this line to `~/.gnupg/gpg.conf`
```
no-tty
```

## Keybase.io

Import key to GPG on another host

```
% keybase pgp export
% keybase pgp export -q E5FEE3B2 | gpg --import
% keybase pgp export -q E5FEE3B2 --secret | gpg --allow-secret-key-import --import
```

Add public GPG key to GitHub

```
% open https://github.com/settings/keys
% keybase pgp export -q E5FEE3B2 | pbcopy
```

## Use GPG with ssh-agent

On your Mac edit the file ~/.gnupg/gpg-agent.conf to contain the following:

```
default-cache-ttl 600
max-cache-ttl 7200
pinentry-program /usr/local/MacGPG2/libexec/pinentry-mac.app/Contents/MacOS/pinentry-mac
enable-ssh-support
write-env-file
```

Now edit ~/.profile to contain the following:

```bash
# Enable GPG keys for SSH Auth
if [ -f "${HOME}/.gpg-agent-info" ]; then
     . "${HOME}/.gpg-agent-info"
       export GPG_AGENT_INFO
       export SSH_AUTH_SOCK
       export SSH_AGENT_PID
fi
```

Then we need to export public keys of sub-keys

```bash
# gpg2 --list-keys
/Users/vadym/.gnupg/pubring.gpg
-------------------------------
pub   4096R/E5FEE3B2 2017-08-11
uid       [ unknown] Vadym Popov <me@vpopov.org>
sub   4096R/8A2940A7 2017-08-11
sub   4096R/565DA837 2017-08-15
sub   4096R/0D8DE79D 2017-08-17
```

Export the authentication private and public subkey:
```bash 
gpg2 --export-secret-subkeys --export-options export-reset-subkey-passwd 8A2940A7 | /usr/local/MacGPG2/bin/gpgkey2ssh 8A2940A7 > gpg-auth-keyfile

```

## See Also

 * https://github.com/pstadler/keybase-gpg-github
 * `/usr/local/MacGPG2` -- this is where MacGPG binaries live
 * https://gpgtools.org
 * https://www.git-tower.com



[https://gist.github.com/danieleggert/b029d44d4a54b328c0bac65d46ba4c65]
