---
title: Persistence
description: How Valkey writes data to disk
---

Persistence refers to the writing of data to durable storage, such as a solid-state disk (SSD). Valkey provides a range of persistence options. These include:

* **RDB** (Valkey Database): RDB persistence performs point-in-time snapshots of your dataset at specified intervals.
* **AOF** (Append Only File): AOF persistence logs every write operation received by the server. These operations can then be replayed again at server startup, reconstructing the original dataset. Commands are logged using the same format as the Valkey protocol itself.
* **No persistence**: You can disable persistence completely. This is sometimes used when caching.
* **RDB + AOF**: You can also combine both AOF and RDB in the same instance.

To learn more about how to evaluate your Valkey persistence strategy, read on.

## RDB advantages

* RDB is a very compact single-file point-in-time representation of your Valkey data. RDB files are perfect for backups. For instance you may want to archive your RDB files every hour for the latest 24 hours, and to save an RDB snapshot every day for 30 days. This allows you to easily restore different versions of the data set in case of disasters.
* RDB is very good for disaster recovery, being a single compact file that can be transferred to far data centers, or onto Amazon S3 (possibly encrypted).
* RDB maximizes Valkey performances since the only work the Valkey parent process needs to do in order to persist is forking a child that will do all the rest. The parent process will never perform disk I/O or alike.
* RDB allows faster restarts with big datasets compared to AOF.
* On replicas, RDB supports [partial resynchronizations after restarts and failovers](replication.md#partial-resynchronizations-after-restarts-and-failovers).

## RDB disadvantages

* RDB is NOT good if you need to minimize the chance of data loss in case Valkey stops working (for example after a power outage). You can configure different *save points* where an RDB is produced (for instance after at least five minutes and 100 writes against the data set, you can have multiple save points). However you'll usually create an RDB snapshot every five minutes or more, so in case of Valkey stopping working without a correct shutdown for any reason you should be prepared to lose the latest minutes of data.
* RDB needs to fork() often in order to persist on disk using a child process. fork() can be time consuming if the dataset is big, and may result in Valkey stopping serving clients for some milliseconds or even for one second if the dataset is very big and the CPU performance is not great. AOF also needs to fork() but less frequently and you can tune how often you want to rewrite your logs without any trade-off on durability.

## AOF advantages

* Using AOF Valkey is much more durable: you can have different fsync policies: no fsync at all, fsync every second, fsync at every query. With the default policy of fsync every second, write performance is still great. fsync is performed using a background thread and the main thread will try hard to perform writes when no fsync is in progress, so you can only lose one second worth of writes.
* The AOF log is an append-only log, so there are no seeks, nor corruption problems if there is a power outage. Even if the log ends with a half-written command for some reason (disk full or other reasons) the valkey-check-aof tool is able to fix it easily.
* Valkey is able to automatically rewrite the AOF in background when it gets too big. The rewrite is completely safe as while Valkey continues appending to the old file, a completely new one is produced with the minimal set of operations needed to create the current data set, and once this second file is ready Valkey switches the two and starts appending to the new one.
* AOF contains a log of all the operations one after the other in an easy to understand and parse format. You can even easily export an AOF file. For instance even if you've accidentally flushed everything using the `FLUSHALL` command, as long as no rewrite of the log was performed in the meantime, you can still save your data set just by stopping the server, removing the latest command, and restarting Valkey again.

## AOF disadvantages

* AOF files are usually bigger than the equivalent RDB files for the same dataset.
* AOF can be slower than RDB depending on the exact fsync policy. In general with fsync set to *every second* performance is still very high, and with fsync disabled it should be exactly as fast as RDB even under high load. Still RDB is able to provide more guarantees about the maximum latency even in the case of a huge write load.


Ok, so what should I use?
---

The general indication you should use both persistence methods is if
you want a degree of data safety comparable to what PostgreSQL can provide you.

If you care a lot about your data, but still can live with a few minutes of
data loss in case of disasters, you can simply use RDB alone.

There are many users using AOF alone, but we discourage it since to have an
RDB snapshot from time to time is a great idea for doing database backups,
for faster restarts, and in the event of bugs in the AOF engine.

The following sections will illustrate a few more details about the two persistence models.

## Snapshotting

By default Valkey saves snapshots of the dataset on disk, in a binary
file called `dump.rdb`. You can configure Valkey to have it save the
dataset every N seconds if there are at least M changes in the dataset,
or you can manually call the `SAVE` or `BGSAVE` commands.

For example, this configuration will make Valkey automatically dump the
dataset to disk every 60 seconds if at least 1000 keys changed:

    save 60 1000

This strategy is known as _snapshotting_.

## No persistence

If you prefer **not** to have persistence (for example when using a Valkey instance solely as a cache) that is also a possibility.

RDB snapshotting is enabled by default. To disable it, set the `save` configuration parameter to the empty string `""` and remove any excess `save` lines that are present in the current configuration file.

    save ""

Alternatively, you can also use the `--save ""` argument on the `valkey-server` binary.

### How it works

Whenever Valkey needs to dump the dataset to disk, this is what happens:

* Valkey [forks](https://linux.die.net/man/2/fork). We now have a child
and a parent process.

* The child starts to write the dataset to a temporary RDB file.

* When the child is done writing the new RDB file, it replaces the old
one.

This method allows Valkey to benefit from copy-on-write semantics.

## Append-only file

Snapshotting is not very durable. If your computer running Valkey stops,
your power line fails, or you accidentally `kill -9` your instance, the
latest data written to Valkey will be lost.  While this may not be a big
deal for some applications, there are use cases for full durability, and
in these cases Valkey snapshotting alone is not a viable option.

The _append-only file_ is an alternative, fully-durable strategy for
Valkey.

You can turn on the AOF in your configuration file:

    appendonly yes

From now on, every time Valkey receives a command that changes the
dataset (e.g. `SET`) it will append it to the AOF.  When you restart
Valkey it will re-play the AOF to rebuild the state.

Valkey uses a multi part AOF mechanism.
That is, the original single AOF file is split into base file (at most one) and incremental files (there may be more than one).
The base file represents an initial (RDB or AOF format) snapshot of the data present when the AOF is [rewritten](#log-rewriting).
The incremental files contains incremental changes since the last base AOF file was created. All these files are put in a separate directory and are tracked by a manifest file.

### Log rewriting

The AOF gets bigger and bigger as write operations are
performed.  For example, if you are incrementing a counter 100 times,
you'll end up with a single key in your dataset containing the final
value, but 100 entries in your AOF. 99 of those entries are not needed
to rebuild the current state.

The rewrite is completely safe.
While Valkey continues appending to the old file,
a completely new one is produced with the minimal set of operations needed to create the current data set,
and once this second file is ready Valkey switches the two and starts appending to the new one.

Valkey supports an interesting feature: it is able to rebuild the AOF in the background without interrupting service to clients.
Whenever you issue a `BGREWRITEAOF`, Valkey will write the shortest sequence of commands needed to rebuild the current dataset in memory.
Valkey will automatically trigger log rewriting automatically (see the example configuration file for more information).

When an AOF rewrite is scheduled, the Valkey parent process opens a new incremental AOF file to continue writing.
The child process executes the rewrite logic and generates a new base AOF.
Valkey will use a temporary manifest file to track the newly generated base file and incremental file.
When they are ready, Valkey will perform an atomic replacement operation to make this temporary manifest file take effect.
In order to avoid the problem of creating many incremental files in case of repeated failures and retries of an AOF rewrite,
Valkey introduces an AOF rewrite limiting mechanism to ensure that failed AOF rewrites are retried at a slower and slower rate.

### How durable is the append only file?

You can configure how many times Valkey will
[`fsync`](https://linux.die.net/man/2/fsync) data on disk. There are
three options:

* `appendfsync always`: `fsync` every time new commands are appended to the AOF. Very very slow, very safe. Note that the commands are appended to the AOF after a batch of commands from multiple clients or a pipeline are executed, so it means a single write and a single fsync (before sending the replies).
* `appendfsync everysec`: `fsync` every second. Fast enough and you may lose 1 second of data if there is a disaster.
* `appendfsync no`: Never `fsync`, just put your data in the hands of the Operating System. The faster and less safe method. Normally Linux will flush data every 30 seconds with this configuration, but it's up to the kernel's exact tuning.

The suggested (and default) policy is to `fsync` every second. It is
both fast and relatively safe. The `always` policy is very slow in
practice, but it supports group commit, so if there are multiple parallel
writes Valkey will try to perform a single `fsync` operation.

### What should I do if my AOF gets truncated?

It is possible the server crashed while writing the AOF file, or the
volume where the AOF file is stored was full at the time of writing. When this happens the
AOF still contains consistent data representing a given point-in-time version
of the dataset (that may be old up to one second with the default AOF fsync
policy), but the last command in the AOF could be truncated.
The latest major versions of Valkey will be able to load the AOF anyway, just
discarding the last non well formed command in the file. In this case the
server will emit a log like the following:

```
* Reading RDB preamble from AOF file...
* Reading the remaining AOF tail...
# !!! Warning: short read while loading the AOF file !!!
# !!! Truncating the AOF at offset 439 !!!
# AOF loaded anyway because aof-load-truncated is enabled
```

You can change the default configuration to force Valkey to stop in such
cases if you want, but the default configuration is to continue regardless of
the fact the last command in the file is not well-formed, in order to guarantee
availability after a restart.

Older versions of Valkey may not recover, and may require the following steps:

* Make a backup copy of your AOF file.
* Fix the original file using the `valkey-check-aof` tool that ships with Valkey:

      $ valkey-check-aof --fix <filename>

* Optionally use `diff -u` to check what is the difference between two files.
* Restart the server with the fixed file.

### What should I do if my AOF gets corrupted?

If the AOF file is not just truncated, but corrupted with invalid byte
sequences in the middle, things are more complex. Valkey will complain
at startup and will abort:

```
* Reading the remaining AOF tail...
# Bad file format reading the append only file: make a backup of your AOF file, then use ./valkey-check-aof --fix <filename>
```

The best thing to do is to run the `valkey-check-aof` utility, initially without
the `--fix` option, then understand the problem, jump to the given
offset in the file, and see if it is possible to manually repair the file:
The AOF uses the same format of the Valkey protocol and is quite simple to fix
manually. Otherwise it is possible to let the utility fix the file for us, but
in that case all the AOF portion from the invalid part to the end of the
file may be discarded, leading to a massive amount of data loss if the
corruption happened to be in the initial part of the file.

### How it works

Log rewriting uses the same copy-on-write trick already in use for
snapshotting.  This is how it works:

**Valkey multi-part AOF**

* Valkey [forks](https://linux.die.net/man/2/fork), so now we have a child
and a parent process.

* The child starts writing the new base AOF in a temporary file.

* The parent opens a new increments AOF file to continue writing updates.
  If the rewriting fails, the old base and increment files (if there are any) plus this newly opened increment file represent the complete updated dataset,
  so we are safe.
  
* When the child is done rewriting the base file, the parent gets a signal,
and uses the newly opened increment file and child generated base file to build a temp manifest,
and persist it.

* Profit! Now Valkey does an atomic exchange of the manifest files so that the result of this AOF rewrite takes effect. Valkey also cleans up the old base file and any unused increment files.

### How I can switch to AOF, if I'm currently using dump.rdb snapshots?

If you want to enable AOF in a server that is currently using RDB snapshots, you need to convert the data by enabling AOF via CONFIG command on the live server first.

**IMPORTANT:** not following this procedure (e.g. just changing the config and restarting the server) can result in data loss!

Preparations:

* Make a backup of your latest dump.rdb file.
* Transfer this backup to a safe place.

Switch to AOF on live database:

* Enable AOF: `valkey-cli config set appendonly yes`
* Optionally disable RDB: `valkey-cli config set save ""`
* Make sure writes are appended to the append only file correctly.
* **IMPORTANT:** Update your `valkey.conf` (potentially through `CONFIG REWRITE`) and ensure that it matches the configuration above.
  If you forget this step, when you restart the server, the configuration changes will be lost and the server will start again with the old configuration, resulting in a loss of your data.

Next time you restart the server:

* Before restarting the server, wait for AOF rewrite to finish persisting the data.
  You can do that by watching `INFO persistence`, waiting for `aof_rewrite_in_progress` and `aof_rewrite_scheduled` to be `0`, and validating that `aof_last_bgrewrite_status` is `ok`.
* After restarting the server, check that your database contains the same number of keys it contained previously.

## Interactions between AOF and RDB persistence

Valkey makes sure to avoid triggering an AOF rewrite when an RDB
snapshotting operation is already in progress, or allowing a `BGSAVE` while the
AOF rewrite is in progress. This prevents two Valkey background processes
from doing heavy disk I/O at the same time.

When snapshotting is in progress and the user explicitly requests a log
rewrite operation using `BGREWRITEAOF` the server will reply with an OK
status code telling the user the operation is scheduled, and the rewrite
will start once the snapshotting is completed.

In the case both AOF and RDB persistence are enabled and Valkey restarts the
AOF file will be used to reconstruct the original dataset since it is
guaranteed to be the most complete.

## Backing up Valkey data

Before starting this section, make sure to read the following sentence: **Make Sure to Backup Your Database**. Disks break, instances in the cloud disappear, and so forth: no backups means huge risk of data disappearing into /dev/null.

Valkey is very data backup friendly since you can copy RDB files while the
database is running: the RDB is never modified once produced, and while it
gets produced it uses a temporary name and is renamed into its final destination
atomically using rename(2) only when the new snapshot is complete.

This means that copying the RDB file is completely safe while the server is
running. This is what we suggest:

* Create a cron job in your server creating hourly snapshots of the RDB file in one directory, and daily snapshots in a different directory.
* Every time the cron script runs, make sure to call the `find` command to make sure too old snapshots are deleted: for instance you can take hourly snapshots for the latest 48 hours, and daily snapshots for one or two months. Make sure to name the snapshots with date and time information.
* At least one time every day make sure to transfer an RDB snapshot *outside your data center* or at least *outside the physical machine* running your Valkey instance.

### Backing up AOF persistence

If you run a Valkey instance with only AOF persistence enabled, you can still perform backups.
AOF files are split into multiple files which reside in a single directory determined by the `appenddirname` configuration.
During normal operation all you need to do is copy/tar the files in this directory to achieve a backup. However, if this is done during a [rewrite](#log-rewriting), you might end up with an invalid backup.
To work around this you must disable AOF rewrites during the backup:

1. Turn off automatic rewrites with<br/>
   `CONFIG SET` `auto-aof-rewrite-percentage 0`<br/>
   Make sure you don't manually start a rewrite (using `BGREWRITEAOF`) during this time.
2. Check there's no current rewrite in progress using<br/>
   `INFO` `persistence`<br/>
   and verifying `aof_rewrite_in_progress` is 0. If it's 1, then you'll need to wait for the rewrite to complete.
3. Now you can safely copy the files in the `appenddirname` directory.
4. Re-enable rewrites when done:<br/>
   `CONFIG SET` `auto-aof-rewrite-percentage <prev-value>`

**Note:** If you want to minimize the time AOF rewrites are disabled you may create hard links to the files in `appenddirname` (in step 3 above) and then re-enable rewrites (step 4) after the hard links are created.
Now you can copy/tar the hardlinks and delete them when done. This works because Valkey guarantees that it
only appends to files in this directory, or completely replaces them if necessary, so the content should be
consistent at any given point in time.


**Note:** If you want to handle the case of the server being restarted during the backup and make sure no rewrite will automatically start after the restart you can change step 1 above to also persist the updated configuration via `CONFIG REWRITE`.
Just make sure to re-enable automatic rewrites when done (step 4) and persist it with another `CONFIG REWRITE`.

## Disaster recovery

Disaster recovery in the context of Valkey is basically the same story as
backups, plus the ability to transfer those backups in many different external
data centers. This way data is secured even in the case of some catastrophic
event affecting the main data center where Valkey is running and producing its
snapshots.

We'll review the most interesting disaster recovery techniques
that don't have too high costs.

* Amazon S3 and other similar services are a good way for implementing your disaster recovery system. Simply transfer your daily or hourly RDB snapshot to S3 in an encrypted form. You can encrypt your data using `gpg -c` (in symmetric encryption mode). Make sure to store your password in many different safe places (for instance give a copy to the most important people of your organization). It is recommended to use multiple storage services for improved data safety.
* Transfer your snapshots using SCP (part of SSH) to far servers. This is a fairly simple and safe route: get a small VPS in a place that is very far from you, install ssh there, and generate a ssh client key without passphrase, then add it in the `authorized_keys` file of your small VPS. You are ready to transfer backups in an automated fashion. Get at least two VPS in two different providers
for best results.

It is important to understand that this system can easily fail if not
implemented in the right way. At least, make absolutely sure that after the
transfer is completed you are able to verify the file size (that should match
the one of the file you copied) and possibly the SHA1 digest, if you are using
a VPS.

You also need some kind of independent alert system if the transfer of fresh
backups is not working for some reason.
