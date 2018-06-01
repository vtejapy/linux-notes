# Shared storage on the network with iSCSI
Many ways to share storage on a network exist. The iSCSI protocol defines a way to see a remote blocks device as a local disk. A remote device on the network is called iSCSI Target, a client which connects to iSCSI Target is called iSCSI Initiator.

## iSCSI Target Setup
Install admin tools first, configure target to persistantly start at boot time and then start it
```
yum -y install targetcli
systemctl enable target
systemctl start target
```
To start using ``targetcli``, run it and to get a layout of the tree interface, run ls
```
targetcli
targetcli shell version 2.1.fb37
Copyright 2011-2013 by Datera, Inc and others.
For help on commands, type 'help'.
/> 
...
```
### Create a Backstore
Backstores enable support for different methods of storing an object on the local machine. Creating a storage object defines the resources the backstore will use. The supported backstores are: block devices, files, pscsi and ramdisks. Block device ``/dev/sdc`` in our case.
```
/> /backstores/block create name=block_storage dev=/dev/sdc
Created block storage object block_backend using /dev/sdc.
```
### Create an iSCSI Target
Create an iSCSI target using a specified name
```
/> iscsi/ create iqn.2017-10.com.noverit.centos:3260
Created target iqn.2015-05.com.noverit.centos:3260.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260
```
### Configure an iSCSI Portal
An iSCSI Portal is an object specifying the IP address and port where the iSCSI target listen to incoming connections.
```
/> /iscsi/iqn.2017-10.com.noverit.centos:3260/tpg1/portals/ create
Using default IP port 3260
Binding to INADDR_ANY (0.0.0.0)
This NetworkPortal already exists in configFS
```
By default, a portal is created when the iSCSI Target is created listening on all IP addresses (0.0.0.0) and the default iSCSI port 3260. Make sure that the 3260 is not used by another application, else specify a different port.

### Configure the LUNs
A Logical Unit Number (LUN) is a number used to identify a logical unit, which is a device addressed by the standard SCSI protocol or Storage Area Network protocols which encapsulate SCSI, such as Fibre Channel or iSCSI itself.  To configure LUNs, create LUNs of already created storage objects.
```
/> /iscsi/iqn.2017-10.com.noverit.centos:3260/tpg1/luns/ create /backstores/block/block_storage
Created LUN 0.
```

### Configure Access List
Create an Access List for each initiator (iscsi client) that will be connecting to the target (iscsi server). This enforces authentication when that initiator connects, allowing only LUNs to be exposed to each initiator. Usually each initator has exclusive access to a LUN. All initiators have unique identifying names IQN. The initiator's unique name IQN must be known to configure ACLs.

Check the iSCSI initiator IQN on the initiator machine you want to give access. For example, for open-iscsi initiator on linux systems, the IQN can be found in the ``/etc/iscsi/initiatorname.iscsi`` file. 

Back to the target machine and configure Access List to permit access from the initiator machine
```
/> cd /iscsi/iqn.2017-10.com.noverit.centos:3260/tpg1/
/> acls create iqn.1994-05.com.redhat:client00.noverit.com
Created Node ACL for iqn.1994-05.com.redhat:client00.noverit.com
Created mapped LUN 0.
/> cd iqn.1994-05.com.redhat:client00.noverit.com
/> set auth userid=user
Parameter userid is now 'user'.
/> set auth password=password
Parameter password is now 'password'.
```

Repeat the steps above for each initiator you need to give permission.

At the end of configuration, the iSCSI target envinronment should look like the following
```
/> ls
o- / .............................................................[...]
  o- backstores ..................................................[...]
  | o- block .....................................................[Storage Objects: 1]
  | | o- block_storage ...........................................[/dev/sdc (12.0GiB) write-thru activated]
  | |   o- alua ..................................................[ALUA Groups: 1]
  | |     o- default_tg_pt_gp ....................................[ALUA state: Active/optimized]
  | o- fileio ....................................................[Storage Objects: 0]
  | o- pscsi .....................................................[Storage Objects: 0]
  | o- ramdisk ...................................................[Storage Objects: 0]
  o- iscsi .......................................................[Targets: 1]
  | o- iqn.2017-10.com.noverit.centos:3260 .......................[TPGs: 1]
  |   o- tpg1 ....................................................[no-gen-acls, no-auth]
  |     o- acls ..................................................[ACLs: 4]
  |     | o- iqn.1994-05.com.redhat:client00.noverit.com .........[Mapped LUNs: 1]
  |     | | o- mapped_lun0 .......................................[lun0 block/block_storage (rw)]
  |     o- luns ..................................................[LUNs: 1]
  |     | o- lun0 ................................................[block/block_storage (/dev/sdc)]
  |     o- portals ...............................................[Portals: 1]
  |       o- 0.0.0.0:3260 ........................................[OK]
  o- loopback ....................................................[Targets: 0]

Save and exit

/> saveconfig
Last 10 configs saved in /etc/target/backup.
Configuration saved to /etc/target/saveconfig.json
/> exit
Global pref auto_save_on_exit=true
Last 10 configs saved in /etc/target/backup.
Configuration saved to /etc/target/saveconfig.json
```

The ``/etc/target/saveconfig.json`` file contains the above configuration.

Restart the target service 
```
systemctl restart target
```
## iSCSI Initiator Setup
After configuring the iSCSI on the target machine, move to setup the iSCSI initiator machine and install the iSCSI initiator

```
yum -y install iscsi-initiator-utils
```

Check the IQN
```
cat /etc/iscsi/initiatorname.iscsi
InitiatorName=iqn.1994-05.com.redhat:2268c31791
```

The last field of the IQN above is the identifier of the the above initiator machine. Change it to a more meaningful string, for example the host name.

```
echo "InitiatorName=iqn.1994-05.com.redhat:client00.noverit.com" > /etc/iscsi/initiatorname.iscsi
```

Edit the ``/etc/iscsi/iscsid.conf`` initiator configuration file
```bash
# line 54: uncomment
node.session.auth.authmethod = CHAP
# line 58,59: uncomment and specify the username and password you set on the iSCSI target server
node.session.auth.username = user
node.session.auth.password = password
```

The iSCSI initiator is composed by two services, iscsi and iscsid, start both and enable to start at system startup
```
systemctl start iscsid iscsi
systemctl enable iscsid iscsi
```

To connect the target, first discover the published iSCSI resouces and then login
```
iscsiadm --mode discovery --type sendtargets --portal centos:3260 --discover
10.10.10.2:3260,1 iqn.2017-10.com.noverit.centos:3260
```
confirm status after discovery
```
iscsiadm -m node -o show 
```
Login to the target by specifying the IQN and the portal
```
iscsiadm --mode node --targetname iqn.2017-10.com.noverit.centos:3260 --portal centos:3260 --login
```

Check the storage block devices on the initiator machine
```
lsblk
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda                         8:0    0 232.9G  0 disk
├─sda1                      8:1    0   500M  0 part /boot
├─sda2                      8:2    0  73.4G  0 part
│ ├─os-swap               253:0    0   3.9G  0 lvm  [SWAP]
│ ├─os-root               253:1    0    50G  0 lvm  /
│ └─os-data               253:2    0 178.5G  0 lvm  /data
└─sda3                      8:3    0   159G  0 part
  └─os-data               253:2    0 178.5G  0 lvm  /data
sdc                         8:32   0    20G  0 disk
```
The disk ``/dev/sdc`` is the remote iSCSI block device exported by the target. It is seen as local block device in the initiator machine. The disk can be used as with standard local disk commands and configurations, including ``fdisk``, ``mkfs``, ``e2label``, etc.

To disconnect the remote devices

    iscsiadm --mode node --targetname iqn.2017-10.com.noverit.centos:3260 --portal centos:3260  --logout

Stop and then disable the services at startup, if required
```
systemctl stop iscsid iscsi
systemctl disable iscsid iscsi
```

## Multipath
Device Mapper Multipath is a feature that allows linux clients to configure multiple I/O paths between clients and a storage array into a single device. These paths are physical network connections that can include separate cables, switches, and network interfaces. Multipath aggregates multiple paths, creating a new device that consists of the aggregated paths. The process of path switching to avoid failed components is known as path failover. In addition to path failover, multipath provides load balancing by distributing I/O loads across multiple paths.

Multipath is used primarily for:

  * **Redundancy:** it provides failover in an active/passive configuration. In an active/passive configuration, only half the paths are used at any time. If any element of a path, i.e. the cable, switch, or network interface fails, the client switches to an alternative path.
  * **Performance:** it can be configured in active/active mode, where all the paths are used in a round-robin fashion. In some cases, multipath can detect the loading on each paths and dynamically re-balance the load.

Pleae, note that multipath protects against network devices failures but does not against storage devices failures.

As example, let to create two paths between the iSCSI Target and the client (initiator). To achieve this, connect the target to the clients by two separate PT networks: netA1 and netA2

```
   iSCSI Initiator                              iSCSI Target
+------------------+                       +-------------------+
|                  +---+      netA1    +---+                   |
|    172.19.20.83  |   +---------------+   | 172.19.20.2:3260  |
|                  +---+               +---+                   |
|                  |                       |   +-----------+   |
|                  |                       |   |   LUN 0   |   |
|                  |                       |   +-----------+   |
|                  +---+      netA2    +---+                   |
|    172.19.21.83  |   +---------------+   | 172.19.21.2:3260  |
|                  +---+               +---+                   |
+------------------+                       +-------------------+
```

### iSCSI Target Multipath configuration
On the iSCSI Target, through the ``targetcli`` command, create two portals listening on two separate network interfaces. They both have to point to the same LUN mapped on the same block device.

```
o- / .............................................................[...]
  o- backstores ..................................................[...]
  | o- block .....................................................[Storage Objects: 1]
  | | o- block_storage ...........................................[/dev/sdc (12.0GiB) write-thru activated]
  | |   o- alua ..................................................[ALUA Groups: 1]
  | |     o- default_tg_pt_gp ....................................[ALUA state: Active/optimized]
  | o- fileio ....................................................[Storage Objects: 0]
  | o- pscsi .....................................................[Storage Objects: 0]
  | o- ramdisk ...................................................[Storage Objects: 0]
  o- iscsi .......................................................[Targets: 1]
  | o- iqn.2017-10.com.noverit.centos:3260 .......................[TPGs: 1]
  |   o- tpg1 ....................................................[no-gen-acls, no-auth]
  |     o- acls ..................................................[ACLs: 4]
  |     | o- iqn.1994-05.com.redhat:client00.noverit.com .........[Mapped LUNs: 1]
  |     | | o- mapped_lun0 .......................................[lun0 block/block_storage (rw)]
  |     o- luns ..................................................[LUNs: 1]
  |     | o- lun0 ................................................[block/block_storage (/dev/sdc)]
  |     o- portals ...............................................[Portals: 1]
  |       o- 172.19.20.2:3260 ....................................[OK]
  |       o- 172.19.21.2:3260 ....................................[OK]
  o- loopback ....................................................[Targets: 0]
```

Save the configuration and exit.

To make sure the iSCSI target is listening on the two different network interfaces, check the sockets

    netstat -natp | grep -i 3260
    tcp        0      0 172.19.20.2:3260        0.0.0.0:*               LISTEN      -
    tcp        0      0 172.19.21.2:3260        0.0.0.0:*               LISTEN      -

### iSCSI Initiator Multipath configuration
On the initiator, we discover two portals

    iscsiadm --mode discovery
    172.19.20.2:3260 via sendtargets
    172.19.21.2:3260 via sendtargets

If we try to login to both of these

    iscsiadm -m node -l

we get two separate storage devices for each path

    lsscsi
    ...
    [20:0:0:0]   disk    LIO-ORG  block_storage    4.0   /dev/sdb
    [21:0:0:0]   disk    LIO-ORG  block_storage    4.0   /dev/sdc

To achieve a multipath support for the iSCSI device, we need to install the multipath daemon on the client linux machine

    yum install device-mapper-multipath -y

and load the kernel module

    modprobe dm-multipath

Set up the multipath with the ``mpathconf`` utility, which creates the multipath ``/etc/multipath.conf`` configuration file 

    mpathconf --enable --with_multipathd y

The default settings for multipath are compiled in to the system and do not need to be explicitly set. However, check the ``/etc/multipath.conf`` file and change it according to your needs

```
defaults {
        polling_interval        10
        path_selector           "round-robin 0"
        failback                immediate
        no_path_retry           fail
        user_friendly_names     yes
        find_multipaths         yes
}
```

To see the complete multipath configuration, use the following command

    multipathd show config

Start the multipath daemon

    systemctl start multipathd

### Multipath naming devices
Multipath provides a way of organizing the I/O paths logically, by creating a single multipath device on top of the underlying devices. Each multipath device has a **World Wide Identifier**, also called **WWI** which is globally unique and unchanging in the system. By default, the name of a multipath device is set to its WWID. Alternately, you can get a more fancy name as ``/dev/mapper/mpath{x}`` by setting the ``user_friendly_names`` option to ``true`` in the multipath configuration file.

When new devices are under the control of the multipath daemon, the new multipath device may be seen in two different places: the ``/dev/mapper/mpath{x}`` and ``/dev/dm-{x}``. All devices of the form ``/dev/dm-{x}`` are for internal use only and should never be used by the administrator.

### Creating multipath devices
On the client machine, make sure the multipath daemon is running without issues and login to all the sessions from the iSCSI Target

    iscsiadm -m node -l

and check the multipath discovery

    multipath -ll
    mpatha (36001405fe7fe49282154438990f322bf) dm-19 LIO-ORG ,block_storage
    size=12G features='0' hwhandler='0' wp=rw
    `-+- policy='round-robin 0' prio=50 status=active
      |- 28:0:0:0 sdb 8:16 active ready running
      `- 29:0:0:0 sdc 8:32 active ready running

We see the two disks ``/dev/sdb`` and ``/dev/sdc`` under the control of the multipath daemon. The are both load balanced with policy ``round-robin 0`` as we set the ``path_selector`` option in the multipath configuration file.

The multipath disk is available with the fancy name ``/dev/mapper/mpatha``.

The user can make a file system on it and make a mount as unique device

    mkfs -t xfs /dev/mapper/mpatha
    mount /dev/mapper/mpatha /mnt
    
    df -Th /mnt
    Filesystem         Type  Size  Used Avail Use% Mounted on
    /dev/mapper/mpatha xfs    12G   33M   12G   1% /mnt

    echo pstree > /mnt/pstree.log
    
In case of failure of one of the network path between client and server, the multipath daemon will take care to manage the failover. To test this, shutdown one of the interface on the client machine


     netstat -nr
      Kernel IP routing table
      Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
      0.0.0.0         10.10.10.1      0.0.0.0         UG        0 0          0 ens32
      10.10.10.0      0.0.0.0         255.255.255.0   U         0 0          0 ens32
      172.19.20.0     0.0.0.0         255.255.255.0   U         0 0          0 ens34
      172.19.21.0     0.0.0.0         255.255.255.0   U         0 0          0 ens35

     ifdown ens35

The multipath detects the failure in the path and then move all the traffic from the faulty path to the healty one

    multipath -ll
    mpatha (36001405fe7fe49282154438990f322bf) dm-19 LIO-ORG ,block_storage
    size=12G features='0' hwhandler='0' wp=rw
    `-+- policy='round-robin 0' prio=50 status=active
      |- 28:0:0:0 sdb 8:16 active faulty running
      `- 29:0:0:0 sdc 8:32 active ready running

However, the iSCSI disk is still there

    echo $(date) > /mnt/date.log
    cat /mnt/date.log
    Sat Jan 27 18:51:06 CET 2018

This demonstrate a simple usage of multipath in iSCSI.

## Open iSCSI administrative commands
The ``iscsiadm`` utility allows access and management of the iSCSI database on Linux. Following the most useful options.

View available portals

    iscsiadm -m discovery -o show

Discover available targets from a portal

    iscsiadm -m discovery -t sendtargets -p ipaddress:port

List targets to login

    iscsiadm -m node -o show

Log into a specific target

    iscsiadm -m node -T targetname -p ipaddress:port -l

Login into all targets

    iscsiadm -m node -l
    
Log out of a specific target

    iscsiadm -m node -T targetname -p ipaddress:port -u

Display information about a target

    iscsiadm -m node -T targetname -p ipaddress:port

Display statistics about a target

    iscsiadm -m node -s -T targetname -p ipaddress:port

Logout from all the targets

    iscsiadm -m node -u

To display all active sessions logged into

    iscsiadm -m session
    
or with more details

    iscsiadm -m session -P 0|1|3

In case of trouble, check the content of ``/var/lib/iscsi/``.






