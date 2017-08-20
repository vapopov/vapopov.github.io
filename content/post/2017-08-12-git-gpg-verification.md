---
title: "Git Gpg Verification"
date: 2017-08-12T19:20:06+02:00
draft: false
tags: ["git", "gpg", "ssh"]
---

# GPG and git on MacOS

## Setup

- Install https://gpgtools.org -- I'd suggest to do a **customized install** and deselect **GPGMail**.
- Create or import a key -- see below for https://keybase.io
- Run `gpg -K` to see all private keys in current machine, use the key ID for the next step (each gpg key has subkeys with different capabilities, its better to choose subkey with sign `S`)
- Configure `git` to use GPG -- replace the key with the one from `gpg --list-secret-keys`

```
git config --global gpg.program /usr/local/MacGPG2/bin/gpg2
git config --global user.signingkey E5FEE3B2 
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

## Use GPG as SSH authentication 

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
GPG_TTY=$(tty)
export GPG_TTY
```
`GPG_TTY` is needed in case if you don't want to use pinentry program from GPGSuite, and must show password dialog in your current active tty 

Then we need to export public keys of sub-keys
```bash
$ gpg2 -K
/Users/vadym/.gnupg/secring.gpg
-------------------------------
sec   4096R/E5FEE3B2 2017-08-11
uid                  Vadym Popov <me@vpopov.org>
uid                  [jpeg image of size 22756]
ssb   4096R/8A2940A7 2017-08-11
ssb   4096R/0D8DE79D 2017-08-17
ssb   4096R/BB84E418 2017-08-17
```

On the moment of writing this article there were some issue with gpg-agent, so I had to export all private keys from old version of gpg that is delivered with GPGSuite to import to latest GnuGPG version.
Export the authentication private key and subkey and import to newest version of GnuGPG:
```bash 
/usr/local/MacGPG2/bin/gpg2 --armor --export-secret E5FEE3B2 > private.gpg
/usr/local/MacGPG2/bin/gpg2 --armor --export-secret-subkeys --export-options export-reset-subkey-passwd BB84E418 > private_sub.gpg 
gpg --import private.gpg
gpg --import private_sub.gpg
```

So now we need add our subkey with auth capability to sshcontrol file and restart gpg-agent
```bash
$ gpg -K --with-keygrip
/Users/vadym/.gnupg/pubring.kbx
-------------------------------
sec   rsa4096 2017-08-11 [SC]
      BFA968E385CDB1DA786BCEB7518ABC2FE5FEE3B2
      Keygrip = 2A76E01E169B09A13F85CD2C755CEBDB46E7A86A
uid        [ unbekannt ] Vadym Popov <me@vpopov.org>
uid        [ unbekannt ] [jpeg image of size 22756]
ssb   rsa4096 2017-08-11 [E]
      Keygrip = F820B0F81509FBC09AB35E25E3DD02FB432E2D3D
ssb   rsa4096 2017-08-17 [S]
      Keygrip = 5ABA3AE4B2D9D3653383B231F56DE540CD38436E
ssb   rsa4096 2017-08-17 [A]
      Keygrip = A46472D1A1C6E67DB78BC7986338CD5B6E5DEE4C
```
Keygrip of our authentification key `A46472D1A1C6E67DB78BC7986338CD5B6E5DEE4C` must be added to `~/.gnugpg/sshcontrol`, after this our key should be visible for `ssh-add -l`
```
$ ssh-add -l
4096 SHA256:iV18sELhAtjOXZ+UD4bwdTcC/spSSaFpoQrMEr5lDzk (none) (RSA)
```

## Links

 * https://github.com/pstadler/keybase-gpg-github
 * https://alexcabal.com/creating-the-perfect-gpg-keypair/
 * http://www.integralist.co.uk/posts/security-basics.html
 * https://www.esev.com/blog/post/2015-01-pgp-ssh-key-on-yubikey-neo/

