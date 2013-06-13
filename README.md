mysql-backup
============

Utilities for consistent MySQL Backups &amp; Archives (including Slave-Backups) with mydumper. 

Features
--------

* Backups of local or remote MySQL hosts from a dedicated server
* Consistent Slave-Backups (saves binlog & binlog-position)
* Rotation of backups
* Keeps one backup per month as monthly snapshot
* Archiving function (e.g. for use with BackupPCs "Archive" function to store your MySQL Backups alongside the normal backups on a dedicated disk)

Installation
============

Prerequisites
-------------

* mydumper / myloader
* mysql-client
* mysqldump

For infos on the installation of **mydumper**, please check this [article](http://fabianpeter.de/backup/mysql-consistent-slave-backup-with-mydumper-myloader/).

Install
-------

* Download / Clone the files.
* Copy binaries (under **usr/local/bin**) to a directory you desire
* Copy config-files (under **etc/mysql-backup**) to /etc/mysql-backup

How to use
==========

Configuration
-------------
Edit the main global configuration in **/etc/mysql-backup/mysql-backup-global.conf** and set your paths and default values.

* $servers: Static list of servers to be backed up. Won't work with any other script than **mysql-backup-daily** since the others are too specific.
* $storagepath: where to store your backups. Directory structure will be **$storagepath/files/$year/$month/$day/$server**
* $configpath: Path to the per-host configs
* $params: special parameters for mysqldump
* rotation_period_default: Default rotation period of backups (i.e. how long backups will be kept) if no specific period is set in host config
* default_archive_path: If no archive_path is set in per-host config, this one will be used when it comes to archiving.

Clone the per-host configuration files from **dummy.conf** under **/etc/mysql-backup/hosts**.

* $hostname: FQDN or IP address of the host to be backupped. Will also be the directory-name.
* $user: User to connect to the MySQL server
* $password: Password for this user
* $exclude_tables: Useful for mysqldump, since it sometimes runs into bugs if tables like "general_log" are missing; currently has no influence on the tables backuped by mydumper
* $exclude_databases: Same here.
* $rotation_period: Number of days to keep a backup
* $threads: Number of parallel threads for mydumper. Should not be more than the number of CPU cores you have. If you run into I/O problems, reduce this number.
* $chunk_size: mydumper can split tables into chunks (useful when using more than 1 thread). Define the size in lines per chunk.
* $archive_path: In case you do periodically archives of your backups to a dedicated storage, this is where the archive for this host would go.

Backup-User
-----------
You can use these SQL commands to create a MySQL user on your backupped hosts with enough permission to do the backup and also get slave-information when needed.

```sql
CREATE USER 'mysql-backup'@'%' IDENTIFIED BY 'PASSWORD';
GRANT SELECT, RELOAD, SHOW DATABASES, LOCK TABLES, REPLICATION CLIENT, SHOW VIEW ON *.* TO 'mysql-backup'@'%';
```

Make a backup
-------------
```bash
mysql-backup-daily HOSTNAME
```

Clean-Up after backup
---------------------
```bash
mysql-backup-clear
```

Move latest backup of all hosts to an archive directory
-------------------------------------------------------
```bash
mysql-backup-clear-archive
```

Import backup
-------------
```bash
mysql -u USER -p -h HOSTNAME < `cat $backup_dir/schemas/*`
myloader -d $backup_dir -q 10000 -o -h HOSTNAME -u USERE -p PASSWORD -v 3 -C -t 4

How it works
============
This toolchain relies on two main components:

mysqldump
---------
**mysqldump** only dumps the table schemas to a special directory within the backup path since **mydumper** isn't able to do this.

mydumper
--------
**mydumper** is a high-performance tool for backups of standalone or slave MySQL servers. This component does the backup itself after **mysqldump** dumped the database schema(s).

Toolchain
=========

mysql-backup-daily
------------------
This is the script that does the actual backup. By now (2013-06-07) the script does the following
* Check if a backup is already running; exit if at least 3 backups are running
* Load global config for storage-path and defaults
* Check if a servername has been given as attribute in the script call; if yes, just backup this server; if not, backup all servers listed in global config
* Load the config file of the host to backup; exit if it doesn't exist
* Check if server is a MySQL slave; if yes, verify that delay is 0 - wait up to 60 secs for the delay to become 0 before exiting
* Check if the backup-dir exists; create if not
* Get all databases and dump their schemas to '''BACKUPDIR/schemas'''; create dir if it doesn't exist; exclude databases listed in host-config
* Dump the data including the binlog infos with '''mydumper'''

mysql-backup-clear
------------------
This is the script fired after a backup
* Load global config for storage-path and defaults
* Check if a servername has been given as attribute in the script call; if yes, just backup this server; if not, backup all servers listed in global config
* Load the config file of the host to backup; exit if it doesn't exist
* Launch **mysql-backup-rotate** for rotation, monthly archives and clearing of empty directories

mysql-backup-rotate
-------------------
* Load global config for storage-path and defaults
* Check if a servername has been given as attribute in the script call; if yes, just backup this server; if not, backup all servers listed in global config
* Load the config file of the host to backup; exit if it doesn't exist
* Check if there's already a monthly archive (we keep one of the backups of each host every month as a monthly snapshot); if yes, check if the folder is empty; If so, or if no archive exists, a new one has to be created (next step). Set a flag to indicate that we need to archive one of our backups
* Here's the cleanup: Gather all existing backups for the host and check if one or more of them is older then the host's or the global '''rotation_period''' (usually 7 days). If yes, delete it. Also, copy one of the backups to the archive directory, if no archive exists yet.
* Find empty directories below **$storagepath** and delete them. This will of course first delete the deepest subdirectories so it may take two or more runs of the script to have a full backup directory removed when it's empty. Since the directory clearing isn't limited to a specific host, this happens with every call of this script.

mysql-backup-list
-----------------
* Load global config for storage-path and defaults
* Check if a servername has been given as attribute in the script call; if yes, just backup this server; if not, backup all servers listed in global config
* Load the config file of the host to backup; exit if it doesn't exist
* Find all existing backups of the given host and list them along with their sizes

mysql-backup-to-archive
-----------------------
This script moves the latest backup of a host to a given archive-directory (or to the default directory, if nothing else is given). This script is fired by ***mysql-backup-clear-archive***
* Load global config for storage-path and defaults
* Check if a servername has been given as attribute in the script call; if yes, just backup this server; if not, backup all servers listed in global config
* Load the config file of the host to backup; exit if it doesn't exist
* Find all existing backups of the given host and list them along with their sizes; copy the latest backup to the archive-directory.

mysql-backup-clear-archive
--------------------------
This script fires **mysql-backup-to-archive**; 
* Load global config for storage-path and defaults
* Check if a servername has been given as attribute in the script call; if yes, just backup this server; if not, backup all servers listed in global config
* Load the config file of the host to backup; exit if it doesn't exist
* Fire **mysql-backup-to-archive** for the host

