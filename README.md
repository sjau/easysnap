# easysnap

A little zfs snapshot tool including backup script. The tool consists of two scripts:

* easysnap: It will zfs snapshots and remove old ones if necessary. No configuration file needed, info is in custom dataset properties.
* easysnapRecv: It will allow to pull snapshots from other servers. A configuration file is needed.

## easysnap

easysnap is a simple bash script which will take snapshots of designated datasets by adding a user property `esaysnap:xxx` with a value. Changing the value will affect how easysnap will take snapshots on said dataset.

### How to use

1. Clone this repo or just fetch the easysnap script and make it executable.
1. You need to a custom user property to the desired dataset. You can do this by `zfs set easysnap:hourly="240:200" pool/path/to/dataset`, where
   * `easysnap:hourly` is the property name.
      * `easysnap:` is the mandatory part. Easysnap will search for this property and must not be changed.
      * `hourly` is also mandatory but it can be anything. Instead of `hourly` you can provide any string; common ones would be `frequent`, `hourly`, `daily`, `weekly`, `monthly` because they can easily be scripted to run by cron or systemd timers.
   * `240:200`:
      * `240` is mandatory and it indicates that it should keep 240 snapshots. Instead for a positive number of snapshots you could also use:
         * `-1` (unlimited snapshots); or
         * `0` (remove all easysnap:xxx snapshots);
         * `false ` to not make any snapshots/changes.
      * `:200` is optionaal and it indicates that easysnap will print a notice when there's less than 200GB free space on the pool.
   * __Notice:__ easysnap removes/adjusts only the number of snapshots provided for this run. E.g. if you run the `./easysnap hourly` routine then it will only remove snapshots containing the string `easysnap-hourly` in the snapshot name. No other snapshots are touched. This is intentional as you might want to make custom snapshots at some points and those should not be automatically removed.
1. When you have setup at least one dataset, you will need to add a cronjob or systemd timer to actually call the easysnap tool with the jobname that it should run. E.g. if you setup `easysnap:hourly` you could add a cron like `0 * * * * /path/to/easysnap hourly`. The parameter provided to the easysnap script (in this case `hourly`) must match the desired user property in the dataset `easysnap:hourly`. You can also run the script manually whenever you want, just call it like `./easysnap hourly`. In the [cron.example](cron.example) file you have examples for the common snapshot intervals.

### Format of the snapshots

Snapshots taken with easysnap look like this:

```
pool/path/to/DS@1540490401_easysnap-hourly_2018-10-25_20h00-CEST
pool/path/to/DS@1540490401_easysnap-frequent_2018-10-25_20h00-CEST

```

For sorting order the first part `1540490401` is the unix timestamp. This will ensure, that we don't end up with two snapshots with the same name. The second part is the `easysnap-interval`, so that we can make sure we only add/delete snapshots of the supplied job interval given. The last part is date / time provided incl. timezone `2018-10-25_20h00-CEST` for easier understanding of when a snapshot was actually taken.


## esaysnapRecv

easysanpRecv is a simple bash script which will zfs send/receive datasets or incremental snapshots from the same machine or a remote one to the machine where the script is run. It also can handle encrypted datasets that are currently in zfs master. By default it will also send/recv the intermediary snapshots between to runs but that can be disabled if wanted. Also, if it fails to receive the intermediary snapshots it will fallback to just an incremental send.

### How to use

1. You need to make a configuration file in `/etc/easysnap` named linke `easysnap.frequent`, `easysnap.hourly` etc. It must correspondent with the easysnap frequency name chosen.
1. The config has the following format: `localDS,keyDS,raw,intermediary,exportPool,remoteHost,remoteDS,snaps,reqFreeG`
   * `localDS`: The path to the local dataset that shall be used, e.g. `tank/path/to/Backups/xxx`. __Notice:__ If the local dataset does not exist, it will be created and it will just have the latest snapshot. If you want to have also previous intermediary snapshots, then you should first make a zfs send/recv starting from the snapshot you want.
   * `keyDS`: In case you make use of encryption, you will need to provide the path to the dataset that holds the encryption (it's a different one when inheritance is used). I just by for all my encrypted zfs the following: `tankXXX/encZFS`. If your local dataset has no encryption, just leave it empty.
   * `raw`: In case the remote dataset that you want to receive is encrypted you can set the raw flag to `y`, then it will be sent in raw mode. This is espeically useful when the receiving machine is untrusted, e.g. features no encryption by default. That way the receiving dataset will still be encrypted.
   * `intermediary`: By default easysnap will use the `-i` flag to just send an incremental snapshot. If you also want intermediar snapshots then set this option to `y` and easysnap will try to send the intermediary snapshots also with the `-I` flag. However, if intermediary sending fails, it will output a notice and retry with just incremental snapshot. __Notice:__ If you use the intermediary sending, it may also send snapshots from other tools and other frequncies. E.g. if you just do a daily send, it will for example also include hourly and frequent snapshots and they won't get removed by the deleting, as deleting of snapshots strictly adheres to just the chosen frequency.
   * `exportPool`: Set to `y` if you want to export the pool after receiving is done. This can be useful if you want to make backups onto removable media such as usb thumb drives or external usb drives. Easysnap will automatically try to import the pool for receiving when it's not imported already. The export happens at the very end of routine.
   * `remoteHost`: Provide the remote user and host that shall send the dataset, e.g. `root@myserver.tld` or you can also use like `root@localhost`
   * `remoteDS`: The path to the remote dataset that shall be sent.
   * `snaps`: The amount of snapshots to keep locally. Remember: This only goes for the selected frequency.
      * `-1`: To keep an unlimited amount of snapshots (never delete them).
      * `0`: To delete all snapshots but still receive current/incremental data.
      * `1+`: The amount of snapshots to keep.
   * `reqFreeG`: Give notice when there's less space than the amount of gigabytes indicated on the pool, e.g. `200` would give a warning when the free space on the pool falls below 200G.
1. Setup a cron or systemd timer that will run the easysnapRecv script when you want to. Use as first parameter the frequency indicator, e.g. `0 * * * * /path/to/easysnap/easysnapRecv hourly`

## ToDo

- easysnapRecv
- pre-/post snapshot hooks for stuff like database locking and flushing
- systemd timer examples
