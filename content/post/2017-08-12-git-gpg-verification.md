---
title: "2017 08 12 Git Gpg Verification"
date: 2017-08-12T19:20:06+02:00
draft: false
tags: ["git", "gpg"]
---

# GPG and git on macOS

## Setup

*No need* for homebrew or anything like that. Works with https://www.git-tower.com and the command line.

1. Install https://gpgtools.org -- I'd suggest to do a **customized install** and deselect **GPGMail**.
1. Create or import a key -- see below for https://keybase.io
1. Run `gpg --list-secret-keys` and look for `sec`, use the key ID for the next step
1. Configure `git` to use GPG -- replace the key with the one from `gpg --list-secret-keys`
```
git config --global gpg.program /usr/local/MacGPG2/bin/gpg2
git config --global user.signingkey A6B167E1 
git config --global commit.gpgsign true 
```
1. Add this line to `~/.gnupg/gpg-agent.conf`
```
pinentry-program /usr/local/MacGPG2/libexec/pinentry-mac.app/Contents/MacOS/pinentry-mac
```
1. Add this line to `~/.gnupg/gpg.conf`
```
no-tty
```

## Keybase.io

### Import key to GPG on another host

```
% keybase pgp export
% keybase pgp export -q CB86A866E870EE00 | gpg --import
% keybase pgp export -q CB86A866E870EE00 --secret | gpg --allow-secret-key-import --import
```

### Add public GPG key to GitHub

```
% open https://github.com/settings/keys
% keybase pgp export -q CB86A866E440EE00 | pbcopy
```

## See Also

 * https://github.com/pstadler/keybase-gpg-github
 * `/usr/local/MacGPG2` -- this is where MacGPG binaries live
 * https://gpgtools.org
 * https://www.git-tower.com



[https://gist.github.com/danieleggert/b029d44d4a54b328c0bac65d46ba4c65]
