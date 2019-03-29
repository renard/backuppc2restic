# Migrate BackupPC to restic

This document expains how to migrate your
[BackupPC](https://backuppc.github.io/backuppc/) backups to
[restic](https://restic.net).

Please note this is not perfect neither optimal. I needed to run this once
so I didn't take time to write a state-of-the-art tool for that. Use it at
your own risk.


This is the process that I followed for my backups migration but you millage
may vary. You may need to adapt this to your own environment. Feel free to
ask me some details if you need some.

I won't fix anything in this repository since it is more here for a
documentation purpose.


## Setup your environment

I assume BackupPC is install on a server (such as `bkp0`) and restic on an
other host (`store0`).

Create a restic respository on `store0`:

```shell
restic init -r /srv/backup/backuppc
```

And create a new server on `bkp0` called `store0`:

```perl
$Conf{ClientNameAlias} = '192.168.1.123';
$Conf{RsyncClientPath} = 'sudo /usr/bin/nice -n 19 /usr/bin/ionice -c3 /usr/bin/rsync';$Conf{RsyncClientCmd} = '$sshPath -l backuppc -q -x $host $rsyncPath $argList+';
$Conf{RsyncClientRestoreCmd} = '$sshPath -l backuppc  -q -x $host $rsyncPath $argList+';
$Conf{RsyncShareName} = [
  '/srv/restore'
];
```

Make sure that `backuppc` user can connect from `bkp0` to `store0` and
perform `sudo` operations. For example add this on the `store0` sudoers:

```
Cmnd_Alias BACKUPPC_RSYNC=/usr/bin/nice -n 19 /usr/bin/ionice -c3 /usr/bin/rsync *, /usr/bin/rsync
backuppc ALL=(ALL) NOPASSWD:BACKUPPC_RSYNC
```

What is going to be done:

* `backuppc` connects from `bkp0` to `store0`
* Perform all operation by *sudoing* a `rsync` command on `store0`.
* The restoration path is `/srv/restore/[SOURCE_PC]`

## Migration

If you just want to migrate your last backups you can do it using the backup
web interface. This process can be long.

If you need to migrate all your backups you can use the [gen-bpc-restore]()
script to generate the restoration files.

```
gen-bpc-restore  serverA
```

This will generate several files such as:

* `serverA_1324_1`: Restoration file for 1st share of backup 1324
* `serverA_1324_2`: Restoration file for 2nd share of backup 1324
* `serverA_1324.txt`: Commands to run to restore serverA's backup 1324


The commands to run are following On `store0` create the target diretories:

```shell
mkdir /srv/restore/serverA_1324/etc
mkdir /srv/restore/serverA_1324/srv
```

On `bkp0` restore both shares:

```shell
time sudo -u backuppc /usr/share/backuppc/bin/BackupPC_restore store0 store0 serverA_1324_1
time sudo -u backuppc /usr/share/backuppc/bin/BackupPC_restore store0 store0 serverA_1324_2
```

Finally backup the restoration to restic repository:
```shell
mv /srv/restore/serverA_1324 /srv/restore/cw-bkp0
time restic --host 'serverA' -r /srv/backup/backuppc backup --verbose --time '2018-08-08 16:00:42.153007776' --tag 1324 /srv/restore/serverA
mv /srv/restore/serverA_
time rm -rf /srv/restore/serverA_
```

## Performances

The migration process takes ages since all backups have to be restored. This
takes even longer is you have slow writing harddrives. Comparing speed here
is non-sense since too many parameters have to be taken in account.

As an example I restored 1.5Tb of compressed backups for 22 hosts for a
total of:

* 285 full backups of total size 47668.72GB (prior to pooling and compression),
* 41 incr backups of total size 300.50GB (prior to pooling and compression).

It took me several weeks and isn't finished yet (at time of writing).


It takes about 2 to 3 hours to restore 100Gb of data and one extra hour to
import into restic. The bottleneck here is mainly the harddrives that have
poor writing performances about 15MB/s (120Mb/s). Those metrics may vary
with the kind of data to migrate.


## Notes

As you may notice each backup has to be fully restored. I didn't manage to
make the `--delete` rsync work in `RsyncRestoreArgs` for `store0`. Thus this
ensure the backup is restored as it should be with any extra files.

If your backup has some kind of append-only backups (such a photo library)
you can only restore the last backup since there is no really reason to keep
all intermediate ones.

If you need to see what files have been modified since the previous backup
you can use following command on `bkp0`:

```shell
/usr/share/backuppc/bin/BackupPC_zcat XferLOG.1324.z | grep -vE '^(  same|  create d|  pool) ' | less
```

See https://backuppc.github.io/backuppc/BackupPC.html#BackupPC-operation for
details on *XferLOG* fields description. Here we remove the existing files,
the created directories and files found in the pool.


## Copyright

Author: Sébastien Gross

License: WTFPL, grab your copy here: http://www.wtfpl.net
