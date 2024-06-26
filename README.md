# Db2 Migration Tooling
The Db2 Migration Tooling (Db2MT) can be used to migrate Db2 Workloads to RDS for Db2. Db2MT eases the migration of existing workload from other EC2 or on prem systems by guiding the migration process and also by having key optimizations to speed things up. Db2MT also allows for transparency and customization during the migration process as well through the customization of the generated scripts.

## Prerequisites
### AIX
Ensure AIX Toolbox for Open Source Software is installed
https://www.ibm.com/support/pages/node/882892

```
lslpp -l rpm.rte
```

Install jq and sed from the Toolbox (Note: default sed that comes with AIX will not work with db2mt)
```
cd /opt/freeware/bin
dnf -y install jq
dnf -y install sed
```

Add to the PATH environment variable
```
export PATH=/opt/freeware/bin:$PATH >> ~/.profile
. ~/.profile
```

Please also ensure that /bin/bash is available on the system.

### Linux
Ensure that jq, sed, and /bin/bash are available on the system.  Most Linux systems should already have these installed.


## Commands
You can find more details for each command below with `--help`.

### db2mt init
`db2mt init` command generates a configuration template to `~/.db2mt/init.yaml`.  The template controls the parameters that will be used to execute the tool.  

You can either edit `~/.db2mt/init.yaml` directly or you can pass in the options as arguments to the command.

Example:

```
db2mt init --targetDatabaseName mytargetdb --targetDatabaseUser mytargetuser --targetDatabasePassword mytargetpasswd
```

### db2mt configure
`db2mt configure` should be run after `init.yaml` has been filled out and before starting data movement.

This command will perform various sanity checks, as we as configuring s3 bucket and other required components.

If you would like db2mt to create an s3 bucket for you, please provide `admawsAccessKey`, `admawsSecretKey`, `s3Region`, `s3User`, and `bucketName`.  (Note: this only works on Linux)

Otherwise, if you already have an existing bucket or if you are on AIX, please create a bucket first and provide `s3awsAccessKey`, `s3awsSecretKey`, `s3Region`, `bucketName` instead in `init.yaml`.

```
./db2mt configure --help
The configure subcommand will configure required components for db2mt that will be used in the migration

Usage:
  db2mt configure [flags]

Flags:
  -h, --help   help for configure
  -n, --n      Catalog the target node and rdsadmin db
  -t, --t      Catalog the target database
```

### db2mt compatibility
The tool supports both offline and online backups.  Use `db2mt compatibility backup` and `db2mt compatibility backup-online` commands to check if the source system is able to migrate using backups. 

### db2mt generate
`db2mt generate` command is used to generate bash scripts that will perform the requested action.

For example, `db2mt generate backup` will generate a script that performs offline backup.

Note: If you make any changes to `init.yaml` during the migration process, please ensure to re-generate the script before execution.

```
./db2mt generate --help
The generate subcommand will generate a collection of bash scripts (e.g. backup, export etc.)

Usage:
    db2mt generate [option]
Options:
    backup           Generate backup scripts for full offline backup
    backup-online    Generate backup scripts for full online backup
    restore          Generate restore scripts for full offline/online backup
    rollforward      Generate rollforward scripts for full online backup
    db2look          Generate db2look scripts that will be used to recreate the database
    export           Generate export scripts that will be used to recreate the database
    setupdb          Generate setupdb scripts that will be used to set up the database (e.g. storage groups, tablespaces, bufferpools, tables) on the target system
    load             Generate load scripts that will be used to populate the tables
    finalize         Generate finalize scripts that will be used to create the remaining database artifacts

Flags:
  -f, --f       Applicable to restore only: Generate restore scripts for full offline backup
  -n, --n       Applicable to restore only: Generate restore scripts for full online backup

```

### db2mt run
`db2mt run` is used to execute the generated scripts.

For example, you can run `db2mt run backup` to execute the backup script.

```
$ ./db2mt run --help
The run subcommand will execute the scripts generated (e.g. backup, export)

Usage:
    db2mt run [option]
Options:
    backup             Run backup scripts for full offline backup
    backup-online      Run backup scripts for full online backup
    restore            Run restore scripts for full offline/online backup
    rollforward        Run rollforward scripts for full online backup
    export             Run export scripts to extract data from tables
    setupdb            Run setupdb scripts to set up the database (e.g. storage groups, tablespaces, bufferpools, tables) on the target system
    load               Run load scripts to ingest data into tables
    finalize           Run finalize scripts to create the remaining database artifacts

```

### db2mt upload
`db2mt upload` command uploads backup images / archive logs / files generated by export to s3.

Note that you must run `db2mt configure` with the proper credendials first before attempting upload.

```
$ ./db2mt upload --help
The upload subcommand will upload database files (e.g. backup/archivelogs/export) from source system to S3

Usage:
   db2mt upload [option]
Options:
   backup          Upload backup image to S3
   archive         Upload archive logs to S3 that are need for rollforward
   export          Upload db2look and export related files to S3

Flags:
  -c, --c       Applicable to archive only: Upload archive logs to S3 and complete rollforward

```

### db2mt s3
`db2mt s3` command provides capability to interact with s3 buckets.  Unlike `db2mt upload`, this command allows you to choose the files you want to upload.  It is especially useful in export/load migration scenarios where you might need to re-run export on selected tables.

Note that you must run `db2mt configure` with the proper credendials first before attempting upload.

```
$ ./db2mt s3 --help
The s3 subcommand will provide built-in interface to download / upload / list files for S3

Usage:
   db2mt s3 [option]
Options:
   download        Download files from S3
   upload          Upload files to S3
   list            List files on S3

```

Examples:

Download `db2look_bufferpools.ddl` from `s3://mybucket/db2mtdir/ddl/db2look_bufferpools.ddl` to local directory `$HOME/db2mtdir`

```
db2mt s3 download --source "db2mtdir/ddl/db2look_bufferpools.ddl" --target "$HOME/db2mtdir"
```

Upload file `$HOME/db2mtdir/export-rerun/T1_export.ixf` to `s3://mybucket/db2mtdir/export/T1.export.ixf`
```
db2mt s3 upload --source "$HOME/db2mtdir/export-rerun" --target "db2mtdir/export" --pattern "T1_export.ixf"
```

If you have multiple files, you can use `"*"` in --pattern
```
db2mt s3 upload --source "$HOME/db2mtdir/export-rerun" --target "db2mtdir/export" --pattern "* --parallelism ${UPLOAD_NUM_THREADS}"
```

To list the content of an s3 bucket
```
db2mt s3 list
```

### db2mt cleanup
Once migration is done, you can clean up everything with `db2mt cleanup configure` command.

In `init.yaml`, there are options that allow you to select what you would like to be cleaned up.


## Export and Load Scenarios
If the source and target systems run on different platforms (e.g source on AIX and target on Linux), the recommended approach is to use export and load to migrate the database.

### Export
Set up the configuration template.  Review and adjust the values accordingly in `init.yaml`.  The relevant sections are `Database`, `Global Options`, `AWS Options`, `Db2look Options`, `Export Options`, and `Upload Options`.
```
db2mt init
```

Configure the instance.
```
db2mt configure
```

Extract DDL defition.  Review the files generated and make any modifications if needed.

Note: Default bufferpools and default tablespaces created by Db2 will be excluded in the DDLs
```
db2mt generate db2look
```

Start db2 export on the source database.

```
db2mt generate export
db2mt run export
```

While export is running, you may also start the upload process at the same time.
```
db2mt upload export
```

Review the log files.  Default location is `$HOME/db2mtdir/logs` unless specified differently in `init.yaml`.

After the export and upload processes are finished, if there are any failed tables, you may re-run export on the selected tables.


### Rerun export on selected tables
Investigate and address the failures.  Common errors include disk full, tables requiring different export modifiers, timeout etc.

Collect the list of failed tables and form a new list.
```
cat export_*_failed.lst >> $HOME/db2mtdir/ddl/current/new_master_tables.lst
```

Update `exportTablesList` and `exportDir` in `init.yaml`
```
db2mt init --exportTablesList "$HOME/db2mtdir/ddl/current/new_master_tables.lst" --exportDir "$HOME/db2mtdir/export-rerun"
```

Re-run export
```
db2mt generate export
db2mt run export
```

After export is done, upload the selected files to s3
```
db2mt s3 upload --source "$HOME/db2mtdir/export-rerun" --target "db2mtdir/export" --pattern "*"
```

### Load

Provision a target system to host the new database.  If you do not have direct access to the machine, you may also use a client machine to perform the data population steps below.

Set up the configuration template on the target or clinet machine.  Review and adjust the values accordingly in `init.yaml`.  The relevant sections are `Database`, `Global Options`, `AWS Options`, and `Load Options`.
```
db2mt init
```

Catalog the node and target database.
```
db2mt configure -n
db2mt configure -t
```

Configure the instance.
```
db2mt configure
```

Recall that we have generated various DDLs files with `db2mt generate db2look` in the previous stage.  If `ddlTransformation` is set `YES` (`YES` is the required for AWS RDS for Db2 migration), you will need to create the bufferpools, tablespaces, and roles manually first, as these objects require special admin access on the RDS system.  Ensure to change the database name in the files before running.  You can download the DDL files from S3:
```
db2mt s3 download --source "db2mtdir/ddl/db2look_bufferpools.ddl" --target "$HOME/db2mtdir"

db2mt s3 download --source "db2mtdir/ddl/db2look_tablespaces.ddl" --target "$HOME/db2mtdir"

db2mt s3 download --source "db2mtdir/ddl/db2look_roles.ddl" --target "$HOME/db2mtdir"
```

After you've created the objects above, proceed to create the tables.

Note: If your target is not on RDS i.e. `ddlTransformation` is `NO`, this step will create bufferpools, tablespaces, and roles automatically assuming the userid you've provided have sufficient privileges.

```
db2mt generate setupdb
db2mt run setupdb
```

If there are errors in creating the objects and you have determined that those errors can be ignored, you can add the SQL code in `ignoreFile` specified in `init.yaml`.  You can re-run setupdb until all objects are created.

SQL0601N is already populated in the ignoreFile by default, you may append additional SQL codes you wish to ignore to the file.  The ignoreFile is not only limited to the setupdb step, it's applicable to other steps in the scenario as well.

Example:
```
SQL0601N    #The name of the object to be created is identical to the existing object
SQL0602N    #
SQL0603N    #
```

After the tables have been created, you can start loading data into the target database.
```
db2mt generate load
db2mt run load
```

The last step is to create the remaining objects in the target database by running the finalize step.  This will also re-create the grants and permissions in the database.

```
db2mt generate finalize
db2mt run finalize
```

Note that if `ddlTransformation` is set `YES` (`YES` is the required for AWS RDS for Db2 migration), you will need to run `db2look_grant_dbadm.ddl` and `db2look_grant_role.ddl` manually, as these statements require special admin access on the RDS system.  Ensure to change the database name in the files before running.  You can download the DDL files from S3:
```
db2mt s3 download --source "db2mtdir/ddl/db2look_grant_dbadm.ddl" --target "$HOME/db2mtdir"

db2mt s3 download --source "db2mtdir/ddl/db2look_grant_role.ddl" --target "$HOME/db2mtdir"
```

### Rerun load on selected tables
Similar to export, if there are tables with failed load, you can re-run load on selected tables.

Collect the list of failed tables and form a new list.
```
cat load_*_failed.lst >> $HOME/db2mtdir/landing/current/new_master_tables.lst
```

Update `downloadLoadTablesList` and `loadTablesList` in `init.yaml`
```
db2mt init --downloadLoadTablesList "NO" --loadTablesList "$HOME/db2mtdir/landing/current/new_master_tables.lst"
```

Re-run load
```
db2mt generate load
db2mt run load
```


## Backup and Restore Scenarios
db2mt supports both offline and online backup.  It also supports backup to S3 and backup to local filesystem.

Check backup compatibility
```
db2mt compatibility backup
    or
db2mt compatibility backup-online
```

Set up the configuration template.  Review and adjust the values accordingly in `init.yaml`.  The relevant sections are `Database`, `Global Options`, `AWS Options`, and `Backup Options`.
```
db2mt init
```

Configure the instance
```
db2mt configure
```

### Offline backup
Run offline backup
```
db2mt generate backup
db2mt run backup
```

### Online backup
Run online backup
```
db2mt generate backup-online
db2mt run backup-online
```

### Upload local backup
For local backup, you may upload the backup images from local filesystem to S3
```
db2mt upload backup
```

### Upload archive logs
For online backup, you may also upload archive logs to S3
```
db2mt upload archive
```
To upload archive to s3 and complete the rollforward
```
db2mt upload archive -c
```

### Restore database on RDS
If the target database is on RDS, you may use the following command to run restore.

For restoring an offline backup
```
db2mt generate restore -f
db2mt run restore
```

For restoring an online backup
```
db2mt generate restore -n
db2mt run restore
```

### Rollforward logs for online backup on RDS
```
db2mt generate rollforward
db2mt run rollforward
```
