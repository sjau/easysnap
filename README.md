# easysnap

A little zfs snapshot tool including backup script. The tool consists of two scripts:

* easysnap: It will zfs snapshots and remove old ones if necessary. No configuration file needed as all required information is in the dataset's user properties.
* easysnapRecv: It will allow to pull snapshots made by easysnap from other servers. A configuration file is needed.
* easysnapRm: This will remove surplus snapshots.
* easysnapList: This will list all datasets that have easysnap:xxx property set or are found in the easysnapRecv configuration file

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
1. Also set a cron or systemd timer to run the easysnapRm script periodically to remove surplus snapshots from the datasets. Maybe once or twice a day should be sufficient.

### Format of the snapshots

Snapshots taken with easysnap look like this:

```
pool/path/to/DS@UNIXTIMESTAMP-easysnap-hourly-UTC-2018-10-25-20-00
pool/path/to/DS@UNIXTIMESTAMP-easysnap-frequent-UTC-2018-10-25-20-00

```

* For sorting order and easy readability the first part contains the unix timestamp, followed by the easysnap-interval and it is set to UTC.
* The second part contains date and time in easy readable format (YYYY-MM-DD-HH-mm).

### Samba Shadow Copy2

The above chosen format of the snapshots can be used in conjunction with [samba's shadow_copy2](https://www.samba.org/samba/docs/current/man-html/vfs_shadow_copy2.8.html) functionality. The following can be used on a share to enable it:

```
vfs objects = shadow_copy2
shadow:snapdir = .zfs/snapshot
shadow:sort = desc
shadow:format = -%Y-%m-%d-%H-%M
shadow:snapprefix = .*
shadow:delimiter = -20
shadow:localtime = yes
```

In NixOS use the above globally in the .extraConfig= '' ... '' or in individual shares like:

```
    "vfs objects" = "shadow_copy2";
    "shadow:snapdir" = ".zfs/snapshot";
    "shadow:sort" = "desc";
    "shadow:format" = "-%Y-%m-%d-%H-%M";
    "shadow:snapprefix" = ".*";
    "shadow:delimiter" = "-20";
    "shadow:localtime" = "yes";
```


Existing easysnap snapshots can be renamed using the following script:

```
#!/usr/bin/env bash

ds="pool/path/to/dataset"
mount="/path/to/mountpoint"

for f in ${mount}/.zfs/snapshot/*easysnap* ; do
        f="${f##*/}"
    if [[ ${f} == easysnap* ]]; then
        es="${f%%_*}"       # Get easysnap-{interval} substring
        gd="${f##*_}"       # Get the date / time substring
        # Parse the date / time substring
        while IFS="-" read -r year month day hour minute tz; do
            td="${year}-${month}-${day} ${hour}:${minute}:00 UTC"
            ts=$(date -d "${td}" +%s)       # Make timestamp

            # Make new format string
            fnew="${ts}-${es}-UTC-${year}-${month}-${day}-${hour}-${minute}"
            printf '%s -> %s\n' "${f}" "${fnew}"
#            zfs rename ${ds}@${f} ${ds}@${fnew}
        done <<< "${gd}"
    fi
done
```

Adjust the dataset and mountpoint variable to your system and run it. It will only convert existing `easysnap` snapshots - others will be left untouched. You'll get an output of which snapshot is transformed into what. Once you verified that it works ok, uncomment the line with `zfs rename`

## esaysnapRecv

easysnapRecv is a simple bash script which will zfs send/receive datasets or incremental snapshots from the same machine or a remote one to the machine where the script is run (pull). It also can handle encrypted datasets. By default it will just send/recv the incremental snapshots. It can optionally run scripts before and after receiving snapshots.

### How to use

1. You need to make a configuration file in `/etc/easysnap/` named it `easysnapRecv`
1. The config has the following format: `localDS;keyDS;raw;intermediary;exportPool;remoteHost;remoteDS,reqFreeG;snapsXX`
   * `localDS`: The path to the local dataset that shall be used, e.g. `tank/path/to/Backups/xxx`. __Notice:__ If the local dataset does not exist, it will be created and it will just have the latest snapshot. If you want to have also previous intermediary snapshots, then you should first make a zfs send/recv starting from the snapshot you want.
   * `keyDS`: In case you make use of encryption, you will need to provide the path to the dataset that holds the encryption key. Because of inheritance, child datasets of an encrypted dataset will also be encrypted, but you'll need to provide the key just to the parent dataset with the original encryption. In my case I just do for all my encrypted zfs the following: `tankXXX/encZFS` and then I create child datasets like `tankXXX/encZFS/Nixos` -> so the value to provide would be `tankXXX/encZFS`. If your local dataset has no encryption, just leave this empty.
   * `raw`: In case the remote dataset that you want to receive is encrypted you can set the raw flag to `y`, then it will be sent in raw mode. This is espeically useful when the receiving machine is untrusted, e.g. features no encryption by default. That way the receiving dataset will still be encrypted.
   * `intermediary`: By default easysnap will use the `-i` flag to just send an incremental snapshot. If you also want intermediary snapshots then set this option to `y` and easysnap will try to send the intermediary snapshots by using the `-I` flag. However, if intermediary sending fails, it will output a notice and retry with just incremental snapshot. __Notice:__ If you use the intermediary sending, it may also send snapshots from other tools and other intervals. E.g. if you just do a daily send, it will for example also include hourly and frequent snapshots and they won't get removed by the deleting, as deleting of snapshots strictly adheres to the chosen interval.
   * `exportPool`: Set to `y` if you want to export the pool after receiving is done. This can be useful if you want to make backups onto removable media such as usb thumb drives or external usb drives. Easysnap will automatically try to import the pool for receiving when it's not imported already. The export happens at the very end of routine.
   * `remoteHost`: Provide the remote user and host that shall send the dataset, e.g. `root@myserver.tld` or you can also use like `root@localhost`
   * `remoteDS`: The path to the remote dataset that shall be sent.
   * `reqFreeG`: Give notice when there's less space than the amount of gigabytes indicated on the pool, e.g. `200` would give a warning when the free space on the pool falls below 200G.
   * `snapsXX`: Here you define the intervals and the amount of snaps it should keep, e.g. ...;daily:3650;hourly:2160;....:99. You can add as many snapsXX there as you want. The give example would keep 10 years worth of daily snaps and 90 days worth of hourly snaps. You can also use -1 and 0 for the amount. See below:
      * `-1`: To keep an unlimited amount of snapshots (never delete them).
      * `0`: To delete all snapshots but still receive current/incremental data.
      * `1+`: The amount of snapshots to keep.
   * __Notice:__ easysnapRecv will by default use the `-F` flag, meaning that it will auto-rollback to the latest snapshot when required.
   * Empty lines and lines starting with # are ignored
1. Setup a cron or systemd timer that will run the easysnapRecv script when you want to.

For running scripts before / after receiving snapshots you just call the script with positional parameters:

`./easysnapRecv /path/to/preRunScript /path/to/postRunScript`

**Caveat:** Because positional parameters are being used, you'll need to provide a preRunScript (or path to an empty script) if you want to only run the postRunScript.

## esaysnapRm

easysnapRM is a simple bash script that loops through the dataset properties to find occurences of easynsap:xxx custom properties. It also parses the easysnapRecv config file (if present) and it will then remove surplus snapshots based on the limits provided. You can use options to only do test run and have verbose output. See help using:

`./easysnapRm -h`

### How to use

1. Just set a cron or systemd timer to run the script once or twice a day. No further configuration is needed.

## easysnapList

easysnapList is a simple bash script that loops through the dataset properties to find occurences of easynsap:xxx custom properties. It also parses the easysnapRecv config file (if present). It prints a list of found dataset incl. how many snapshots it should keep.
It contains a few options to allows you to output non-coloured text, only dataset with an easynspa custom property or only datasets in the `/etc/easynsap/easysnapRecv` file. See help using:

`./easysnapList -h`

### How to use

1. Just run the script from the terminal. No configuration necessary and it will not delete any snapshots.

## ToDo

- pre-/post snapshot hooks for stuff like database locking and flushing
- systemd timer examples
