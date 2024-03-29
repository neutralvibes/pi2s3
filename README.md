# PI2S3 (Backup to S3 Storage)

Create regular compressed tar backups of your data to S3 or compatible storage.

Features:

* Full backups weekly
* Snapshots created daily to save space
* Easy configuration
* Auto weekly rotation to save S3 storage space
* Creates saved file listing along with backup file
* Stored backup warnings along with backup file

## Introduction

`pi2s3` is a script I wrote to handle creating off-site backups for a few Raspberry PI's, other Linux based systems, and some Virtual Private Servers (VPS) to S3 storage providers, including but not limited to Amazon's.

I have found using `pi2s3` gives me more flexibility, while also helping to drastically reduce trauma on a few occasions in the past, and that  others may also find it useful.

> With some S3 providers offering upwards of 250GB storage for the price of a couple of bars of chocolate a month, including unlimited in/outbound traffic, the barriers for entry are now lower.

`pi2s3` creates a compressed tar backup of selected folders/files on a Linux system to a bucket on Amazon S3 storage or compatible services, using battle hardened [s3cmd](https://s3tools.org/s3cmd) to send the data.

As with any backup solution, it is always advisable to test, and check regularly that it is performing as expected.

## How it works

A full backup is performed at the beginning of each week  (Monday) followed by differentials known as `snapshots` each day for the remaining days. `pi2s3` checks if a full backup has been performed that week **regardless** of the current day and if not one will be created instead of a snapshot.

It is totally up to you how often you decide to perform backups, daily, once a week, every few days, scheduled via `cron` or manually.

### Snapshots

Snapshots copy only the files that have *changed* since the last *full* backup for that week. This method is used to reduce the amount of storage space required, and ensure no more than two operations are ever needed to restore everything.

### Rotation

By default backups are stored on a 4 week rotation with the oldest week being removed. The weekly rotation period can be altered in the configuration file.

### S3 Bucket folders

Backups are grouped in bucket folders by hostname to allow for multiple systems, then the Monday (YYYY-MM-DD) of the week they were performed. The daily backup archives within that folder are then named by the actual date and time they ran during that particular week.

The format of a S3 path looks like this:`[bucket-name]/backups/[hostname]/YYYY-MM-DD/`. As mentioned previously, the date part of the location is the date of the **Monday** of the week the backup was performed.

## Get S3cmd

Before using `pi2s3` we need a working installation of `s3cmd`.

*If you are already using `s3cmd` you can skip this step, although you may still need to create a configuration and/or bucket for your backups if required.*

[s3cmd](https://s3tools.org/s3cmd) is a python utility widely used to write to and from S3 storage. `pi2s3` uses the `s3cmd` utility to communicate with your chosen S3 compatible storage provider.

Installation instructions [here](https://github.com/s3tools/s3cmd/blob/master/INSTALL.md)

```bash
s3cmd --configure
```


Further information about the `s3cmd` tool can be obtained from https://s3tools.org/s3cmd and https://github.com/s3tools/s3cmd.

*Note if your storage provider does not support dns buckets hit [space] + [enter] on this option*

### Create a bucket

Once you have configured s3cmd to use your storage provider, if the bucket you wish to save the data into *does not exist* you need to create one, as `pi2s3` needs a bucket already in existence.

```bash
s3cmd mb s3://new-bucket-name-here

# See it by
s3cmd ls s3://new-bucket-name-here

```

### Important Note

One thing to be aware of  is that `s3md` stores its configuration files in the users home folder `~\.s3cfg`. If/when using `pi2s3` as root, which is likely the case, you should copy this file to `/root`


## Installing pi2s3

Get the files
```bash
git clone --depth 1 https://github.com/neutralvibes/pi2s3.git
cd pi2s3
cp config/samples/* config/
cp scripts/samples/* scripts/
chmod +x pi2s3 scripts/*
```

Edit `config/backup.cfg` which is the configuration file and provide the name of the bucket that will be receiving your backup data. 

It is important to look through this file as there are other options such as `KEEP_BACKUP_WEEKS` to handle rotation, or `CHECK_ROOT` to force root only execution.

### Choosing folders/files to be archived

Edit `config/include_list.txt` which is where you place the paths to names of the folders/files to be archived. You can remove or add to the entries already there. Each entry should be *one* per line.

### Excluding files/folders

Files/folders you wish to be excluded should be placed one per line into `config/exclude_list.txt`.

### Command line options

Typing `./pi2s3` will show you the options available.

```bash

./pi2s3

# Prints
Usage: pi2s3 [option]

Create tar backups to S3 storage

    info        Config info
    run         Run backup
    ls          List backup weeks
    ls  [week]  List contents of week folder
```

Typing `./pi2s3 info` will show you the settings you have chosen.

### Running a backup

```bash
.\pi2s3 run
```

Once configured that's all you need to do. Of course once you have it working the way you like, scheduling a `cron` job may well be the next thing.

### Tasks before & after backup

>* scripts/before_backup
>* scripts/after_backup

Sometimes you need to prepare a system before a backup can run reliably. This could involve dumping a database, stopping/restarting services, sending [pushover notifications](https://github.com/neutralvibes/pushover) of start/completion status, or any other such task.

The two files are located in the `scripts` folder, named `before_backup` and `after_backup`. These can be modified to perform the tasks you require before and after each backup.

### Listing archives

You can of course use the `s3cmd` tool, but for convenience `pi2s3` can save you typing in the full paths.

* To list the week folders created by the backup.

```bash
.\pi2s3 ls
```

* To list a single week folder created by the backup. You would use `pi2s3 ls [YYYY-MM-DD]`, the date being the *Monday* of the week in question.

```bash
.\pi2s3 ls 2022-04-11

# Prints
# I have removed the date and size columns but they are shown.
s3://bucket-name/backups/hostname/2022-04-11/2022-04-11_1104_backup_full.apt_installed.lst.gz
s3://bucket-name/backups/hostname/2022-04-11/2022-04-11_1104_backup_full.tar.gz
s3://bucket-name/backups/hostname/2022-04-11/2022-04-11_1104_backup_full.lst.gz
s3://bucket-name/backups/hostname/2022-04-11/2022-04-11_1104_backup_full.warnings.log.gz
s3://bucket-name/backups/hostname/2022-04-11/2022-04-11_1104_bck.log
s3://bucket-name/backups/hostname/2022-04-11/backup.snar
s3://bucket-name/backups/hostname/2022-04-11/backup.snar-1
```

As a week of backups progresses, snapshot files and their listings will also be added to this view.

>You may notice a file ending with `.apt_installed.lst` in my example. This a list of items installed by `apt` on the system at the time. Enabling/disabling the creation of this file is available in the config file.

The full path to these files can of course be viewed by using `s3cmd ls s3://full-path`


### Restoring  from backups

>To perform any type of file/folder recovery to another host, you only need to install `s3cmd` and configure it to point to your S3 provider first.

A lot of what is possible depends on your backup schedules. It is entirely your choice if you perform backups daily, once a week, or every few days. Daily backups provide the most protection but require more storage, although `pi2s3` use of `tar` snapshots can significantly reduce these demands.

#### About full restores

As mentioned previously, one full backup is performed weekly with snapshots made for the other days. Any snapshots contain only *changes* from the last *full* backup. So to restore *everything* archived up-to the date in question if archiving daily, there would be no more than *two* restore operations required. Restoring FIRST from the full backup for the week in question, followed by the daily snapshot of the day required within that week. If your backup schedule is only once a week, you can only restore from that full backup.

#### Individual folders/files

If you want to recover files/folders that have been changed or removed, sometimes it may be possible to retrieve them from just a snapshot, particularly although not exclusively if they had been created on a day after the full backup for that period. Otherwise the full backup may be the one to use. Either way, the archive listings automatically created can make finding what you want easier.

#### Archive listings

To aid in any restore, all the folders/files that have been archived are listed conveniently in separate dated files alongside each backup and typically have an extension `tar.gz.lst`. It is much quicker viewing these listings when deciding which versions to retrieve, than having to create these listing from the tar archive at a later date before being able to examine them. I have found the relatively small space these files take up usually worth creating them ahead of time.


##### How to view listings

An example of copying an archive listing to the local machine for viewing.

```bash
s3cmd get s3://bucket-name/backups/hostname/2022-04-04/2022-04-05_1403_backup_full.lst.gz
gzip -d 2022-04-05_1403_backup_full.lst.gz

# Creates: 2022-04-05_1403_backup_full.lst
```

You can then use `cat`, `less` or `grep` to view it, or other relevant operations to determine if a particular archive contains what you require.

An example of viewing without storing a local copy.

```bash
s3cmd --no-progress get s3://bucket-name/backups/hostname/2022-04-04/2022-04-05_1403_backup_full.lst.gz - | zcat
```

You could exchange the `cat` in the example for `less` or a `grep` etc. If you may need to perform multiple operations such as searches or views then copying the file locally as mentioned previously would be better.


#### Extracting files/folder from a backup

Extracting files/folders from a backup without first making a copy of the archive locally is both possible and often desirable.

Here is a base example you can alter to your needs.

```bash
s3cmd --no-progress get s3://bucket-name/backups/hostname/2022-04-04/2022-04-05_1403_backup_full.tar.gz - | sudo tar -xzpv folder-to-get1 folder-to-get2
```

The `folder-to-get1 folder-to-get2` are space separated examples of paths to files and/or folders you wish to retrieve from an archive *specified on the left hand side of the line*.

The `--no-progess` flag is important here, to ensure that the `s3cmd` progress notifications don't corrupt the file.

## Disclaimer

You are free to use this code however you see fit, and FULLY accept that in doing so it is completely at your OWN risk. Further more, your use of this code in part or whole, fully accepts absolving this author of any liability whatsoever.

## Credits

I would like to thank my cat, but I don't have one. So, the various failed devices, people who take the time to write tutorials/code, support forums, develop SBC's etc.. internet wide, all deserve my praise.
