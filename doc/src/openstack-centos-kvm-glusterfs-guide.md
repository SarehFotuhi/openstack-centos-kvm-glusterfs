% A Step-by-Step Guide to Installing OpenStack on CentOS Using the KVM Hypervisor and GlusterFS
  Distributed File System
% Anton Beloglazov; Sareh Fotuhi Piraghaj; Mohammed Alrokayan; Rajkumar Buyya


\newpage


# Introduction

- Cloud Computing [@armbrust2010view; @buyya2009cloud]
- Public / Private / Hybrid
- Why Open Source Cloud Platforms are Important
- OpenStack / Eucalyptus / CloudStack / OpenNebula
- Complexity of Installing OpenStack
- Our Step-by-Step Scripted Installation Approach
- The purpose is not just having an up and running OpenStack installation, but also learning the steps
  required to perform the installation from the ground up and understanding the responsibilities and
  interaction of the OpenStack components.

# Comparison of Open Source Cloud Platforms

- OpenStack
- Eucalyptus
- CloudStack
- OpenNebula

# Overview of the OpenStack Cloud Platform

- History
- Features
- Main Services
- Service Interaction

# Existing OpenStack Installation Tools

- DevStack^[http://devstack.org/]
- Puppet / Chef^[http://docs.openstack.org/trunk/openstack-compute/admin/content/openstack-compute-deployment-tool-with-puppet.html]
- Difference From our Approach


# Step-by-Step OpenStack Installation

## Hardware Setup

The testbed used for testing the installation scripts consists of the following hardware:

- 1 x Dell Optiplex 745
	- Intel(R) Core(TM)2 CPU (2 cores, 2 threads) 6600 @ 2.40GHz
	- 2GB DDR2-667
	- Seagate Barracuda 80GB, 7200 RPM SATA II (ST3808110AS)
	- Broadcom 5751 NetXtreme Gigabit Controller

- 4 x IBM System x3200 M3
	- Intel(R) Xeon(R) CPU (4 cores, 8 threads), X3460 @ 2.80GHz
	- 4GB DDR3-1333
	- Western Digital 250 GB, 7200 RPM SATA II (WD2502ABYS-23B7A)
	- Dual Gigabit Ethernet (2 x Intel 82574L Ethernet Controller)

- 1 x Netgear ProSafe 16-Port 10/100 Desktop Switch FS116

The Dell Optiplex 745 machine has been chosen to serve as a management host running all the major
OpenStack services. The management host is referred to as the *controller* further in the text. The 4
IBM System x3200 M3 servers are used as *compute hosts*, i.e. for hosting VM instances.

Due to specifics of our setup, the only one machine connected to public network and the Internet is
one of the IBM System x3200 M3 servers. This server is refereed to as the *gateway*. The gateway is
connected to the public network via the eth0 network interface.

All the machines form a local network connected through the Netgear FS116 network switch. The
compute hosts are connected to the local network via their eth1 network interfaces. The controller
is connected to the local network through its eth0 interface. To provide the access to the public
network and the Internet, the gateway performs Network Address Translation (NAT) for the hosts from
the local network.


## Organization of the Installation Package

The project contains a number of directories, whose organization is explained in this section. The
`config` directory includes configuration files, which are used by the installation scripts and
should be modified prior to the installation. The `lib` directory contains utility scripts that are
shared by the other installation scripts. The `doc` directory comprises the source and compiled
versions of the documentation.

The remaining directories directly include the step-by-step installation scripts. The names of these
directories have a specific format. The prefix (before the first dash) is the number denoting the
order of execution. For example, the scripts from the directory with the prefix *01* must be
executed first, followed by the scripts from the directory with the prefix *02*, etc. The middle
part of a directory name denotes the purpose of the scripts in this directory. The suffix (after the
last dash) specifies the host, on which the scripts from this directory should be executed on. There
are 4 possible values of the target host prefix:

- *all* -- execute the scripts on all the hosts;
- *compute* -- execute the scripts on all the compute hosts;
- *controller* -- execute the scripts on the controller;
- *gateway* -- execute the scripts on the gateway.

For example, the first directory is named `01-network-gateway`, which means that (1) the scripts from this
directory must be executed in the first place; (2) the scripts are supposed to do a network set up;
and (3) the scripts must be executed only on the gateway. The name `02-glusterfs-all` means: (1) the
scripts from this directory must be executed after the scripts from `01-network-gateway`; (2) the
scripts set up GlusterFS; and (3) the scripts must be executed on all the hosts.

The names of the installation scripts themselves follow a similar convention. The prefix denotes the
order, in which the scripts should be run, while the remaining part of the name describes the
purpose of the script.


## Configuration Files

The `lib` directory contains configuration files used by the installation scripts. These
configuration files should be modified prior to running the installation scripts. The configuration
files are described below.


  `configrc:`

  :    This files contains a number of environmental variables defining various aspects of OpenStack's
       configuration, such as administration and service account credentials, as well as access
       points. The file must be "sourced" to export the variables into the current shell session.
       The file can be sourced directly by running: `. configrc`, or using the scripts described
       later. A simple test to check whether the variables have been correctly exported is to `echo`
       any of the variables. For example, `echo $OS_USERNAME` must output `admin` for the
       default configuration.

  `hosts:`

  :    This files contains a mapping between the IP addresses of the hosts in the local network and
       their host names. We apply the following host name convention: the compute hosts are named
       *computeX*, where *X* is replaced by the number of the host. According the described hardware
       setup, the default configuration defines 1 `controller` (192.168.0.1), and 4 compute hosts:
       `compute1` (192.168.0.1), `compute2` (192.168.0.2), `compute3` (192.168.0.3), `compute4`
       (192.168.0.4). As mentioned above, in our setup one of the compute hosts is connected to the
       public network and acts as a gateway. We assign to this host the host name `compute1`, and
       also alias it as `gateway`.


  `ntp.conf:`

  :    This files contains a list of Network Time Protocol (NTP) servers to use by all the hosts. It
       is important to set accessible servers, since time synchronization is important for OpenStack
       services to interact correctly. By default, this file defines servers used within the
       University of Melbourne. It is advised to replace the default configuration with a list of
       preferred servers.

It is important to replaced the default configuration defined in the described configuration files,
since the default configuration is tailored to the specific setup of our testbed.


## Installation Procedure

### CentOS Installation

The installation scripts have been tested with CentOS 6.3^[http://www.centos.org/], which has been
installed on all the hosts. The CentOS installation mainly follows the standard process described in
detail in the Red Hat Enterprise Linux 6 Installation Guide [@redhat2012installation]. The steps of
the installation process that differ from the standard are discussed in this section.


#### Network Configuration.

The simplest way to configure network is during the OS installation process. As mentioned above, in
our setup, the gateway is connected to two networks: to the public network through the eth0
interface; and to the local network through the eth1 interface. Since in our setup the public
network configuration can be obtain from a DHCP server, in the configuration of the eth0 interface
it is only required to enable automatic connection by enabling the "Connect Automatically" option.
We use static configuration for the local network; therefore, eth1 has be configured manually. Apart
from enabling the "Connect Automatically" option, it is necessary to configure IPv4 by adding an IP
address and netmask. According to the configuration defined in the `hosts` file described above, we
assign 192.168.0.1/24 to the gateway.

One difference in the network configuration of the other compute hosts (`compute2`, `compute3`, and
`compute4`) from the gateway is that eth0 should be kept disabled, as it is unused. The eth1
interface should be enabled by turning on the "Connect Automatically" option. The IP address and
netmask for eth1 should be set to 192.168.0.*X*/24, where *X* is replaced by the compute host
number. The gateway for the compute hosts should be set to 192.168.0.1, which the IP address of the
gateway host. The controller is configured similarly to the compute hosts with the only difference
that the configuration should be done for eth0 instead of eth1, since the controller has only one
network interface.


#### Hard Drive Partitioning.

The hard drive partitioning scheme is the same for all the compute hosts, but differs for the
controller. Table 1 shows the partitioning scheme for the compute hosts. `vg_base` is a volume group
comprising the standard Operating System (OS) partitions: `lv_root`, `lv_home` and `lv_swap`.
`vg_gluster` is a special volume group containing a single `lv_gluster` partition, which is
dedicated to serve as a GlusterFS brick. The `lv_gluster` logical volume is formatted using the
XFS^[http://en.wikipedia.org/wiki/XFS] file system, as recommended for GlusterFS bricks.

Table: Partitioning scheme for the compute hosts

+---------------------+----------+--------------------+---------+
|Device               |Size\ (MB)|Mount Point / Volume|Type     |
+=====================+==========+====================+=========+
|*LVM Volume Groups*  |          |                    |         |
+---------------------+----------+--------------------+---------+
|\ \ vg\_base         |20996     |                    |         |
+---------------------+----------+--------------------+---------+
|\ \ \ \ lv\_root     |10000     |/                   |ext4     |
+---------------------+----------+--------------------+---------+
|\ \ \ \ lv\_swap     |6000      |                    |swap     |
+---------------------+----------+--------------------+---------+
|\ \ \ \ lv\_home     |4996      |/home               |ext4     |
+---------------------+----------+--------------------+---------+
|\ \ vg\_gluster      |216972    |                    |         |
+---------------------+----------+--------------------+---------+
|\ \ \ \ lv\_gluster  |216972    |/export/gluster     |xfs      |
+---------------------+----------+--------------------+---------+
|*Hard Drives*        |          |                    |         |
+---------------------+----------+--------------------+---------+
|\ \ sda              |          |                    |         |
+---------------------+----------+--------------------+---------+
|\ \ \ \ sda1         |500       |/boot               |ext4     |
+---------------------+----------+--------------------+---------+
|\ \ \ \ sda2         |21000     |vg\_base            |PV (LVM) |
+---------------------+----------+--------------------+---------+
|\ \ \ \ sda3         |216974    |vg\_gluster         |PV (LVM) |
+---------------------+----------+--------------------+---------+


Table 2 shows the partitioning scheme for the controller. It does not include a `vg_gluster` volume
group since the controller is not going to be a part of the GlusterFS volume. However, the scheme
includes two new important volume groups: `nova-volumes` and `vg_images`. The `nova-volumes` volume
group is used by OpenStack Nova to allocated volumes for VM instances. This volume group is managed
by Nova; therefore, there is not need to create logical volumes manually. The `vg_images` volume
group and its `lv_images` logical volume are devoted for storing VM images by OpenStack Glance.
The mount point for `lv_images` is `/var/lib/glance/images`, which is the default directory used by
Glance to store image files.

Table: Partitioning scheme for the controller

+-------------------+----------+----------------------+---------+
|Device             |Size\ (MB)|Mount Point / Volume  |Type     |
+===================+==========+======================+=========+
|*LVM Volume Groups*|          |                      |         |
+-------------------+----------+----------------------+---------+
|\ \ nova-volumes   |29996     |                      |         |
+-------------------+----------+----------------------+---------+
|\ \ \ \ Free       |29996     |                      |         |
+-------------------+----------+----------------------+---------+
|\ \ vg\_base       |16996     |                      |         |
+-------------------+----------+----------------------+---------+
|\ \ \ \ lv\_root   |10000     |/                     |ext4     |
+-------------------+----------+----------------------+---------+
|\ \ \ \ lv\_swap   |2000      |                      |swap     |
+-------------------+----------+----------------------+---------+
|\ \ \ \ lv\_home   |4996      |/home                 |ext4     |
+-------------------+----------+----------------------+---------+
|\ \ vg\_images     |28788     |                      |         |
+-------------------+----------+----------------------+---------+
|\ \ \ \ lv\_images |28788     |/var/lib/glance/images|ext4     |
+-------------------+----------+----------------------+---------+
|*Hard Drives*      |          |                      |         |
+-------------------+----------+----------------------+---------+
|\ \ sda            |          |                      |         |
+-------------------+----------+----------------------+---------+
|\ \ \ \ sda1       |500       |/boot                 |ext4     |
+-------------------+----------+----------------------+---------+
|\ \ \ \ sda2       |17000     |vg\_base              |PV (LVM) |
+-------------------+----------+----------------------+---------+
|\ \ \ \ sda3       |30000     |nova-volumes          |PV (LVM) |
+-------------------+----------+----------------------+---------+
|\ \ \ \ sda4       |28792     |                      |Extended |
+-------------------+----------+----------------------+---------+
|\ \ \ \ \ \ sda5   |28788     |vg_images             |PV (LVM) |
+-------------------+----------+----------------------+---------+


### Network Gateway

Once CentOS is installed on all the machines, the next step is to configure NAT on the gateway to
enable the Internet access on all the hosts. First, it is necessary to check whether the Internet
is available on the gateway itself. If the Internet is not available, the problem might be in the
configuration of eth0, the network interface connected to the public network in our setup.

In all the following steps, it is assumed that the user logged in is `root`. If the Internet is
available on the gateway, it is necessary to install
Git^[http://en.wikipedia.org/wiki/Git_(software)] to be able to clone the repository containing the
installation scripts. This can be done using yum, the default package manager in CentOS, as follows:

```Bash
yum install -y git
```

Next, the repository can be clone using the following command:

```Bash
git clone https://github.com/beloglazov/openstack-centos-kvm-glusterfs.git
```

Then, we can proceed to continue the configuration using the scripts contained in the cloned Git
repository. As described above, the starting point is the directory called `01-network-gateway`.

```Bash
cd openstack-centos-kvm-glusterfs/01-network-gateway
```

All the scripts described below can be run by executing `./<script name>.sh` on the command line.


(@) `01-iptables-nat.sh`

This script flushes all the default `iptables` rules to open all ports. This is acceptable for
testing; however, it is not recommended for production environments due to security concerns. Then,
the script sets up NAT using `iptables` by forwarding packets from eth1 (the local network
interface) through eth0. The last stage is saving the defined `iptables` rules restarting the service.

```Bash
# Flush the iptables rules.
iptables -F
iptables -t nat -F
iptables -t mangle -F

# Set up packet forwarding for NAT
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -i eth1 -j ACCEPT
iptables -A FORWARD -o eth1 -j ACCEPT

# Save the iptables configuration into a file and restart iptables
service iptables save
service iptables restart
```

(@) `02-ip-forward.sh`

By default, IP packet forwarding is disabled in CentOS; therefore, it is necessary to enable it by
modifying the `/etc/sysctl.conf` configuration file. This is done by the `02-ip-forward.sh` script
as follows:

```Bash
# Enable IP packet forwarding
sed -i 's/net.ipv4.ip_forward = 0/net.ipv4.ip_forward = 1/g' \
    /etc/sysctl.conf

# Restart the network service
service network restart
```

(@) `03-copy-hosts.sh`

This script copies the `hosts` file from the `config` directory to `/etc` locally, as well to all
the other hosts: the remaining compute hosts and the controller. The `hosts` files defines a mapping
between the IP addresses of the hosts and host names. For convenience, prior to copying you may use
the `ssh-copy-id` program to copy the public key to the other hosts for password-less SSH access.
Once the `hosts` file is copied to all the hosts, they can be accessed by using their respective
host names instead of the IP addresses.

```Bash
# Copy the hosts file into the local configuration
cp ../config/hosts /etc/

# Copy the hosts file to the other nodes.
scp ../config/hosts root@compute2:/etc/
scp ../config/hosts root@compute3:/etc/
scp ../config/hosts root@compute4:/etc/
scp ../config/hosts root@controller:/etc/

```

From this point, all the installation steps on any host can be performed remotely over SSH.


### GlusterFS Distributed Replicated Storage

In this section, we describe how to set up distributed replicated storage using GlusterFS.

#### 02-glusterfs-all (all nodes)

The steps discussed in this section need to be run on all the hosts. The easiest way to manage
multi-node installation is to SSH into all the hosts from another machine using separate terminals.
This way the hosts can be conveniently managed from a single machine. Before applying further
installation, it is necessary to run the following commands:

```Bash
yum update -y
yum install -y git
git clone https://github.com/beloglazov/openstack-centos-kvm-glusterfs.git

```

It is optional but might be useful to install other programs on all the hosts, such as `man`,
`nano`, or `emacs` for reading manuals and editing files.


(@) `01-iptables-flush.sh`

This script flushes all the default `iptables` rules to allow connections through all the ports. As
mentioned above, this is insecure and not recommended for production environments. For production it
is recommended to open the specific required ports.

```Bash
# Flush the iptables rules.
iptables -F

# Save the configuration and restart iptables
service iptables save
service iptables restart

```


(@) `02-selinux-permissive.sh`

This script switches SELinux^[http://en.wikipedia.org/wiki/Security-Enhanced_Linux] into the
permissive mode. By default, SELinux blocks certain operations, such as VM migrations. Switching
SELinux into the permissive mode is not recommended for production environments, but is acceptable
for testing purposes.

```Bash
# Set SELinux into the permissive mode
sed -i 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/selinux/config
echo 0 > /selinux/enforce
```


(@) `03-glusterfs-install.sh`

This script installs GlusterFS services and their dependencies.

```Bash
# Install GlusterFS and its dependencies
yum -y install \
    openssh-server wget fuse fuse-libs openib libibverbs \
	http://download.gluster.org/pub/gluster/glusterfs/LATEST/CentOS/glusterfs-3.3.0-1.el6.x86_64.rpm \
	http://download.gluster.org/pub/gluster/glusterfs/LATEST/CentOS/glusterfs-fuse-3.3.0-1.el6.x86_64.rpm \
	http://download.gluster.org/pub/gluster/glusterfs/LATEST/CentOS/glusterfs-server-3.3.0-1.el6.x86_64.rpm
```


(@) `04-glusterfs-start.sh`

This script starts the GlusterFS service, and sets the service to start during the system start up.

```Bash
# Start the GlusterFS service
service glusterd restart
chkconfig glusterd on
```

#### 03-glusterfs-controller (controller)

The scripts described in this section need to be run only on the controller.


(@) `01-glusterfs-probe.sh`

This script probes the compute hosts to add them to a GlusterFS cluster.

```Bash
# Probe GlusterFS peer hosts
gluster peer probe compute1
gluster peer probe compute2
gluster peer probe compute3
gluster peer probe compute4
```


(@) `02-glusterfs-create-volume.sh`

This scripts creates a GlusterFS volume out of bricks exported by the compute hosts mounted to
`/export/gluster` for storing VM instances. The created GlusterFS volume is replicated across all
the 4 compute hosts. Such replication provides fault tolerance, as if any of the compute hosts fail,
the VM instance data will be available from the remaining replicas. Compared to a Network File System
(NFS) exported by a single server, the complete replication provided by GlusterFS improves the read
performance, since all the read operations are local. This is important to enable efficient live
migration of VMs.

```Bash
# Create a GlusterFS volume replicated over 4 gluster hosts
gluster volume create vm-instances replica 4 \
    compute1:/export/gluster compute2:/export/gluster \
	compute3:/export/gluster compute4:/export/gluster

# Start the created volume
gluster volume start vm-instances
```


#### 04-glusterfs-all (all nodes)

The script described in this section needs to be run on all the hosts.


(@) `01-glusterfs-mount.sh`

This scripts adds a line to the `/etc/fstab` configuration file to automatically mount the GlusterFS
volume during the system start up to the `/var/lib/nova/instances` directory. The
`/var/lib/nova/instances` directory is the default location where OpenStack Nova stores the VM
instances related data. This directory must be shared be all the compute hosts to enable live
migration of VMs. The `mount -a` command re-mounts everything from the config file after it has been
modified.

```Bash
# Mount the GlusterFS volume
mkdir -p /var/lib/nova/instances
echo "localhost:/vm-instances /var/lib/nova/instances glusterfs defaults 0 0" \
    >> /etc/fstab
mount -a
```


### KVM

The scripts included in the `05-kvm-compute` directory need to be run on the compute hosts. KVM is
not required on the controller, since it is not going to be used to host VM instances.

Prior to enabling KVM on a machine, it is important to make sure that the machine uses either Intel
VT or AMD-V chipsets that support hardware-assisted virtualization. This feature might be disabled
in the Basic Input Output System (BIOS); therefore, it is necessary to verify that it is enabled. To
check whether hardware-assisted virtualization is supported by the hardware, the following Linux
command can be used:

```Bash
grep -E 'vmx|svm' /proc/cpuinfo
```

If the command returns any output, it means that the supports hardware-assisted virtualization. The
`vmx` processor feature flag represents an Intel VT chipset, whereas the `svm` flag represents
AMD-V. Depending on the flag returned, you need to modify the `02-kvm-modprobe.sh` script.


(@) `01-kvm-install.sh`

This script installs KVM and the related tools.

```Bash
# Install KVM and the related tools
yum -y install kvm qemu-kvm qemu-kvm-tools
```


(@) `02-kvm-modprobe.sh`

This script enables KVM in the OS. If the `grep -E 'vmx|svm' /proc/cpuinfo` command described above
returned `vmx`, there is no need to modify this script, as it enables the Intel KVM module by
default. If the command returned `svm`, it is necessary to comment the `modprobe kvm-intel` line and
uncomment the `modprobe kvm-amd` line.

```Bash
# Create a script for enabling the KVM kernel module
echo "
modprobe kvm

# Uncomment this line if the host has an AMD CPU
#modprobe kvm-amd

# Uncomment this line if the host has an Intel CPU
modprobe kvm-intel
" > /etc/sysconfig/modules/kvm.modules

chmod +x /etc/sysconfig/modules/kvm.modules

# Enable KVM
/etc/sysconfig/modules/kvm.modules
```


(@) `03-libvirt-install.sh`

This script installs Libvirt^[http://en.wikipedia.org/wiki/Libvirt], its dependencies and the
related tools. Libvirt provides an abstraction and a common Application Programming Interface (API)
over various hypervisors. It is used by OpenStack to provide support for multiple hypervisors. After
the installation, the script starts the `messagebus` and `avahi-daemon` services, which are
prerequisites of Libvirt.

```Bash
# Install libvirt and its dependecies
yum -y install libvirt libvirt-python python-virtinst avahi dmidecode

# Start the services required by livirt
service messagebus restart
service avahi-daemon restart

# Start the service during the system start up
chkconfig messagebus on
chkconfig avahi-daemon on
```


(@) `04-libvirt-config.sh`

This script modifies the Libvirt configuration to enable communication over TCP. This is required by
OpenStack to enable live migration of VM instances.

```Bash
# Enable the communication with libvirt over TCP without
# authentication. This configuration is required to enable live
# migration through OpenStack.
sed -i 's/#listen_tls = 0/listen_tls = 0/g' \
    /etc/libvirt/libvirtd.conf
sed -i 's/#listen_tcp = 1/listen_tcp = 1/g' \
    /etc/libvirt/libvirtd.conf
sed -i 's/#auth_tcp = "sasl"/auth_tcp = "none"/g' \
    /etc/libvirt/libvirtd.conf
sed -i 's/#LIBVIRTD_ARGS="--listen"/LIBVIRTD_ARGS="--listen"/g' \
    /etc/sysconfig/libvirtd
```


(@) `05-libvirt-start.sh`

This script starts the `libvirtd` service and sets it to automatically start during the system start up.

```Bash
# Start the libvirt service
service libvirtd restart
chkconfig libvirtd on
```


### OpenStack

This section contains a few subsection describing different phases of OpenStack installation.

#### 06-openstack-all (all nodes)

The scripts described in this section need to be executed on all the hosts.


(@) `01-epel-add-repo.sh`

This scripts adds the Extra Packages for Enterprise Linux^[http://fedoraproject.org/wiki/EPEL]
(EPEL) repository, which contains the OpenStack related packages.

```Bash
# Add the EPEL repo: http://fedoraproject.org/wiki/EPEL
yum install -y \
    http://dl.fedoraproject.org/pub/epel/6/i386/epel-release-6-7.noarch.rpm
```


(@) `02-ntp-install.sh`

This script install the NTP service, which is required to automatically synchronize the time with
external NTP servers.

```Bash
# Install NTP
yum install -y ntp
```


(@) `03-ntp-config.sh`

This script replaces the default servers specified in the `/etc/ntp.conf` configuration file with
the servers specified in the `config/ntp.conf` file described above. If the default set of servers
is satisfactory, then the execution of this script is not required.

```Bash
# Fetch the NTP servers specified in ../config/ntp.conf
SERVER1=`cat ../config/ntp.conf | sed '1!d;q'`
SERVER2=`cat ../config/ntp.conf | sed '2!d;q'`
SERVER3=`cat ../config/ntp.conf | sed '3!d;q'`

# Replace the default NTP servers with the above
sed -i "s/server 0.*pool.ntp.org/$SERVER1/g" /etc/ntp.conf
sed -i "s/server 1.*pool.ntp.org/$SERVER2/g" /etc/ntp.conf
sed -i "s/server 2.*pool.ntp.org/$SERVER3/g" /etc/ntp.conf
```


(@) `04-ntp-start.sh`

This script starts the `ntpdate` service and sets it to start during the system start up. Upon the
start, the `ntpdate` service synchronizes the time with the servers specified in the `/etc/ntp.conf`
configuration file.

```Bash
# Start the NTP service
service ntpdate restart
chkconfig ntpdate on
```


#### 07-openstack-controller (controller)

The scripts described in this section need to be run only on the controller host.


(@) `01-source-configrc.sh`

This scripts is mainly used to remind of the necessity to "source" the `configrc` file prior to
continuing, since some scripts in this directory use the environmental variable defined in
`configrc`. To source the file, it is necessary to run the following command `. 01-source-configrc.sh`.

```Bash
echo "To make the environmental variables available \
    in the current session, run: "
echo ". 01-source-configrc.sh"

# Export the variables defined in ../config/configrc
. ../config/configrc
```


(@) `02-mysql-install.sh`

This script installs the MySQL server, which is required to host the databases used by the OpenStack
services.

```Bash
# Install the MySQL server
yum install -y mysql mysql-server
```


(@) `03-mysql-start.sh`

This script start the MySQL service and initializes the password of the `root` MySQL user using the
variable from the `configrc` file called `$MYSQL_ROOT_PASSWORD`.

```Bash
# Start the MySQL service
service mysqld start
chkconfig mysqld on

# Initialize the MySQL root password
mysqladmin -u root password $MYSQL_ROOT_PASSWORD

echo ""
echo "The MySQL root password has been set \
    to the value of $MYSQL_ROOT_PASSWORD: \"$MYSQL_ROOT_PASSWORD\""
```


(@) `04-keystone-install.sh`

This script installs Keystone - the OpenStack identity management service, and other OpenStack
command line utilities.

```Bash
# Install OpenStack utils and Keystone - the identity management service
yum install -y openstack-utils openstack-keystone
```


(@) `05-keystone-create-db.sh`

This script creates a MySQL database for Keystone called `keystone`, which is used to store various
identity data. The script also creates a `keystone` user and grants full permissions to the
`keystone` database to this user.

```Bash
# Create a database for Keystone
../lib/mysqlq.sh "CREATE DATABASE keystone;"

# Create a keystone user and grant all privileges to the keystone database
../lib/mysqlq.sh "GRANT ALL ON keystone.* TO 'keystone'@'controller' \
    IDENTIFIED BY '$KEYSTONE_MYSQL_PASSWORD';"
```


(@) `06-keystone-generate-admin-token.sh`

This script generates a random token used to authorize the Keystone admin account. The generated
token is stored in the `./keystone-admin-token` file.

```Bash
# Generate an admin token for Keystone and save it into
# ./keystone-admin-token
openssl rand -hex 10 > keystone-admin-token
```


(@) `07-keystone-config.sh`

This script modifies the configuration file of Keystone, `/etc/keystone/keystone.conf`. It sets the
generated admin token and the MySQL connection configuration using the variables defined in `configrc`.

```Bash
# Set the generated admin token in the Keystone configuration
openstack-config --set /etc/keystone/keystone.conf DEFAULT \
    admin_token `cat keystone-admin-token`

# Set the connection to the MySQL server
openstack-config --set /etc/keystone/keystone.conf sql \
    connection mysql://keystone:$KEYSTONE_MYSQL_PASSWORD@controller/keystone

```


(@) `08-keystone-init-db.sh`

This script initializes the `keystone` database using the `keystone-manage` command line tool. The
executed command creates tables in the database.

```Bash
# Initialize the database for Keystone
keystone-manage db_sync
```


(@) `09-keystone-permissions.sh`

This script sets restrictive permissions (640) on the Keystone configuration file, since it contains
the MySQL account credentials and the admin token. Then, the scripts sets the ownership of the
Keystone related directories to the `keystone` user and `keystone` group.

```Bash
# Set restrictive permissions on the Keystone config file
chmod 640 /etc/keystone/keystone.conf

# Set the ownership for the Keystone related directories
chown -R keystone:keystone /var/log/keystone
chown -R keystone:keystone /var/lib/keystone
```


(@) `10-keystone-start.sh`

This script starts the Keystone service and sets it to automatically start during the system start up.

```Bash
# Start the Keystone service
service openstack-keystone restart
chkconfig openstack-keystone on
```


(@) `11-keystone-create-users.sh`

The purpose of this script is to create user accounts, roles and tenants in Keystone for the admin
user and service accounts for each OpenStack service: Keystone, Glance, and Nova. Since the process
is complicated when done manually (it is necessary to define relations between database records), we
use the *keystone-init* project^[https://github.com/nimbis/keystone-init] to automate the process.
The *keystone-init* project allows one to create a configuration file in the "YAML Ain't Markup
Language"^[http://en.wikipedia.org/wiki/YAML] (YAML) data format defining the required OpenStack
user accounts. Then, according the defined configuration, the required database are automatically
created.

Our script first installs a dependency of *keystone-init* and clones the project's repository. Then,
the script modifies the default configuration file provided with the *keystone-init* project by
populating it with the values defined by the environmental variables defined in `configrc`. The last
step of the script is to invoke *keystone-init*. The script does not remove the *keystone-init*
repository to allow one to browse the generated configuration file, e.g. to check the correctness.
When the repository is not required anymore, it can be removed by executing `rm -rf keystone-init`.

```Bash
# Install PyYAML, a YAML Python library
yum install -y PyYAML

# Clone a repository with Keystone initialization scripts
git clone https://github.com/nimbis/keystone-init.git

# Replace the default configuration with the values defined be the
# environmental variables in configrc
sed -i "s/192.168.206.130/controller/g" \
    keystone-init/config.yaml
sed -i "s/012345SECRET99TOKEN012345/`cat keystone-admin-token`/g" \
    keystone-init/config.yaml
sed -i "s/name:        openstackDemo/name:        $OS_TENANT_NAME/g" \
    keystone-init/config.yaml
sed -i "s/name:     adminUser/name:     $OS_USERNAME/g" \
    keystone-init/config.yaml
sed -i "s/password: secretword/password: $OS_PASSWORD/g" \
    keystone-init/config.yaml
sed -i "s/name:     glance/name:     $GLANCE_SERVICE_USERNAME/g" \
    keystone-init/config.yaml
sed -i "s/password: glance/password: $GLANCE_SERVICE_PASSWORD/g" \
    keystone-init/config.yaml
sed -i "s/name:     nova/name:     $NOVA_SERVICE_USERNAME/g" \
    keystone-init/config.yaml
sed -i "s/password: nova/password: $NOVA_SERVICE_PASSWORD/g" \
    keystone-init/config.yaml
sed -i "s/RegionOne/$OS_REGION_NAME/g" \
    keystone-init/config.yaml

# Run the Keystone initialization script
./keystone-init/keystone-init.py ./keystone-init/config.yaml

echo ""
echo "The applied config file is keystone-init/config.yaml"
echo "You may do 'rm -rf keystone-init' to remove \
    the no more needed keystone-init directory"
```


(@) `12-glance-install.sh`

This script install Glance -- the OpenStack VM image management service.

```Bash
# Install OpenStack Glance -- an image management service
yum install -y openstack-glance
```


(@) `13-glance-create-db.sh`

This script creates a MySQL database for Glance called `glance`, which is used to store VM image
metadata. The script also creates a `glance` user and grants full permissions to the `glance`
database to this user.

```Bash
# Create a database for Glance
../lib/mysqlq.sh "CREATE DATABASE glance;"

# Create a glance user and grant all privileges
# to the glance database
../lib/mysqlq.sh "GRANT ALL ON glance.* TO 'glance'@'controller' \
    IDENTIFIED BY '$GLANCE_MYSQL_PASSWORD';"
```


(@) `14-glance-config.sh`

This scripts modifies the configuration files of the Glance services, which include the API and
Registry services. Apart from various credentials, the script also sets Keystone as the identity
management service used by Glance.

```Bash
# Make Glance API use Keystone as the identity management service
openstack-config --set /etc/glance/glance-api.conf paste_deploy \
    flavor keystone

# Set Glance API user credentials
openstack-config --set /etc/glance/glance-api-paste.ini filter:authtoken \
    admin_tenant_name $GLANCE_SERVICE_TENANT
openstack-config --set /etc/glance/glance-api-paste.ini filter:authtoken \
    admin_user $GLANCE_SERVICE_USERNAME
openstack-config --set /etc/glance/glance-api-paste.ini filter:authtoken \
    admin_password $GLANCE_SERVICE_PASSWORD

# Set Glance Cache user credentials
openstack-config --set /etc/glance/glance-cache.conf DEFAULT \
    admin_tenant_name $GLANCE_SERVICE_TENANT
openstack-config --set /etc/glance/glance-cache.conf DEFAULT \
    admin_user $GLANCE_SERVICE_USERNAME
openstack-config --set /etc/glance/glance-cache.conf DEFAULT \
    admin_password $GLANCE_SERVICE_PASSWORD

# Make Glance Registry use Keystone as the identity management service
openstack-config --set /etc/glance/glance-registry.conf paste_deploy \
    flavor keystone

# Set the connection to the MySQL server
openstack-config --set /etc/glance/glance-registry.conf DEFAULT \
    sql_connection mysql://glance:$GLANCE_MYSQL_PASSWORD@controller/glance

# Set Glance Registry user credentials
openstack-config --set /etc/glance/glance-registry-paste.ini filter:authtoken \
    admin_tenant_name $GLANCE_SERVICE_TENANT
openstack-config --set /etc/glance/glance-registry-paste.ini filter:authtoken \
    admin_user $GLANCE_SERVICE_USERNAME
openstack-config --set /etc/glance/glance-registry-paste.ini filter:authtoken \
    admin_password $GLANCE_SERVICE_PASSWORD
```


(@) `15-glance-init-db.sh`

This scripts initializes the `glance` database using the `glance-manage` command line tool.

```Bash
# Initialize the database for Glance
glance-manage db_sync
```


(@) `16-glance-permissions.sh`

This scripts sets restrictive permissions (640) on the Glance configuration files, since they
contain sensitive information. The script also set the ownership of the Glance related directories
to the `glance` user and `glance` group.

```Bash
# Set restrictive permissions for the Glance config files
chmod 640 /etc/glance/*.conf
chmod 640 /etc/glance/*.ini

# Set the ownership for the Glance related directories
chown -R glance:glance /var/log/glance
chown -R glance:glance /var/lib/glance
```


(@) `17-glance-start.sh`

This script starts the Glance services: both API and Registry. The script sets the services to
automatically start during the system start up.

```Bash
# Start the Glance Registry and API services
service openstack-glance-registry restart
service openstack-glance-api restart

chkconfig openstack-glance-registry on
chkconfig openstack-glance-api on
```


(@) `18-add-cirros.sh`

This script downloads the Cirros VM image^[https://launchpad.net/cirros/] and imports it into
Glance. This image is very simplistic: its size is just 9.4 MB; however, it is sufficient for
testing OpenStack.

```Bash
# Download the Cirros VM image
mkdir /tmp/images
cd /tmp/images
wget https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img

# Add the downloaded image to Glance
glance add name="cirros-0.3.0-x86_64" is_public=true \
    disk_format=qcow2 container_format=bare < cirros-0.3.0-x86_64-disk.img

# Remove the temporary directory
rm -rf /tmp/images
```


(@) `19-add-ubuntu.sh`

This script download the Ubuntu Cloud Image^[http://uec-images.ubuntu.com/] and imports it into
Glance. This is a VM image with a pre-installed version of Ubuntu that is customized by Ubuntu
engineering to run on cloud-platforms such as Openstack, Amazon EC2, and LXC.

```Bash
# Download an Ubuntu Cloud image
mkdir /tmp/images
cd /tmp/images
wget http://uec-images.ubuntu.com/precise/current/precise-server-cloudimg-amd64-disk1.img

# Add the downloaded image to Glance
glance add name="ubuntu" is_public=true disk_format=qcow2 \
    container_format=bare < precise-server-cloudimg-amd64-disk1.img

# Remove the temporary directory
rm -rf /tmp/images
```


(@) `20-nova-install.sh`

This script install Nova -- the OpenStack compute service, as well as the Qpid AMQP message broker.
The message broker is required by the OpenStack services to communicate with each other.

```Bash
# Install OpenStack Nova (compute service)
# and the Qpid AMQP message broker
yum install -y openstack-nova* qpid-cpp-server
```


(@) `21-nova-create-db.sh`

This script creates a MySQL database for Nova called `nova`, which is used to store VM instance
metadata. The script also creates a `nova` user and grants full permissions to the `nova` database
to this user. The script also enables the access to the database from hosts other than controller.

```Bash
# Create a database for Nova
../lib/mysqlq.sh "CREATE DATABASE nova;"

# Create a nova user and grant all privileges
# to the nova database
../lib/mysqlq.sh "GRANT ALL ON nova.* TO 'nova'@'controller' \
    IDENTIFIED BY '$NOVA_MYSQL_PASSWORD';"

# The following is need to allow access
# from Nova services running on other hosts
../lib/mysqlq.sh "GRANT ALL ON nova.* TO 'nova'@'%' \
    IDENTIFIED BY '$NOVA_MYSQL_PASSWORD';"
```


(@) `22-nova-permissions.sh`

This script sets restrictive permissions on the Nova configuration file, since it contains sensitive
information, such as user credentials. The script also sets the ownership of the Nova related
directories to the `nova` group.

```Bash
# Set restrictive permissions for the Nova config file
chmod 640 /etc/nova/nova.conf

# Set the ownership for the Nova related directories
chown -R root:nova /etc/nova
chown -R nova:nova /var/lib/nova
```


(@) `23-nova-config.sh`

This scripts invokes the Nova configuration script provided in the `lib` directory, since it is
shared by the scripts setting up Nova on all the controller, and the compute hosts.

```Bash
# Run the Nova configuration script
# defined in ../lib/nova-config.sh
../lib/nova-config.sh
```

The content of the `nova-config.sh` script is given below:

```Bash
# This is a Nova configuration shared
# by the compute hosts, gateway and controller

# Enable verbose output
openstack-config --set /etc/nova/nova.conf DEFAULT \
    verbose True

# Set the connection to the MySQL server
openstack-config --set /etc/nova/nova.conf DEFAULT \
    sql_connection mysql://nova:$NOVA_MYSQL_PASSWORD@controller/nova

# Make Nova use Keystone as the identity management service
openstack-config --set /etc/nova/nova.conf DEFAULT \
    auth_strategy keystone

# Set the host name of the Qpid AMQP message broker
openstack-config --set /etc/nova/nova.conf DEFAULT \
    qpid_hostname controller

# Set Nova user credentials
openstack-config --set /etc/nova/api-paste.ini filter:authtoken \
    admin_tenant_name $NOVA_SERVICE_TENANT
openstack-config --set /etc/nova/api-paste.ini filter:authtoken \
    admin_user $NOVA_SERVICE_USERNAME
openstack-config --set /etc/nova/api-paste.ini filter:authtoken \
    admin_password $NOVA_SERVICE_PASSWORD
openstack-config --set /etc/nova/api-paste.ini filter:authtoken \
    auth_uri $NOVA_OS_AUTH_URL

# Set the network configuration
openstack-config --set /etc/nova/nova.conf DEFAULT \
    network_host compute1
openstack-config --set /etc/nova/nova.conf DEFAULT \
    fixed_range 10.0.0.0/24
openstack-config --set /etc/nova/nova.conf DEFAULT \
    flat_interface eth1
openstack-config --set /etc/nova/nova.conf DEFAULT \
    flat_network_bridge br100
openstack-config --set /etc/nova/nova.conf DEFAULT \
    public_interface eth1

# Set the Glance host name
openstack-config --set /etc/nova/nova.conf DEFAULT \
    glance_host controller

# Set the VNC configuration
openstack-config --set /etc/nova/nova.conf DEFAULT \
    vncserver_listen 0.0.0.0
openstack-config --set /etc/nova/nova.conf DEFAULT \
    vncserver_proxyclient_address controller

# This is the host accessible from outside,
# where novncproxy is running on
openstack-config --set /etc/nova/nova.conf DEFAULT \
    novncproxy_base_url http://$PUBLIC_IP_ADDRESS:6080/vnc_auto.html

# This is the host accessible from outside,
# where xvpvncproxy is running on
openstack-config --set /etc/nova/nova.conf DEFAULT \
    xvpvncproxy_base_url http://$PUBLIC_IP_ADDRESS:6081/console

# Set the host name of the metadata service
openstack-config --set /etc/nova/nova.conf DEFAULT \
    metadata_host $METADATA_HOST
```

Apart from user credentials, the script configures a few other important options:

- the identity management service -- Keystone;
- the Qpid server host name -- controller;
- the host running the Nova network service -- compute1 (i.e. gateway);
- the network used for VMs -- 10.0.0.0/24;
- the network interface used to bridge VMs to -- eth1;
- the Linux bridge used by VMs -- br100;
- the public network interface -- eth1;
- the Glance service host name -- controller;
- the VNC server host name -- controller;
- the IP address of the host running VNC proxies (they must be run on the host that can be accessed
  from outside; in our setup it is gateway) -- `$PUBLIC_IP_ADDRESS`;
- the Nova metadata service host name -- controller.


(@) `24-nova-init-db.sh`

This scripts initializes the `nova` database using the `nova-manage` command line tool.

```Bash
# Initialize the database for Nova
nova-manage db sync
```


(@) `25-nova-start.sh`

This script starts various Nova services, as well as their dependencies: the Qpid AMQP message
broker, and iSCSI target daemon used by the `nova-volume` service.

```Bash
# Start the Qpid AMQP message broker
service qpidd restart

# iSCSI target daemon for nova-volume
service tgtd restart

# Start OpenStack Nova services
service openstack-nova-api restart
service openstack-nova-cert restart
service openstack-nova-consoleauth restart
service openstack-nova-direct-api restart
service openstack-nova-metadata-api restart
service openstack-nova-scheduler restart
service openstack-nova-volume restart

# Make the service start on the system startup
chkconfig qpidd on
chkconfig tgtd on
chkconfig openstack-nova-api on
chkconfig openstack-nova-cert on
chkconfig openstack-nova-consoleauth on
chkconfig openstack-nova-direct-api on
chkconfig openstack-nova-metadata-api on
chkconfig openstack-nova-scheduler on
chkconfig openstack-nova-volume on
```


#### 08-openstack-compute (compute nodes)

The scripts described in this section should be run on the compute hosts.

(@) `01-source-configrc.sh`

This scripts is mainly used to remind of the necessity to "source" the `configrc` file prior to
continuing, since some scripts in this directory use the environmental variable defined in
`configrc`. To source the file, it is necessary to run the following command `. 01-source-configrc.sh`.

```Bash
echo "To make the environmental variables available \
    in the current session, run: "
echo ". 01-source-configrc.sh"

# Export the variables defined in ../config/configrc
. ../config/configrc
```


(@) `02-install-nova.sh`

This script installs OpenStack Nova and OpenStack utilities.

```Bash
# Install OpenStack Nova and utils
yum install -y openstack-nova* openstack-utils
```


(@) `03-nova-permissions.sh`

This script sets restrictive permissions (640) on the Nova configuration file, since it contains
sensitive information, such as user credentials. Then, the script sets the ownership on the Nova and
Libvirt related directories to the `nova` user and `nova` group. The script also sets the user and
group used by the Quick EMUlator^[http://en.wikipedia.org/wiki/QEMU] (QEMU) service to `nova`. This
is required since a number of directories need to accessed by both Nova using the `nova` user and
`nova` group, and QEMU.

```Bash
# Set restrictive permissions for the Nova config file
chmod 640 /etc/nova/nova.conf

# Set the ownership for the Nova related directories
chown -R root:nova /etc/nova
chown -R nova:nova /var/lib/nova
chown -R nova:nova /var/cache/libvirt
chown -R nova:nova /var/run/libvirt
chown -R nova:nova /var/lib/libvirt

# Make Qemu run under the nova user and group
sed -i 's/#user = "root"/user = "nova"/g' /etc/libvirt/qemu.conf
sed -i 's/#group = "root"/group = "nova"/g' /etc/libvirt/qemu.conf
```


(@) `04-nova-config.sh`

This scripts invokes the Nova configuration script provided in the `lib` directory, which has been
detailed above.

```Bash
# Run the Nova configuration script
# defined in ../lib/nova-config.sh
../lib/nova-config.sh
```


(@) `05-nova-compute-start.sh`

First, this script restarts the Libvirt service since its configuration has been modified. Then, the
script starts Nova compute service and sets it to automatically start during the system start up.

```Bash
# Start the libvirt and Nova services
service libvirtd restart
service openstack-nova-compute restart
chkconfig openstack-nova-compute on
```


#### 09-openstack-gateway (network gateway)

This scripts described in this section need to be run only on the gateway.


(@) `01-source-configrc.sh`

This scripts is mainly used to remind of the necessity to "source" the `configrc` file prior to
continuing, since some scripts in this directory use the environmental variable defined in
`configrc`. To source the file, it is necessary to run the following command `. 01-source-configrc.sh`.

```Bash
echo "To make the environmental variables available \
    in the current session, run: "
echo ". 01-source-configrc.sh"

# Export the variables defined in ../config/configrc
. ../config/configrc
```


(@) `02-nova-start.sh`

It is assumed that the gateway host is one of the compute hosts; therefore, the OpenStack compute
service has already been configured and is running. This scripts starts 3 additional Nova services
that are specific to the gateway host: `openstack-nova-network`, `openstack-nova-novncproxy`, and
`openstack-nova-xvpvncproxy`. The `openstack-nova-network` service is responsible for bridging VM
instances into the physical network, and configuring the
Dnsmasq^[http://en.wikipedia.org/wiki/Dnsmasq] service for assigning IP addresses to the VMs. The
VNC proxy services enable VNC connections to VM instances from the outside network; therefore, they
must be run on a machine that has access to the public network, which is the gateway in our case.

```Bash
# Start the libvirt and Nova services
# (network, compute and VNC proxies)
service libvirtd restart
service openstack-nova-network restart
service openstack-nova-compute restart
service openstack-nova-novncproxy restart
service openstack-nova-xvpvncproxy restart

# Make the service start on the system start up
chkconfig openstack-nova-network on
chkconfig openstack-nova-compute on
chkconfig openstack-nova-novncproxy on
chkconfig openstack-nova-xvpvncproxy on
```


(@) `03-nova-network-create.sh`

This service creates a Nova network 10.0.0.0/24, which is used to allocate IP addresses from by
Dnsmasq to VM instances. The created network is configured to use the `br100` Linux bridge to
connect VM instances to the physical network.

```Bash
# Create a Nova network for VM instances: 10.0.0.0/24
nova-manage network create --label=public --fixed_range_v4=10.0.0.0/24 \
    --num_networks=1 --network_size=256 --bridge=br100
```


(@) `04-nova-secgroup-add.sh`

This script adds two rules to the default OpenStack security group. The first rule enables the
Internet Control Message Protocol (ICMP) for VM instances (the ping command). The second rule
enables TCP connection via the 22 port, which is used by SSH.

```Bash
# Enable ping for VMs
nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0

# Enable SSH for VMs
nova secgroup-add-rule default tcp 22 22 0.0.0.0/0
```


(@) `05-dashboard-install.sh`

This script installs the OpenStack dashboard. The OpenStack dashboard provides a web-interface to
managing an OpenStack environment. Since the dashboard is supposed to be accessed from outside, this
service must be installed on a host that has access to the public network, which is the gateway in
our setup.

```Bash
# Install OpenStack Dashboard
yum install -y openstack-dashboard
```


(@) `06-dashboard-config.sh`

This script configures the OpenStack dashboard. Particularly, the script sets the
`OPENSTACK_HOST` configuration option denoting the host name of the management host to `controller`.
The script also sets the default Keystone role to the value of the `$OS_TENANT_NAME` environmental
variable.

```Bash
# Set the OpenStack management host
sed -i 's/OPENSTACK_HOST = "127.0.0.1"/OPENSTACK_HOST = "controller"/g' \
    /etc/openstack-dashboard/local_settings

# Set the Keystone default role
sed -i "s/OPENSTACK_KEYSTONE_DEFAULT_ROLE = \"Member\"/OPENSTACK_KEYSTONE_DEFAULT_ROLE = \"$OS_TENANT_NAME\"/g" \
    /etc/openstack-dashboard/local_settings
```


(@) `07-dashboard-start.sh`

This script starts the httpd service, which is a web server configured to serve the OpenStack
dashboard. The script also sets the httpd service to start automatically during the system start up.
Once the service is started, the dashboard will be available at `http://localhost/dashboard`, where
'localhost' should be replaced by the public IP address of the gateway host for accessing the
dashboard from the outside network.

```Bash
# Start the httpd service.
service httpd restart
chkconfig httpd on
```

At this point the installation of OpenStack can be considered completed. The next steps are only for
testing the environment.


#### 10-openstack-controller (controller)

The scripts described in this section need to be run only on the controller. These scripts are not a
part of the installation process and are only used for testing the correctness of the performed
OpenStack installation.


(@) `01-source-configrc.sh`

This scripts is mainly used to remind of the necessity to "source" the `configrc` file prior to
continuing, since some scripts in this directory use the environmental variable defined in
`configrc`. To source the file, it is necessary to run the following command `. 01-source-configrc.sh`.

```Bash
echo "To make the environmental variables available \
    in the current session, run: "
echo ". 01-source-configrc.sh"

# Export the variables defined in ../config/configrc
. ../config/configrc
```


(@) `02-boot-cirros.sh`

This script creates a VM instance using the Cirros image added to Glance previously.

```Bash
# Create a VM instance from the Cirros image
nova boot --image cirros-0.3.0-x86_64 --flavor m1.small cirros
```


(@) `03-keypair-add.sh`

This script creates a key pair, which is injected by OpenStack into VMs to allow password-less SSH
connections. The generated private key is save into the `../config/test.pem` file.

```Bash
# Create a key pair
nova keypair-add test > ../config/test.pem
chmod 600 ../config/test.pem
```


(@) `04-boot-ubuntu.sh`

This script creates a VM instance using the Ubuntu Cloud image added to Glance previously. The
executed command instructs OpenStack to inject the previously generated public key called `test` to
allow password-less SSH connections.

```Bash
# Create a VM instance from the Ubuntu Cloud image
nova boot --image ubuntu --flavor m1.small --key_name test ubuntu
```


(@) `05-ssh-into-vm.sh`

This script shows how to SSH into a VM instance, which has been injected with the previously
generated `test` key. The script accepts one argument: the IP address of the VM instance.

```Bash
# SSH into a VM instance using the generated test.pem key.

if [ $# -ne 1 ]
then
    echo "You must specify one arguments - \
	    the IP address of the VM instance"
    exit 1
fi

ssh -i ../config/test.pem -l test $1
```


(@) `06-nova-volume-create.sh`

This script shows how to create a 2 GB Nova volume called `myvolume`. Once created, the volume can
be dynamically attached to a VM instance, as shown in the next script.

```Bash
# Create a 2GB volume called myvolume
nova volume-create --display_name myvolume 2
```


(@) `07-nova-volume-attach.sh`

This script shows how to attached a volume to a VM instance. The script accepts two arguments: (1)
the name of the VM instance to attach the volume to; and (2) the ID of the volume to attach to the
VM instance. Once attached, the volume will be available inside the VM instance as the `/dev/vdc/
device. The volume is provided as a block storage, which means it has be formatted before it can be
used.

```Bash
# Attach the created volume to a VM instance as /dev/vdc.

if [ $# -ne 2 ]
then
    echo "You must specify two arguments:"
    echo "(1) the name of the VM instance"
    echo "(2) the ID of the volume to attach"
    exit 1
fi

nova volume-attach $1 $2 /dev/vdc
```


## OpenStack Troubleshooting

# Conclusion

# References