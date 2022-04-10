# PI2S3 (Backup to S3 Storage)

Create daily tar backups of your data to S3 or compatible storage.

Features:

* Weekly full backups
* Snapshots created daily
* Easy configuration
* Auto rotation to save S3 storage space


## Introduction

`pi2s3` is a script I wrote to handle creating off-site backups for a few Raspberry PI's & other Linux boxes I have. As it has helped to drastically reduce the trauma I have had on a few occasions in the past, I thought others may also find it useful.

`pi2s3` creates a tar backup of selected folders/files on a Linux system to a bucket on Amazon S3 storage or compatible services, using [s3cmd](https://s3tools.org/s3cmd) to send the data.

## How it works

A full backup is performed at the beginning of each week  (Monday) followed by differentials known as `snapshots` each day for the remaining days. `pi2s3` checks if a full backup has been performed that week regardless of the current day and if not one will be created.

### Snapshots

Snapshots copy only the files that have *changed* since the last *full* backup for that week. This method is used to reduce the amount of storage space required, and ensure no more than two operations are ever needed to restore everything.

### Rotation

By default backups are stored on a 4 week rotation with the oldest week being removed. The weekly rotation period can be altered in the configuration file.

### S3 Bucket folders

Backups are grouped in bucket folders by hostname to allow for multiple systems, then the Monday (YYYY-MM-DD) of the week they were performed. The daily backup archives within that folder are then named by the actual date and time they ran during that particular week.

The format of a S3 path looks like this:`[bucket-name]/backups/[hostname]/YYYY-MM-DD/`. As mentioned previously, the date part of the location is the date of the **Monday** of the week the backup was performed.

## Installing pi2s3

```bash
git clone --depth 1 https://github.com/neutralvibes/pi2s3.git
cd pi2s3
cp config/samples/* config/
cp scripts/samples/* scripts/
chmod +x pi2s3 scripts/*
```

### Configuring pi2s3

Edit `config/backup.cfg` and provide the name of the bucket that will be receiving your backup data. It is important to look through the file as there are other options such as `KEEP_BACKUP_WEEKS` to handle rotation, or `CHECK_ROOT` to force root only execution.

### Get S3cmd

*If you are already using `s3cmd` you can skip this step, although you may still need to create a configuration and/or bucket for your backups if required.*

[s3cmd](https://s3tools.org/s3cmd) is a python utility widely used to write to and from S3 storage. `pi2s3` uses the `s3cmd` utility to communicate with your chosen S3 compatible storage provider.

Let's install it.

```bash
sudo pip install pip --upgrade
sudo pip installl s3cmd
```

Once `s3cmd` has been installed you need to configure it.

```bash
s3cmd --configure
```

Further information about the `s3cmd` tool can be obtained from https://s3tools.org/s3cmd and https://github.com/s3tools/s3cmd.

*Note if your storage provider does not support dns buckets hit [space] + [enter] on this option*

Once you have configured s3cmd if the bucket you wish to save the data into *does not exist* you need to create it as `pi2s3` needs a bucket that already exists.

```bash
s3cmd mb s3://new-bucket-name-here
```

### Files to be archived

Edit `config/include_list.txt` which is where you place the paths to names of the folders/files to be archived. You can remove or add to the entries already there. Each entry should be *one* per line.

### Excluding files/folders

Files/folders you wish to be excluded should be placed one per line into `config/exclude_list.txt`.

### Running a backup

```bash
.\pi2s3 run
```

Once configured that's all you need to do. Of course once you have it working they way you like scheduling a cron job would probably be the next thing.

### Tasks before & after backup

Sometimes you need to prepare a system before a backup can run reliably. This could involve dumping a database, stopping/restarting services or any other such task.

Two files in the `scripts` folder called `before_backup` and `after_backup` can be modified to perform the tasks you require before and after a backup.

## Listing archives

You can of course use the `s3cmd` tool, but for convenience `pi2s3` can save you typing in the full paths.

* To list the week folders created by the backup.

```bash
.\pi2s3 ls
```

* To list a single week folder created by the backup.

```bash
.\pi2s3 ls 2022-03-14
```

## Restoring files

Extracting files/folders from the backup without first copying the archive is possible.

Here is a base example you can alter to your needs.

```bash
s3cmd --no-progress get s3://bucket-name/backups/hostname/2022-04-04/2022-04-05_1403_backup_full.tar.gz - | sudo tar -xzpv folder-to-get1 folder-to-get2
```
