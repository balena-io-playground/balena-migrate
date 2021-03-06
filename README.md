# BalenaOS Migration

**Please Note:** This Repository contains code that successfully migrated RPI and X86 devices to balena but as of April 2019 
it is not being actively maintained any more. 

If you would like to migrate devices to balena please contact support on https://forums.balena.io/ . The balena 
team will be happy to provide a migration config and instructions for your devices.    
 
This project attempts to provide a generic solution to migrate a range of different device types running linux operating
systems to balenaOS.

## How to use balena-migrate

**Warning:**

* During the migration the primary storage of your device will be overwritten and all data that has not been saved prior to migration will be lost.
* When migrating devices, that contain critical data or are not easily accessible, please make sure to test your setup thoroughly in a test environment before applying it to production devices.

It makes sense to read this document completely to understand the concepts and the risks involved.

### Preparing the Migration Environment

The *migratecfg* folder in this repository will be your migration environment.

Please note that all files referenced in the configuration and used during migration and are expected to be found in the migratecfg directory and are referenced by pathnames relative to this directory.

To migrate a device you will have to copy some more files into this directory and create / edit configuration files. Therefore it makes sense to copy the entire folder to a different location. Ultimately - once your setup is complete you will copy the directory to the device you want to migrate.

After copying the migration environment the next step is to create a balena-migrate.conf file, that will contain your configuration. There is a sample config file contained in the directory that you can use as a template. The
easiest way is to copy that file to *balena-migrate.conf*. To migrate a Raspberry PI device you might invoke:

``` bash
 cp balena-migrate-sample.conf balena-migrate.conf
```

#### Preparing the OS Image

Next copy the balenaOS image that you want to install to the migratecfg folder and set the ```IMAGE_FILE``` variable in the config file to the name of the image.

When migrating Raspberry PI devices you can use the unmodified image, that you have downloaded from the dashboard.

On intel-based devices you will have downloaded a flasher image that can not be used directly with *balena-migrate*. Please read the next section 'Extracting the balenaOS image and grub config from a Flasher Image' to extract your
balenaOS image and grub config file.

The image you downloaded is typically zip compressed. Internally balena-migrate works with the gzip format to be able to flash images directly from the compressed archive. For this reason balena-migrate will unzip zip-compressed image files and recompresss them using gzip. If you are planning to use the same setup multiple times it makes sense to convert the image file to gzip manually to save time and disk space when devices are actually migrated. To do this use unzip to unpack the zip archive containing the image and then gzip to compress the unpacked file. You can then remove the zip archive and configure the gzipped file to be your image file.

Example:

``` bash
unzip balena-cloud-appname-paspberrypi3-2.26.0+rev1-dev-v8.0.0.img.zip
gzip balena-cloud-appname-paspberrypi3-2.26.0+rev1-dev-v8.0.0.img
rm balena-cloud-appname-paspberrypi3-2.26.0+rev1-dev-v8.0.0.img.zip
```

##### Extracting the balenaOS image and grub config from a Flasher Image

The easiest way to extract the balenaOS image is to use the extract.sh script supplied in the util folder 
of this repository.
The extract script can be invoked as follows:

``` bash
sudo <path to repo>/util/extract.sh --img <image destination> <flasher image>
```

Example:

``` bash
sudo extract  --img resin-image-genericx86-64.resinos.img.gz \
               balena-cloud-appname-intel-nuc-2.26.0+rev1-dev-v8.0.0.img.zip
```

The above command will extract the gzipped image from a zip archive containing the flasher image.

The extract script can work with a zip archive or with the raw flasher image.

**Warning:** If working on a zip archive please make sure you have about 3GB of disk space available to unpack the image.

The extract.sh script can also be used to extract grub configuration or boot images from the balena flasher image. Please call ```extract.sh -h``` for a complete list of options.


##### Setting up the Config File

Using the a text editor like nano, vim or gedit, edit your balena-migrate.conf file to contain at least the following
values:

``` bash
IMAGE_NAME=<name of your zip or gzip image file>
```

Several other options can be set that are describe in section 'Configuration'.

### Starting Migration

Once prepared the migration environment can be transferred to the target device. Migration is started by invoking
balena-migrate as su from inside the migration environment.

Example:

``` bash
sudo ./balena-migrate
```

### Using the migdb scripts

If you are migrating a number of similar devices, you might want to use the migdb-scripts that are provided in the util 
directory of this repository.

Please note that for the migdb scripts to work, the following conditions need to apply:

* migdb-migrate will use the balena-cli to pre-register devices. The balena-cli needs to be installed on the host running migdb-migrate. The balena-cli either needs to be logged in with an appropriate user or a valid api token or login credentials need to be provided using the ```MIG_BALENA_TOKEN``` option.
* migdb-migrate needs to be able to establish a ssh connection to the target devices.
* Using a password to establish the ssh connection is not encouraged. Your password might be stored unencrypted in config or log files. Please use public key authentification instead. 
* If you do use password authentification,  **migdb-migrate** will attempt to use the **sshpass** utility that needs to be installed prior to using **migdb-migrate** with the --passwd option.
* The user you use to establish the ssh session needs to be able to invoke ```sudo``` **without password** on the target device. This can be achieved by adding an entry in the target devices ```/etc/sudoers``` file.

The following migdb scripts are available in the util directory:

* **migdb-add-unit** will submit a device for migration.
* **migdb-migrate** is the worker script that copies the migrate configuration to devices and executes the migrate script on the device. This script is meant to run continuously while migrating devices. It can be started in multiple instances to migrate devices in parallel.
* **migdb-check-done** is the worker script that checks migrated devices to see if they come online in the balena backend. This script is meant to run continuously while migrating devices. It can be started in multiple instances to migrate devices in parallel.  
* **migdb-stats** Scans all files in the **done** and **fail** folder of the migdb database and creates a report in CSV format.
* **migdb-stop** will gracefully stop all running migdb-migrate or migdb-check-done scripts.

The scripts use a migrate config that you provide and apply it to an number of devices. A directory structure is used to submit devices for migration and track the state of the migration process. The scripts will pre register a device in the balena dashboard for every unit file submitted and then attempt to migrate the unit.
The balena cli is used to register devices and track the progress. The balena cli has to be installed and logged in on the computer that runs the migdb scripts.

The migrated devices will come up in balena with the device uuid that was created during pre registration. This way the success of every migration can be tracked end to end.
  
The **migdb-migrate** script attempts to connect to the device using ssh. The ssh connection is then used to transmit the migratecfg (migration environment). When the environment has been transmitted successfully the migdb-migrate script will register the device, generate and transmit a config.json file and will then call **balena-migrate** on the device to start the actual migration.

Options and parameters given in **migdb-add-unit** are mostly unit specific. They will be written to the unit file and will override any parameters given to **migdb-migrate** on the command line or in environment variables. The (unit specific) information necessary to create a ssh connection needs to be provided when creating a unit file.

Example:

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;```migdb-add-unit --host 192.168.1.15 --user pi --passwd secret my-unit-id```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Will create a unit file with the given ssh credentials and will attempt to migrate that host.

The **unit-id** is a unique identifier for the device. It can be used to track the unit during migration by looking for files tagged with the unit-id. Reusing unit-ids will lead to file name clashes and failure of the migdb processes.

Non unit specfic parameters can be supplied as defaults when starting **migdb-migrate**. Options given to **migdb-migrate** are defaults that will be used for all units if not overridden by unit files.

Eg. if all devices use the same user and config directory, starting **migdb-migrate** as follows will supply a default user and configuration:

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;```migdb-migrate --user=pi --mig-cfg-dir=./migratecfg```


The scripts maintain a directory structure, that contains folders for different device states, logs and tmp files:

* **db/units** - this is where **migdb-add-unit** submits files. **migdb-migrate** will scan this directory and remove files as they are being processed.
* **db/process** - unit files in this directory have been migrated but have not been seen online yet.
* **db/done** - unit files in this directory have been migrated successfully and are online.
* **db/fail** - when processing fails unit files are moved to this directory. The file is appended with an error message,a timestamp and the path of the process log.
* **db/tmp** - unit files are being moved here while they are being processed. If the process script terminates unexpectedly you will typically find leftover files in this directory.
* **db/log** - log files

The migdb scripts can be configured using using command line parameters or environment variables:

#### MIG_APP

Used by: **migdb-add-unit**, **migdb-migrate**  

Command Line Option: ```--app <application>```

The balena application to register the device to. The **migdb-migrate** script will attempt to pre register a device with balena and retrieve a uuid and a config file (config.json) for the device. This way  the device can be tracked. The device UUID will be saved in the device unit file.

If this parameter is present in **migdb-add-unit**, it will be written to the unit file and override the same option given elsewhere.

#### MIG_BALENA_TOKEN

Used by: **migdb-migrate**

Command Line Option: ```--balena-token <api-token>```

At startup the ```migdb-migrate``` command will check if a user is logged in to the balena cli and fail if that is not the case.

If ```MIG_BALENA_TOKEN``` is set, the script will attempt to log in a user using ```balena login -t <api-token>``` and only fail if login does not succeed.

Example:

```MIG_BALENA_VER="secretapitokenvalue"```

#### MIG_BALENA_VER

Used by: **migdb-add-unit**, **migdb-migrate**

Command Line Option: ```--balena-ver <version>```

The balena version that will be installed on the device. This is needed for preregistering the device in Balena.

Example:

```MIG_BALENA_VER="2.26.0"```

#### MIG_CFG

Used by: **migdb-add-unit**, **migdb-migrate**, **migdb-check-done**

Command Line Option: ```--cfg <config file>```

Path to a configuration file. The script will read the file (in bash syntax) and use the variables defined in it. It can contain any of the variables listed here and will override values given on the command line or in the environment.

#### MIG_CFG_ARCHIVE

Used by: **migdb-add-unit**, **migdb-migrate**

Command Line Option: ```--cfg-tgz <path to tar archive>```

The path to a compressed tar archive containing the migratecfg folder.

The migdb utilities expect this archive to contain only the contents of the migratecfg folder, not the folder itself.

#### MIG_CFG_DIR

Used by: **migdb-add-unit**, **migdb-migrate**

Command Line Option: ```--cfg-dir <path to migratecfg>```

The path the migratecfg folder.

#### MIG_CONN_ATTEMPTS

Used by: **migdb-migrate**

Command Line Option: ```--conn-attempts <connect-attempts>```

How often migdb-migrate will attempt to initiate a ssh connection before migration fails.

The parameter ```MIG_RECONN_TIME``` determines the amount of time between two reconnect attempts.

The time between reconnect attempts will grow exponentially. The first reconnect will take place after the time specified in  ```MIG_RECONN_TIME```. On further reconnects the time will be multiplied by 2 to the power the number of attempts. For example with a reconnect time of 10 seconds reconnects will take place after 10, 20, 40, 80, 160 seconds up to a maximum of 24 hours.

#### MIG_DB_DIR

Used by: **migdb-add-unit**, **migdb-migrate**, **migdb-check-done**

Command Line Option: ```--base <base dir>```

The path to the directory the db directory will be created in, defaults to ./

#### MIG_DEF_SSH_OPTS

Used by: **migdb-migrate**

Command Line Option in migdb-migrate: ```--ssh-opts <ssh options>```

Default ssh options.

Example: ```--ssh-opts "-o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no"```

#### MIG_DURATION

Used by: **migdb-migrate**

Command Line Option in migdb-migrate: ```--dur <duration>```

Wait time in seconds before looking for migrated devices in balena, defaults to 180 sec.


#### MIG_EXEC_TIMEOUT

Used by: **migdb-migrate**

Set a timeout in seconds for the execution of the migrate script on the target device. Defaults to 600 = 10 minutes.

#### MIG_LOG_TO

Used by: **migdb-add-unit**, **migdb-check-done**, **migdb-migrate**

Command Line Option: ```--log-to <log-file>```

The path to a directory to log output to.

#### MIG_MAX_STATUS_AGE

Used by: **migdb-check-done**

Command Line Option: ```--max-age <age in seconds>```

Wait time in seconds before before assuming, that a device has failed to show up in the balena backend. Defaults to 900 (15 minutes). This timer starts after the devices has been migrated (STATUS="MIGRATED").


#### MIG_MIN_AGE

Used by: **migdb-migrate**

Command Line Option: ```--min-age <min age in seconds>```

Miniumum age of a unit file before being processed in seconds.

#### MIG_RECONN_TIME

Used by: **migdb-migrate**

Command Line Option: ```--reconn-time <seconds-between-reconnects>```

How often long (in seconds) to wait between two reconnect attempts.

The parameter ```MIG_CONN_ATTEMPTS``` determines how many reconnects to attempts before migration fails.

#### MIG_SSH_PASSWD

Used by: **migdb-add-unit**, **migdb-migrate**

Command Line Option: ```--passwd <password>```

SSH password for device.

##### Warning

* Please be aware that passwords are being stored unencrypted in text files and might show up in logs. It is recommended to use private/public key authentification instead.
* When using this option the **migdb-migrate** will attempt to use **sshpass** utility that needs to be installed prior to using **migdb-migrate** with the --passwd option.

If this parameter is present in **migdb-add-unit**, it will be written to the unit file and override the same option given elsewhere.

#### MIG_SSH_PORT

Used by: **migdb-add-unit**, **migdb-migrate**

Command Line Option in migdb-migrate: ```--port <port>```

SSH port used to connect to the device, defaults to 22.

If this parameter is present in **migdb-add-unit**, it will be written to the unit file and override the same option given elsewhere.


#### MIG_SSH_HOST

Used by: **migdb-add-unit**, **migdb-migrate**

Command Line Option in migdb-migrate: ```--host <ssh host>```

The ssh host name or IP address of the device to migrate. 

If this parameter is present in **migdb-add-unit**, it will be written to the unit file and override the same option given elsewhere.

#### MIG_SSH_OPTS

Used by: **migdb-add-unit**

Command Line Option in migdb-add-unit: ```--ssh-opts <ssh options>```

Unit specific ssh options. 

If this parameter is present in **migdb-add-unit**, it will be written to the unit file and override the same option given elsewhere.

#### MIG_SSH_USER

Used by: **migdb-add-unit**, **migdb-migrate**

Command Line Option in migdb-migrate: ```--user <ssh user>```

The ssh user name used to connected to the device.

If this parameter is present in **migdb-add-unit**, it will be written to the unit file and override the same option given elsewhere.

#### MIG_TX_TIMEOUT

Used by: **migdb-migrate**

Set a timeout in seconds for the transfer of files to the target device. Defaults to 1200 = 20 minutes.

## Strategy

The migration is performed in two stages. Stage 1 is invoked by running the script balena-migrate.
Stage 2 is invoked during boot from inside an initramfs. The stage 2 scripts will install balenaOS if
configured to do so.

### Stage 1

The first stage ensures that all requirements are met, triggers the creation of a modified initramfs and installs it to be used at the next system reboot. It will reboot the system if configured to do so.


The balena-migrate script will create a configuration file in */etc/balena-migration-stage2.conf* that will determine the actions and required files during initramfs creation as well as for stage 2 which is run from inside initramfs after booting the device.

The balena-migrate script creates an initramfs file that contains all scripts, programs and configuration files needed for phase 2.

The stage 1 script will also perform migration tasks such as creating a backup and migrating network (WIFI) configurations to be installed in balenaOS (resin-boot/system-connections).
The backup files files as well as the actual balenaOS image file are likely too big to be contained in the root partition where the initramfs resides and will be copied from the root file system to initramfs in stage 2.

The stage 1 script will reconfigure the bootloader of the system to use the created initramfs and optionally reboot the system. On systems using grub as bootloader the system is configured to have one shot at the modified boot configuration using grub-reboot. On RPI the */boot/config.txt* and */boot/cmdline.txt* files are modified. They are restored to their original values by the stage 2 script.


### Stage 2

When booted from the modified initramfs the contained stage 2 scripts will attempt to store all required files inside the tmpfs that the initram system is mounted on. The configuration created in stage 1
(*/etc/balena-migrate-stage2.conf*) contains instructions on which files to transfer to the tmpfs.

Any failures up to this point can be tolerated by resetting the boot configuration to the prior state. In grub based boot loaders this is achieved automatically, for RPI devices the */boot/config.txt* and */boot/cmdline.txt* files have to be restored to their prior state and the systems should be able to reboot into its former configuration. For Raspberry PI's the stage 1 script stores copies of the original files in the boot partition renamed to config.txt.TS and cmdline.txt.TS where TS is a timestamp.

After all files have been secured, the stage 2 scripts will unmount the former root file system and flash the configured balenaOS image to the target device. After flashing, the script will trigger a reread of the devices partition information and attempt to mount resin-boot and resin-data to transfer files to.

On recent Balena images and generally on raspberry PI devices the balenaOS image contains all information necessary to reboot the system.

On older Balena images (up to version 2.15) for the x86 platform, the boot loader is not contained in the image. For these installs the stage 2 script needs to create a new boot loader configuration by copying a *grub.cfg* to the 
resin-boot partition and then calling grub-install or flashing boot loader images on the boot device. This process is enabled by setting the **GRUB\_INSTALL** variable to **TRUE** in the balena-migrate.conf file.

A grub.cfg can be supplied by seting the parameter *GRUB_CFG* in the config file. If this parameter is not set, the script will use a simple grub.cfg created by the stage 1 script. Due to changes in recent versions of balenaOS the autocreated grub.cfg does not work any more. The safe way to do this is to supply a cfg.grub that matches the OS version that is being installed by extracting it from the flasher image.


## Supported Platforms

So far the scripts are being tested and are working on the following platforms:

* Raspberry PI 2 on Raspian-9
* Raspberry PI 3 on Raspian-9
* Raspberry PI 3 running NOOBS and Raspian-9
* *Virtualbox x86\_64 systems running ubuntu 14.04 and 18.04 in grub legacy mode.
* x86\_64 intel-nuc running ubuntu 14.04 and 18.04 in grub EFI mode.
* 32 bit intel devices running Ubuntu 14.04 in grub legacy mode


Work is in progress on following platforms: 

* beaglebone black/green

## Migration Stage 1 in Detail

The script *balena-migrate* will check the prerequisites for migration before it attempts to modify the system. It reads its configuration from a file and also supports a number of command line parameters. Parameters set on the command line override options given in the config file.

The script itself can be located anywhere in the file system. It attempts to read its configuration from a file which  defaults to **balena-migrate.conf**. It will look for the file in the current directory and in the directory pointed to by ```HOME_DIR```. The location and name of the config file and ```HOME_DIR``` can both be set using command line parameters.

The directory specified by ```HOME_DIR``` is used to store temporary files and is also expected to contain the initramfs-tools directory provided in this repository. All other paths given in the config file or on the command line
are expected to be relative to ```HOME_DIR``` which defaults to the current working directory.

All parameters have a default except for ```IMAGE_NAME``` which **must** be set in the config file or using command line parameters. It points to the balenaOS image to be flashed.

The script contains a list of supported operating systems and hardware platforms that it has been tested on. It will reject any OS or hardware not contained in that list.

### Prerequisites

#### Environment

The stage 1 scripts expects the boot configuration to be available under ```/boot```. It expects the ```/boot``` directory to be on the same hard drive as the root file system and will fail with an error message if this is not the case.


To create the initramfs *balena-migrate* will create a copy of the systems initramfs in configuration that is expected to be found in ```/etc/initramfs-tools```. It will then set up a modified version using migration scripts contained in the init-scripts folder in *HOME\_DIR*.  

* balena-init - this script copies programs & files to the initramfs at creation time.
* balena-common* - this script contains some helper functions used from other scripts during initramfs boot.
* balena-stage2-default* - this is the main boot time script.

#### Currently accepted Platforms

OS version extracted from */etc/os-release*:

* ubuntu-18.04
* ubuntu-14.04
* raspian-9

Architectures taken from *uname -m*:


  |**uname tag**   |**Architecture**       |**Action Taken**|
  |---------------| ----------------------| ----------------------------------------------------------------|
  | armv7l          |arm v7                 |distinguish RPI's / other devices by analysing */proc/cpuinfo* |
  |  x86\_64         |intel 64 bit systems||   
  |i686            |intel 32 bit systems||



#### Required Programs

The stage 1 script will also make sure, that all the software required to perform the migration is available. Required
software depends on the architecture and OS and includes the following:

  |**Program**|   **Architecture**|          **Version**|
  |--------------| -------------------------| -------------|
  |Stage 1 ||
  |tar            |all                       |unspecified|
  |findmnt        |all                       |unspecified|
  |parted         |all                       |unspecified|
  |lsblk          |all                       |unspecified|
  |readlink       |all                       |unspecified|
  |sed            |all                       |unspecified|
  |mkinitramfs    |all                       |unspecified|
  |grub-update    |x86\_64, i686             |V2|
  |nc    |all             |unspecified|
  |Stage 2|||
  |mount          |all                       |unspecified|
  |dd             |all                       |unspecified|
  |gzip           |all                       |unspecified|
  |partprobe      |all                       |unspecified|
  |udevadm        |all                       |unspecified|
  |grub-install   |x86\_64, i686, optional           |V2|
  |busybox        |x86\_64, i686             |unspecified|


### Configuration

The script *balena-migrate* uses a configuration file to read its configuration from. It defaults to ```balena-migrate.conf```.

*balena-migrate* will print a help message when called with the -h or --help option that states the available command
line options.

The **config file** can be specified on the commandline using the -c or --config option:

``` bash
balena-migrate --config ./migrate.conf
```


The config file uses shell syntax, valid settings for this file are as follows.

#### BACKUP\_SCRIPT, BACKUP\_DEFINITION

**Warning:** Processing of the ```BACKUP_DEFINITION``` variable is unfinished / work in progress. This feature is unlikely to work in the current state of the project and my be subject to changes of definition and implementation in future versions.

**Default:**

``` bash
BACKUP_SCRIPT=
BACKUP_DEFINITION=
```

These variables define how to backup the system.

:If ```BACKUP_SCRIPT``` is specified balena-migrate will invoke the script with one parameter, which specifies the name of the backup file to
create. This is currently the file ```backup.tgz``` and the backup script is expected to create a gzipped tar file with the
given name (eg. by calling ```tar -czf backup.tgz /path-to-backup-directories```).

If the ```BACKUP_SCRIPT``` variable is empty,  ```balena-migrate``` will check the ```BACKUP_DEFINITION``` variable. If it is set and 
points to a valid file, ```balena-migrate``` will attempt to create a backup according to the instructions given in the definition 
file.

The resulting backup file will be transferred to the data partition of the balena installation. On first boot, balena-os will attempt to
create a volume for every top level directory in the backup file, that is referenced by an application container.

Example: 
If your application container uses a volume  'migrated_data' and you have created a backup that contains a top level folder 
'migrated_data' balenaOS will create the volume 'migrated_data' for your application and populate it with the contents of the 
'migrated_data' folder of your backup file.

The ```migratecfg``` folder contains a sample backup script (```sample-backup.sh```), that can be modified to suit your needs. 

Teh sample backup scripts demonstrates how to create a valid backup file. It creates a temporary folder that will contain the 
file system structure to be backed up. It then creates the top level directories that will be migrated to volumes and 
links (via symbolic link) all files and directories that are part of the backup to their target locations in the volumes.  

The backup script will be processed by *balena-migrate* (in stage 1) so it is not critical and can be tested without risking
rendering a device unresponsive.

##### Backup file Size considerations

The stage 1 script attempts to estimate the space available and warn if there is not enough space left.
The backup file and the balenaOS image will be copied to initramfs during stage 2. If the size of the backup and the image
together with all other files needed for migration in stage 2 exceeds the size of the available memory or does not
leave enough memory for processing, stage 2 will fail. This means that the boot process will stall and the device might
remain in a state that requires manual intervention.

The stage 1 and stage 2 scripts attempt to detect this situation and terminate, but the ammount of memory needed can only be 
estimated so care should be taken to keep the backup as small as possible. 

balena-migrate will display a message stating the size of the image, the size of the backup and an estimate of the memory left for 
processing and will fail if the estimated memory falls below a set threshold.

To avoid this please make sure to keep the backup as small as possible.

##### Backup Definition File Syntax

The backup definition file that can be specified using the ```BACK_DEFINITION``` variable follows YAML syntax. 

The ```balena-migrate``` script expects to find a list of one or more volume definitions at the root of the file.
Each volume definition consists of the tag 'volume' and the name of the volume.
The 'content' property contains a list of volume content defintions consisting of:

* 'source' - the  source file or directory
* 'target' - the destination of the file or directory within the volume
* 'filter' - optional - a regular expression that can be used to exclude files from the backup that do not match the regulat expression.


The filter parameter can only be used with directories as it attempts to filter the files within the source directory.

If source points to a directory target can be left empty. In this case    
If neither target nor filter is specified and source is a directory the source directory will be mapped directly to the volume. Only one 
such content definition should be given any volume. 


Example:

``` yaml
- volume: 'sample-volume1'
  content:
  # map source file to target file in volume
  - source: 'path-to-source-file'
    target: 'path-to-target-file'
  # map source directory to target directory in volume
  - source: 'path-to-source-dir'
    target: 'path-to-target-dir'
- volume: 'sample-volume2'
  content:
  # map source dir to be volume directory
  - source: 'path-to-source-dir'
- volume: 'sample-volume3'
  content:
  # store all conf files in source directory in target directory 
  - source: 'path-to-source-dir'
    target: 'path-to-target-dir'
    filter: '.*\.conf'
  # store all conf files in source directory in volume directory 
  - source: 'path-to-source-dir'
    filter: '.*\.conf'
```


#### BALENA\_API\_HOST, BALENA\_API\_PORT

**Default:**

``` bash
BALENA_API_HOST=api.balena-cloud.com
BALENA_API_PORT=443
```

If ```BALENA_API_HOST``` is not empty, the script will attempt to connect to the host and port specified and fail if a connection cannot be  established within the time period specified by ```BALENA_CONNECT_TIMEOUT```.
The parameters can be set to the empty string to disable connection testing or to an alternate URL when using a different API host.  


#### BALENA\_CONFIG

**Default:** *BALENA\_CONFIG=

If specified, the name of a file in ```HOME_DIR``` that will be interpreted as config.json file and copied to the resin-boot partition in stage 2.

The balena config.json file can be specified as a command line parameter using the *--balena-cfg* option as follows:

Example:

``` bash
balena-migrate --balena-cfg=project-config.json
```

#### BALENA\_CONNECT\_TIMEOUT

**Default:** ```BALENA_CONNECT_TIMEOUT=20```

Provides the timeout for the connection attempt triggered by ```BALENA_API_HOST``` or ```BALENA_VPN_HOST```. The timeout in seconds to wait for a connection to the hosts.

#### BALENA\_VPN\_HOST, BALENA\_VPN\_PORT

**Default:**

``` bash
BALENA_VPN_HOST=vpn.balena-cloud.com
BALENA_VPN_PORT=443
```

If ```BALENA_VPN_HOST``` is not empty, the script will attempt to connect to the host and port specified and fail if a connection cannot be established within the time period specified by ```BALENA_CONNECT_TIMEOUT```.
The parameters can be set to the empty string to disable connection testing or to an alternate URL when using a different VPN host.  

#### BALENA\_WIFI

**Default:** ```BALENA_WIFI=```

If this variable is set to *TRUE* and ```BALENA_CONFIG``` points to a file in ```HOME_DIR``` the config file will be scanned for the
ssid & key of a wifi network. If an SSID is found a network manager file will be created for this network and copied
to ```/boot/system-connections``` in stage 2. 

On boot BalenaOS will attempt to create a network connection for every file found in ```/boot/system-connections```.

**Warning:** This option requires jq ( https://stedolan.github.io/jq/ ) to be installed on the target device. Migration will 
fail with an error if jq is not available.


#### COPY\_NMGR\_FILES

**Default:** ```COPY_NMGR_FILES=```

If this variable is set to a space separated list of files, these files will be copied to the ```/boot/system-connections``` folder 
in stage 2. The files must be located in ```HOME_DIR```. 

On boot BalenaOS will attempt to create a network connection for every file found in ```/boot/system-connections```.

#### DEBUG

**Default:** ```DEBUG=```

Sets the debug flag in the kernel command line if set to *TRUE*. If this setting is enabled the initramfs log can be
found in */run/initramfs/initramfs.debug*.

##### DO\_REBOOT

**Default:** ```DO_REBOOT=```

When this variable is set to a numeric value the *balena-migrate* script will reboot the computer after finishing successfully.
It will display a warning and reboot after the number of seconds specified in ```DO_REBOOT```. By default this option is
disabled and the device has to be rebooted manually after stage 1 has terminated.

The variable can be set on the command line by using the -r or --reboot parameters:

``` bash
balena-migrate -r 10

balena-migrate --reboot=10
```

#### GRUB\_INSTALL

**Default:** *GRUB\_INSTALL=*

The variable is the enabling flag for grub installation. If it is set to anything other than TRUE a grub boot manager will not be
installed. The variable is only checked in intel 32/64 bit environments as these are the only setups in which grub is used.
The most recent balenaOS x86 images contain a valid grub boot configuration, so in most cases the variable can remain unset.

When setting ```GRUB_INSTALL``` to *TRUE* please make sure to supply a valid setup using the GRUB variables described in the following sections.

#### GRUB\_BOOT\_DEV

**Default:** *GRUB\_BOOT\_DEV=hd0*

The variable supplies a hint to which grub device to use for boot. The stage 1 script will attempt to determine the GRUB
boot device by using UUID tags. If this fails it will fallback to the above variable and set root to

*\$GRUB\_BOOT\_DEV,X* - X being a grub partition specifier.

For root a partition of /dev/sda1 this would be either *msdos1* or *gpt1* depending on the type of partition table and
thus result to a boot menu entry of

``` bash
set root=hdo,msdos1
```

#### GRUB\_CFG

**Default:** *GRUB\_CFG=*

Specify a grub config file for balenaOS.

The setting is only relevant for platforms on which balenaOS uses grub to boot, relevant platforms are intel 32 or 64
bit devices.

If the variable is not set balena-migrate will create a simple grub.cfg file. At the time of writing this
file has been tested on balena release 2.13 and is expected to work on earlier releases.

Newer versions of balenaOS (seen on 2.26) use a modified layout of the boot files and require a matching *grub.cfg* file
to be provided using this variable.

**Warning:** If an incompatible grub.cfg is used the system will not boot into balenaOS successfully. As the balenaOS
image is already written to the disk, the only way to recover is to repair the boot configuration.

#### HAS\_WIFI\_CFG, MIGRATE\_ALL\_WIFIS, MIGRATE\_WIFI\_CFG

Default:

``` bash
HAS_WIFI_CFG=
MIGRATE_ALL_WIFIS=
MIGRATE_WIFI_CFG=
```

These variables configure the *balena-migrate* scripts attempts to migrate wifi configurations.

* *If ```HAS_WIFI_CFG``` is set to ```TRUE``` the balena-migrate script will assume that the balenaOS image already contains one wifi configuration in ```/resin-boot/system-connections/resin-wifi01``` and will attempt not to overwrite it. The migrated configurations will start with *resin-wifi02*.
* If ```MIGRATE_ALL_WIFIS``` is set to ```TRUE``` all wifi configurations found in ```/etc/wpa\_supplicant/wpa\_supplicant.conf``` or ```/etc/NetworkManager/system-connections``` will result in a ```resin-wifiXX``` file that will be transferred to ```/resin-boot/system-connections``` in stage 2.
* If ```MIGRATE_WIFI_CFG``` is set to the name of a file in ```HOME_DIR``` the migrated wifi configurations will be filtered by the contents of the file. The file is expected to contain one wifi name (ssid) per line. Only configurations referenced in the file will result in a ```resin-wifiXX``` file in ```/resin-boot/system-connections```.

#### HOME\_DIR

**Default:** ```HOME_DIR=./```

The working directory of the migration script and the location where it expects to find initramfs-tools directory,
as well as setup and image files. The default is to use the current working directory.

This variable can be set on the command line using the --home parameter:

``` bash
balena-migrate --home=/migrate-dir
```

#### IMAGE\_NAME

This option **must** be specified. It specifies the balenaOS image that will be flashed to the devices boot device.
The script will fail if the file is not specified or does not exist.

The script expects the image to be in gzipped or zip format. In case of a zip archive the file will
be unpacked and recompressed using gzip.  

**Warning:** Please make sure to supply a valid image for the device. The scripts do not currently ensure a valid image type for the target device. Specifying an invalid image will lead to devices failing to boot. 

**Example:**
*IMAGE\_NAME="resin-raspberrypi3-2.15.1+rev2-dev-v7.16.6.img.gz*"


#### LOG\_DRIVE , LOG\_FS\_TYPE

**Default:** ```LOG\_DRIVE=```

Specify a separate device to receive log files written during stage 2. The device **must** be separate from the
installation device (hard drive) to be able to survive flashing of the balenaOS image. Use a separate hard disk or a
USB stick. The logfiles can be used for debugging of problems in stage 2.

**Example:**

``` bash 
LOG_DRIVE=/dev/sdb1
LOG_FS_TYPE=ext4
```

#### MAX\_BADBLOCKS

**Default:** ```MAX_BADBLOCKS=```

This is currently work in progress. If set the stage 2 script will perform a non distructive write test on the installation device. This operation is performed using ```badblocks -v -n <device>``` and can take anything from several minutes to several hours depending on type and state of the device.

* If ```MAX_BADBLOCKS``` is set to a numerical value, migration will fail if more bad blocks are detected that specified in ```MAX_BADBLOCKS```.
* If ```MAX_BADBLOCKS``` is set to a numerical value terminated with a '%', migration will fail the percentage of bad blocks exceeds the value specified in ```MAX_BADBLOCKS```.

**Example:**

``` bash 
# fail on an bad block detected
MAX_BADBLOCKS=0
```

#### NO\_FLASH

**Default:** ```NO_FLASH=TRUE```

When this variable is set to *TRUE* the script will proceed normally but will terminate at the point in stage 2 where
the mounted root file system would be unmounted and the balenaOS image would be flashed to the device. How to terminate
is determined by the value of the variable ```TERM_EXIT```.

#### NO\_SETUP

Default: ```NO_SETUP=```

When this variable is set to *TRUE* the *balena-migrate* script does not attempt to modify the boot configuration.
It will check the prerequisites create an initramfs but not create any disruptive configuration changes.


#### TERM\_EXIT

**Default:** ```TERM_EXIT="exit 0"```

Determines how to exit when the script terminates prematurely. Default is *“exit 0”* in which case the boot process is
continued normally. Alternative is *“reboot -f”* to force a reboot.
**Warning:** If the boot configuration has not been reset successfully , setting ```TERM_EXIT="reboot -f"``` can lead to
a boot loop.


## Migration Stage 2 in Detail

Stage 2 is made up of a set of scripts that are run when the initramfs is started. The initramfs contains an init script
that will orchestrate actions and call different scripts in different phases.

Scripts in the *local-bottom* directory are called last and after the root file system has been mounted. Only remaining
steps in initramfs at that stage, are to identify the systems init script and hand over control to that script. The ```balena-stage2-default``` script runs
in this stage. If it has successfully written and configured balenaOS it will terminate the boot process by forcing a
reboot.  

The ```balena-stage2-default``` script carries out the following tasks:

### Restore Boot Configuration

The ```/local-bottom/balena-stage2-default``` script attempts to open the balena-migrate-stage2.conf from which it
takes its configuration.

First steps are to restore the original boot configuration on raspberry pi devices. To do this it attempts to mount the
boot partition and restore ```/boot/config.txt``` and ```/boot/cmdline.txt``` from previously created backups.

The script attempts do do this in a failsafe way, not terminating on failure.

### Establish external Logging

If an external log device was configured the ```/local-bottom/balena-stage2-default``` script attempts to mount the
configured log drive and redirect all logging. The output can be found in the file ```migrate.log``` in the root of the device.

The script attempts do do this in a failsafe way, not terminating on failure.

### Copy BalinaOS image, Backups and configs to TMPFS

Next the stage2 script attempts to copy all files, that need to be transferred to balinaOS and are not yet contained in initramfs 
to the tmpfs storage the initramfs resides in. This is a critical stage, that could fail in a non recoverable way. 
If too much data is copied to initramfs the system will run out of memory and will likely fail in a way that will stall 
the boot process. For this reason it is important to make sure that enough memory is available for files still leaving 
enough memory for initramfs processing to work.

The stage 2 script attempts to estimate the available space and will fail in a recoverable way if insufficient space is available.

### Flash balenaOS

After all the above preparations have succeeded the script will unmount the root file system and flash the balenaOS to
the target device. Beginning with this stage the migration is not recoverable. The exit strategy on failure is changed
to ```reboot -f``` to attempt to reboot into balenaOS if something goes wrong.

For the raspberry PI platform and recent images in general a reboot at this stage this is likely to succeed, but the system is still missing
network manager configuration files other than those contained in the image. This will result in the system will not configure its network. 
Grub based systems based on version 2.26 and ealier balenaOS images need a further step to be able to boot successfully.

### Transfer Data to BalenaOS

The script will now attempt to mount the ```/resin-boot``` and ```/resin-data``` partitions.
To do this it calls ```partprobe``` on the root device to reread the changed partition tables.

After the partitions have been mounted successfully the backup files are being transferred to ```/resin-data``` and migrated
wifi-configurations and other supplied network manager files are copied to ```/resin-boot/system-connections```.
If specified the config.json file is copied to ```/resin-boot/```.

### Installing a new Bootloader

On intel based images grub boot manager installation can be configured. This is only necesarry for version 2.26 and earlier balenaOS images.

There are two variants of how grub can be installed:

* If the flasher image contains grub boot images (boot.img and core.img) these images and a valid grub.cfg file can be extracted from the flasher image and copied / flashed to the boot partition.
* For images that do not contain the boot images, a boot loader can be installed using grub. Using this variant shoukd be thoroughly tested on the target platform as grub cannot be guaranted to work from within initramfs.

All relevant files (mainly grub.cfg) have to be prepared in stage 1 so the stage 2 script only places the files in the
appropriate locations / calls grub-install with parameters prepared in stage 1. Calling grub-install from within
initramfs has to be thoroughly tested for all relevant platforms and grub versions. The program requires files to be
copied to initramfs from ```/usr/share/grub``` and attempts to access ```/usr/locale``` but tolerates it not being available.
Having ready made boot.img / core.img files (even integrated into the images) is the more reliable solution.

### Unmounting partitions and reboot

Last step is to unmount all mounted partitions and reboot the system.
