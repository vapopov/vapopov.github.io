---
title: "Base usefull commands"
date: 2017-07-21T07:35:01+02:00
draft: false
tags: ["bash", "rsync", "redis"]
---

## Redis multi actions

Remove values by pattern
```bash
redis-cli -h dev-1.example.com EVAL "local keys = redis.call('keys', ARGV[1])
for i=1,#keys,5000 do
  redis.call('del', unpack(keys, i, math.min(i+4999, #keys))) 
end 
return keys" 0 pattern:*
```
Check TTL by pattern
```bash
redis-cli -h dev-1.example.com EVAL "
local ttls = {}
for i, name in ipairs(redis.call('KEYS', ARGV[1])) do 
  ttls[i] = {redis.call('ttl', name), name}; 
end
return ttls" 0 *pattern*
```

## Rsync

### Create and transfer archive directly to the remote host
```bash
$ tar -zc ./path | ssh user@remote.host "cat > ~/file.tar.gz"
$ tar -zc ./path | ssh user@remote.host "tar -zx -C /destination"
```

### Transfer data with excluding files by pattern
```bash
rsync -e ssh -azv --exclude 'exclude.py' user@example.com:/var/www/example.com/ /var/www/example.com/
```

### Sync files and directory locally

```bash
$ rsync -zvh backup.tar /tmp/backups/
created directory /tmp/backups
backup.tar
sent 14.71M bytes  received 31 bytes  3.27M bytes/sec
total size is 16.18M  speedup is 1.10
```
-z, --compress          compress file data during the transfer
-v, --verbose           increase verbosity
-h, --human-readable    output numbers in a human-readable format

### Rsync Over SSH
To specify a protocol with rsync you need to give “-e” option with protocol name you want to use. Here in this example, We will be using “ssh” with “-e” option and perform data transfer.

```bash
$ rsync -azv -e ssh fahlo@store-staging.fahlo.me:/var/www/store.fahlo.me /var/www/store.fahlo.me
```
-a, --archive           archive mode; equals -rlptgoD (no -H,-A,-X), recursive, links, permissions, group, owner, devices
-z, --compress          compress file data during the transfer
-v, --verbose           increase verbosity
-h, --human-readable    output numbers in a human-readable format 

### Use of –include and –exclude Options
These two options allows us to include and exclude files by specifying parameters with these option helps us to specify those files or directories which you want to include in your sync and exclude files and folders with you don’t want to be transferred.

```bash
$ rsync -avze ssh --include 'R*' --exclude '*' root@192.168.0.101:/var/lib/rpm/ /root/rpm
```

### Use of –delete Option
If a file or directory not exist at the source, but already exists at the destination, you might want to delete that existing file/directory at the target while syncing .

We can use ‘–delete‘ option to delete files that are not there in source directory.

```bash
$ rsync -avz --delete root@192.168.0.100:/var/lib/rpm/ .
```

### Automatically Delete source Files after successful Transfer (with --dry-run)

```bash
$ rsync --dry-run --remove-source-files -zvh backup.tar /tmp/backups/
```
