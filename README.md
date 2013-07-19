nfs_automount 
=============

Version 1.0 (2013-07-18)

The goal of this script is to provide static (i.e. /etc/fstab-like) NFS mounts, while at the same time supporting cross-mounts between servers.  

The other non-fstab alternative is to lazy-mount NFS shares with autofs (where available), but with it NFS shares are not continually maintained. When a remote share is accessed, it takes a few moments for it to become accessible as autofs mounts the share on-demand.  While autofs times out a mounted share after some time of inactivity, it does not unmount the share before the timeout has lapsed in the event the remote server becomes inaccessible. While on-demand mounting may save some bandwidth, it is not suitable for all applications. Furthermore, when a system has one or more active mounted shares off of a server that goes offline, unexpected behavior is often observed on the client server until the now-defunct NFS shares are unmounted, or the remote server becomes available once again.

nfs_automount offers a solution: 

* The NFS shares are not statically defined in /etc/fstab so that the system startup is not delayed even when the remote server is not available. As soon as the shares become available they're automatically mounted. If multiple servers cross-mount NFS shares from each other, and the servers are turned on at the same time, nfs_automount ensures that all mounts are established as soon as the shares become available.

* The shares are monitored at a frequency you define, for example, every 60 seconds. If a share has become dismounted, stale, or their exporting server has become inaccessible, nfs_automount takes action to correct the situation: dismounted and stale shares are attempted to be remounted (stale shares are first immediately unmounted), and shares whose remote NFS service has disappeared are unmounted to prevent impact on the client system stability. Once a remote NFS service returns online, or definition of a previously stale share is reinstated, any shares that were unmounted as a result of those conditions are remounted.

* The script is intended to run as a daemon (an upstart job script is provided for Ubuntu), and it reads its configuration from /etc/nfs-automount.conf where you can conveniently define the shares to be mounted and monitored along with some other options.  You can also set 'RUNTYPE' option to 'cron', and run the script from crontab if you so choose.

* You can define the shares to be mounted either as Read/Write, or Read Only. Of course, a share will be Read Only regardless of this setting if it has been exported as Read Only on the remote server.

* An option to define a remote check file is provided.  If provided in the configuration for a share, its unreachability can alert of a problem on the exporting server, such as a failed filesystem mount, even when the NFS share is otherwise working correctly.  You can easily expand this feature to add additional functionality.

* Provides clear logging which provides alerts by default, and more informative detail if you turn 'DEBUGLOG' setting to 'true'.

* Written in bash script with modular and clear syntax.  

* Tested on Ubuntu 12.x (should also work on Debian) and CentOS 6.x (should also work on RedHat).  

* Distributed under MIT license.


This complete rewrite of nfs_automount is based on older versions I wrote ([July 2010](http://my.galagzee.com/2010/07/23/mounting-nfs-share-after-boot-and-checking-up-on-it-periodically/), [May 2011](http://my.galagzee.com/2011/05/26/nfs-automount-linux-version/), and [December 2011](http://my.galagzee.com/2011/12/19/nfs-enforcer/)).  When I started making further changes to the script in July 2013 I was unhappy with the original script's deeply nested structure which made it problmatic to extend it as I wanted.  I also came across [AutoNFS](https://help.ubuntu.com/community/AutomaticallyMountNFSSharesWithoutAutofsHowto) script on Ubuntu's Community Wiki which gave me further ideas and inspiration. 

Please let me know if you come across any problems! Pull requests are welcome.


Installation
------------

** NOTE: The service installation instructions have been written for Ubuntu, so if you're installing the script for CentOS/RedHat, you will need to alter the installation steps somewhat.

** NOTE: The provided configuration file and the provided upstart job file are both named nfs_automount.conf; they are provided in separate directories mirroring the actual locations in the cloned folder, i.e. **_[cloned folder]_/etc/init/nfs_automount.conf** and **_[cloned folder]_/etc/nfs_automount.conf**.

Clone nfs_automount to the location of your choice (here '/opt/nfs_automount'), assign the ownership of the file **nfs_automount** to the root, and set its permissions to 700. Finally, either symlink the file from /usr/local/bin, or simply copy it there.

    sudo su
    cd /opt
    git clone https://github.com/vwal/nfs_automount.git
    cd nfs_automount
    chmod 700 nfs_automount
    chown root:root nfs_automount

    ln -s /opt/nfs_automount/nfs_automount /usr/local/bin/nfs_automount
     -or-
    cp /opt/nfs_automount/nfs_automount /usr/local/bin/nfs_automount

Save the configuration file **./etc/nfs_automount.conf** to the **/etc** folder, assign its ownership to the root, and set its permissions to 640.

    sudo su
    cd /etc
    cp /opt/nfs_automount/etc/nfs_automount.conf .
    chmod 640 nfs_automount.conf
    chown root:root nfs_automount.conf
    
Modify the configuration in nfs_automount.conf.  Specifically you want to replace the example "MOUNTS" entries with those of your own.  You may also want to adjust the INTERVAL which determines the frequency the NFS the defined NFS shares are checked.

At this point you can test the script by running it at the console.  Setting 'RUNTYPE' to 'cron' will result in the script running once and then terminating, while 'service' setting runs continuously, checking the shares every X seconds (as defined by the 'INTERVAL' option).  Extended logging is enabled by default in the nfs_automount.conf, so you will full details on screen while the script is running in the console.

Next, in order to install the script as a daemon, give the upstart job file **./etc/init/nfs_automount** the proper permissions, assign the ownership to the root, and copy it from the cloned directory to **/etc/init**.  Finally add a reference link in **/etc/init.d** so that the service can be controlled like a normal service:

    sudo su
    cd /opt/nfs_automount/etc/init
    chown root:root nfs_automount.conf
    cp nfs_automount.conf /etc/init
    cd /etc/init.d
    ln -s /lib/init/upstart-job nfs_automount
    
Start the service with `start nfs_automount`.  Note that the log will be written to **/var/log/upstart/nfs_automount.log** unless you have changed 'LOGTYPE' from 'echo' to 'log' (in which case log is written to the location of your choice, by default /var/log/nfs_automount.log).

Also note that whenever you modify the configuration file at /etc/nfs_automount.conf, you'll need to issue `service nfs_automount restart` in order for the changes to become effective.

