# easysnap

A little zfs snapshot tool including backup script. The tool consists of two scripts:

* easysnap: It will zfs snapshots and remove old ones if necessary. No configuration file needed, info is in custom dataset properties.
* easysnapRecv: It will allow to pull snapshots from other servers. A configuration file is needed.

## easysnap

easysnap is a simple bash script which will take snapshots of designated datasets by adding a user property `esaysnap:xxx` with a value. Changing the value will affect how easysnap will take snapshots on said dataset.

### How to use

1. You need to a custom user property to the desired dataset. You can do this by `zfs set easysnap:hourly="240:200" pool/path/to/dataset`, where
   * `easysnap:hourly` is the property name. The mandatory part is the `easysnap` one and the custom one is the `hourly` one. Instead of `hourly` you can provide any string, but common ones would be `frequent`, `hourly`, `daily`, `weekly`, `monthly` because they can easily be scripted to run by cron or systemd timers. But as said, you can provide any optional string.
   * `240:200` indicates that it should keep 240 snapshots and give a notices when there's less than 200GB free space on the pool. The free space notice is optional, so you could also just use `200`. Instead for a positive number of snapshots you could also use `-1` (unlimited snapshots) or `0` (remove all easysnap:xxx snapshots) or set it to `false` (don't make any snapshots/changes).
   * __Notice:__ easysnap removes/adjusts only the number of snapshots provided for this run. E.g. if you run the `./easysnap hourly` routine then it will only remove snapshots containing the string `èasysnap-hourly` in the snapshot name. No other snapshots are touched. This is intentional as you might want to make custom snapshots at some points and those should not be automatically removed.
1. When you have setup at least one dataset, you will need to add a cronjob or systemd timer to actually call the easysnap tool with the jobname that it should run. E.g. if you setup `easysnap:hourly` you could add a cron like `0 * * * * /path/to/easysnap hourly`. The parameter provided to the easysnap script (in this case `hourly`) must match the desired user property in the dataset `easysnap:hourly`. You can also run the script manually whenever you want, just call it like `./easysnap hourly`. In the [cron.example](cron.example) file you have examples for the common snapshot intervals.



## esaysnapRecv

## ToDo

- easysnapRecv
- pre-/post snapshot hooks for stuff like database locking and flushing
- systemd timer examples
