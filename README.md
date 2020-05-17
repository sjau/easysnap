# easysnap

A little zfs snapshot tool including backup script. The tool consists of two scripts:

* easysnap: It will zfs snapshots and remove old ones if necessary. No configuration file needed as all required information is in the dataset's user properties.
* easysnapRecv: It will allow to pull snapshots made by easysnap from other servers. A configuration file is needed.

## easysnap

easysnap is a simple bash script which will take snapshots of designated datasets by adding a user property `esaysnap:xxx` with a value. Changing the value will affect how easysnap will take snapshots on said dataset.

### How to use

1. Clone this repo or just fetch the easysnap script and make it executable.
1. You need to set a user property on the desired dataset. You can do this with `zfs set easysnap:hourly="240:200:m" pool/path/to/dataset`, where
   * `easysnap:hourly` is the property name.
      * `easysnap:` is the mandatory part. Easysnap will search for this property and must not be changed.
      * `hourly` is also mandatory but it can be anything. Instead of `hourly` you can provide any string; common ones would be `frequent`, `hourly`, `daily`, `weekly`, `monthly` because they can easily be scripted to run by cron or systemd timers.
   * `240:200`:
      * `240` is mandatory and it indicates that it should keep 240 snapshots. Instead for a positive number of snapshots you could also use:
         * `-1`: keep unlimited snapshots; or
         * `0`: remove all easysnap:xxx snapshots; or
         * `false `: do not not make any snapshots/changes.
         * __Notice:__ easysnap only removes/adjusts the number of snapshots provided for this interval. E.g. if you run the `./easysnap hourly` routine then it will only remove snapshots containing the string `easysnap-hourly` in the snapshot name. No other snapshots are touched. This is intentional as you might want to make custom snapshots at some points and those should not be automatically removed.
      * `:200` is optional and it indicates that easysnap will print a notice when there's less than 200GB free space on the pool.
      * `:m` is optional and it indicates that MySQL/MariaDB tables should be locked before the snapshot and unlocked after the snapshot. If MySQL/MariaDB is started with privileges, you'll also need to create a `.my.cnf` file in the user's home under which the script is run, e.g. `/root/.my.cnf`, so that it reads out username and password from that file, see [.my.cnf.example](.my.cnf.example). If you don't want to be warned about remaining space, you can also just set the value like `240::m`.
1. When you have setup at least one dataset, you will need to add a cronjob or systemd timer to actually run the easysnap script with the interval that it should run. E.g. if you setup `easysnap:hourly` you could add a cron like `0 * * * * /path/to/easysnap hourly`. The parameter provided to the easysnap script (in this case `hourly`) must match the desired user property in the dataset `easysnap:hourly`. You can also run the script manually whenever you want, just call it like `./easysnap hourly`. In the [cron.example](cron.example) file you have examples for the common snapshot intervals.

### Format of the snapshots

Snapshots taken with easysnap look like this:

```
pool/path/to/DS@easysnap-hourly_2018-10-25-20-00-UTC
pool/path/to/DS@easysnap-frequent_2018-10-25-20-00-UTC

```

* For sorting order and easy readability the first part contans the date and time in `YYYY-MM-DD_HHhMM` format. It will use UTC instead of your custom timezone and adjust the date values accordingly.
* The second part is the `easysnap-interval`, so that we can make sure we only add/delete snapshots of the supplied interval.

### Samba Shadow Copy2

The above chosen format of the snapshots can be used in conjunction with [samba's shadow_copy2](https://www.samba.org/samba/docs/current/man-html/vfs_shadow_copy2.8.html) functionality. The following can be used on a share to enable it:

```
vfs objects = shadow_copy2
shadow:snapdir = .zfs/snapshot
shadow:sort = desc
shadow:format = _%Y-%m-%d-%H-%M-UTC
shadow:snapprefix = ^easysnap-\(frequent\)\{0,1\}\(hourly\)\{0,1\}\(daily\)\{0,1\}\(monthly\)\{0,1\}
shadow:delimiter = _
shadow: localtime = yes
```

In NixOS use:

```
    "vfs objects" = "shadow_copy2";
    "shadow:snapdir" = ".zfs/snapshot";
    "shadow:sort" = "desc";
    "shadow:format" = "_%Y-%m-%d-%H-%M-UTC";
    "shadow:snapprefix" = "^easysnap-\\(frequent\\)\\{0,1\\}\\(hourly\\)\\{0,1\\}\\(daily\\)\\{0,1\\}\\(monthly\\)\\{0,1\\}";
    "shadow:delimiter" = "_";
    "shadow:localtime" = "yes";
```

If you have defined different interval names than `frequent`, `hourly`, `daily` or `monthly` then adjust the `shadow:snapprefix` accordingly. If you only want to show old versions from `daily` snapshots, adjust the prefix accordingly.

Existing easysnap snapshots can be renamed using the following script:

```
#!/usr/bin/env bash

ds="pool/path/to/dataset"
mount="/path/to/mountpoint"

for f in ${mount}/.zfs/snapshot/*easysnap* ; do
        f="${f##*/}"
        timestamp="${f##*_}"
        re='^[0-9]+$'
        if ! [[ ${timestamp} =~ ${re} ]] ; then
            printf '%s\n' "Couldn't convert snapshot ${f}"
        else
            fnew=$(date -u --date=@$timestamp +easysnap_hourly_%Y-%m-%d-%H-%M-UTC);
            printf '%s -> %s\n' "${f}" "${fnew}"
#            zfs rename ${ds}@${f} ${ds}@${fnew}
        fi
done
```

Adjust the dataset and mountpoint variable to your system and run it. It will only convert existing `easysnap` snapshots - others will be left untouched. You'll get an output of which snapshot is transformed into what. Once you verified that it works ok, uncomment the line with `zfs rename`

## esaysnapRecv

easysnapRecv is a simple bash script which will zfs send/receive datasets or incremental snapshots from the same machine or a remote one to the machine where the script is run (pull). It also can handle encrypted datasets (that feature is still in zfs master). By default it will just send/recv the incremental snapshot

### How to use

1. You need to make a configuration file in `/etc/easysnap/` named like `easysnap.frequent`, `easysnap.hourly` etc. It must correspondent with the easysnap interval name chosen.
1. The config has the following format: `localDS,keyDS,raw,intermediary,exportPool,remoteHost,remoteDS,snaps,reqFreeG`
   * `localDS`: The path to the local dataset that shall be used, e.g. `tank/path/to/Backups/xxx`. __Notice:__ If the local dataset does not exist, it will be created and it will just have the latest snapshot. If you want to have also previous intermediary snapshots, then you should first make a zfs send/recv starting from the snapshot you want.
   * `keyDS`: In case you make use of encryption, you will need to provide the path to the dataset that holds the encryption key. Because of inheritance, child datasets of an encrypted dataset will also be encrypted, but you'll need to provide the key just to the parent dataset with the original encryption. In my case I just do for all my encrypted zfs the following: `tankXXX/encZFS` and then I create child datasets like `tankXXX/encZFS/Nixos` -> so the value to provide would be `tankXXX/encZFS`. If your local dataset has no encryption, just leave this empty.
   * `raw`: In case the remote dataset that you want to receive is encrypted you can set the raw flag to `y`, then it will be sent in raw mode. This is espeically useful when the receiving machine is untrusted, e.g. features no encryption by default. That way the receiving dataset will still be encrypted.
   * `intermediary`: By default easysnap will use the `-i` flag to just send an incremental snapshot. If you also want intermediary snapshots then set this option to `y` and easysnap will try to send the intermediary snapshots by using the `-I` flag. However, if intermediary sending fails, it will output a notice and retry with just incremental snapshot. __Notice:__ If you use the intermediary sending, it may also send snapshots from other tools and other intervals. E.g. if you just do a daily send, it will for example also include hourly and frequent snapshots and they won't get removed by the deleting, as deleting of snapshots strictly adheres to the chosen interval.
   * `exportPool`: Set to `y` if you want to export the pool after receiving is done. This can be useful if you want to make backups onto removable media such as usb thumb drives or external usb drives. Easysnap will automatically try to import the pool for receiving when it's not imported already. The export happens at the very end of routine.
   * `remoteHost`: Provide the remote user and host that shall send the dataset, e.g. `root@myserver.tld` or you can also use like `root@localhost`
   * `remoteDS`: The path to the remote dataset that shall be sent.
   * `snaps`: The amount of snapshots to keep locally. Remember: This only goes for the selected interval.
      * `-1`: To keep an unlimited amount of snapshots (never delete them).
      * `0`: To delete all snapshots but still receive current/incremental data.
      * `1+`: The amount of snapshots to keep.
   * `reqFreeG`: Give notice when there's less space than the amount of gigabytes indicated on the pool, e.g. `200` would give a warning when the free space on the pool falls below 200G.
   * __Notice:__ easysnapRecv will by default use the `-F` flag, meaning that it will auto-rollback to the latest snapshot when required.
1. Setup a cron or systemd timer that will run the easysnapRecv script when you want to. Use as first parameter the interval indicator, e.g. `0 * * * * /path/to/easysnap/easysnapRecv hourly`

## ToDo

- pre-/post snapshot hooks for stuff like database locking and flushing
- systemd timer examples
