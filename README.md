# Linaro's Automated Validation Architecture (LAVA) Docker Container

## Introduction

The goal of lava-docker is to simplify the install and maintenance of
a LAVA lab in order to participate in distributed test efforts such as
kernelCI.org.

With lava-docker, you describe the devices under test (DUT) in a
simple YAML file, and then a custom script will generate the necessary
LAVA configuration files automatically.

Similarly, LAVA users and authentication tokens are described in
a(nother) YAML file, and the LAVA configurations are automatically generated.

This enables the setup of a LAVA lab with minimal knowledge of the
underlying LAVA configuration steps necessary.

## Prerequisites
lava-docker has currently been tested primarily on Debian stable (stretch).
The following packages are necessary on the host machine:
* docker
* docker-compose
* pyyaml

## Quickstart
Example to use lava-docker with only one QEMU device:

* Checkout the lava-docker repository
* Generate configuration files for LAVA, udev, serial ports, etc. from boards.yaml via
```
./lavalab-gen.py
```
* Go to output/local directory
* Build docker images via
```
docker-compose build
```
* Start all images via
```
docker-compose up -d
```

* Once launched, you can access the LAVA web interface via http://localhost:10080/.
With the default users, you can login with admin:admin.

* By default, a LAVA healthcheck job will be run on the qemu device.
You will see it in the "All Jobs" list: http://localhost:10080/scheduler/alljobs

* You can also see full job output by clicking the blue eye icon ("View job details") (or via http://localhost:10080/scheduler/job/1 since it is the first job ran)

* For more details, see https://validation.linaro.org/static/docs/v2/first-job.html

### Adding your first board:
#### device-type
To add a board you need to find its device-type, standard naming is to use the same as the official kernel DT name.
(But a very few DUT differ from that)

You could check in https://github.com/Linaro/lava-server/tree/release/lava_scheduler_app/tests/device-types if you find yours.

Example:
For a beagleboneblack, the device-type is beaglebone-black (Even if official DT name is am335x-boneblack)
So you need to add in the boards section:
```
    - name: beagleboneblack-01
      type: beaglebone-black
```

#### UART
Next step is to gather information on UART wired on DUT.<br>
If you have a FTDI, simply get its serial (visible in lsusb -v or for major distribution in dmesg)<br>
<br>
For other UART type (or for old FTDI without serial number) you need to get the devpath attribute via:
```
udevadm info -a -n /dev/ttyUSBx |grep ATTR|grep devpath | head -n1
```
Example with a FTDI UART:
```
[    6.616707] usb 4-1.4.2: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[    6.704305] usb 4-1.4.2: SerialNumber: AK04TU1X
The serial is AK04TU1X
```
So you have now:
```
    - name: beagleboneblack-01
      type: beaglebone-black
      uart:
        idvendor: 0x0403
	idproduct: 0x6001
	serial: AK04TU1X
```

Example with a FTDI without serial:
```
[2428401.256860] ftdi_sio 1-1.4:1.0: FTDI USB Serial Device converter detected
[2428401.256916] usb 1-1.4: Detected FT232BM
[2428401.257752] usb 1-1.4: FTDI USB Serial Device converter now attached to ttyUSB1
udevadm info -a -n /dev/ttyUSB1 |grep devpath | head -n1
    ATTRS{devpath}=="1.5"
```
So you have now:
```
    - name: beagleboneblack-01
      type: beaglebone-black
      uart:
        idvendor: 0x0403
	idproduct: 0x6001
	devpath: "1.5"
```

#### PDU (Power Distribution Unit)
Final step is to manage the powering of the board.<br>
Many PDU switchs could be handled by a command line tool which control the PDU.<br>
You need to fill boards.yaml with the command line to be ran.<br>

Example with an ACME board:
If the beagleboneblack is wired to port 3 and the ACME board have IP 192.168.66.2:
```
      pdu_generic:
        hard_reset_command: /usr/local/bin/acme-cli -s 192.168.66.2 reset 3
        power_off_command: /usr/local/bin/acme-cli -s 192.168.66.2 power_off 3
        power_on_command: /usr/local/bin/acme-cli -s 192.168.66.2 power_on 3
```

#### Example:
beagleboneblack, with FTDI (serial 1234567), connected to port 5 of an ACME
```
    - name: beagleboneblack-01
      type: beaglebone-black
      pdu_generic:
        hard_reset_command: /usr/local/bin/acme-cli -s 192.168.66.2 reset 5
        power_off_command: /usr/local/bin/acme-cli -s 192.168.66.2 power_off 5
        power_on_command: /usr/local/bin/acme-cli -s 192.168.66.2 power_on 5
      uart:
        idvendor: 0x0403
	idproduct: 0x6001
	serial: 1234567
```

## Architecture
The basic setup is composed of a host which runs the following docker images and DUT to be tested.<br/>
* lava-master: run lava-server along with the web interface
* lava-slave: run lava-dispatcher, the compoment which sends jobs to DUTs

The host and DUTs must share a common LAN.<br/>
The host IP on this LAN must be set as dispatcher_ip in boards.yaml.<br/>

Since most DUTs are booted using TFTP, they need DHCP for gaining network connectivity.<br/>
So, on the LAN shared with DUTs, a running DHCPD is necessary. (See DHCPD below)<br/>

![lava-docker diagram](doc/lava-docker.png)

## Multi-host architectures
Lava-docker support multi-host architecture, Master and slaves could be on different host.

Lava-docker support multiples slaves, but with a maximum of one slave per host.
This is due to that slave need TFTP port accessible from outside.

### Power supply
You need to have a PDU for powering your DUT.
Managing PDUs is done via pdu_generic

### Network ports
The following ports are used by lava-docker and are proxyfied on the host:
- 69/UDP	proxyfied to the slave for TFTP
- 80		proxyfied to the slave for TODO (transfer overlay)
- 5500		proxyfied to the slave for Notification
- 5555		proxyfied to the master (LAVA logger)
- 5556		proxyfied to the master (LAVA master)
- 10080		proxyfied to the master (Web interface)
- 55950-56000	proxyfied to the slave for NBD

### DHCPD
A DHCPD service is necessary for giving network access to DUT.

The DHCPD server could be anywhere with the condition that it is accessible of DUTs. (Could be on host, in a docker in the host, or is the ISP box on the same LAN.<br/>

### Examples
#### Example 1: Basic LAB with home router
Router: 192.168.1.1 which handle DHCP for 192.168.1.10-192.168.1.254<br>
Lab: 192.168.1.2<br>

So the dispatcher_ip is set to 192.168.1.2

#### Example 2: Basic LAB without home router
Lab: 192.168.1.2 which handle DHCP for 192.168.1.10-192.168.1.254<br>

So the dispatcher_ip is set to 192.168.1.2

#### Example 3: LAB with dedicated LAN for DUTs
A dedicated LAN is used for DUTs. (192.168.66.0/24)
The host have two NIC:
- eth0: (192.168.1.0/24) on home LAN. (The address could be static or via DHCP)
- eth1: (192.168.66.0/24) with address set to 192.168.66.1

On the host, a DHCPD give address in range of 192.168.66.3-192.168.66.200

So the dispatcher_ip is set to 192.168.66.1

#### DHCPD examples:
##### isc-dhcpd-server
A sample isc-dhcpd-server DHCPD config file is available in the dhcpd directory.<br/>
##### dnsmasq
Simply set interface=interfacename where interfacename is your shared LAN interface

## Generating files

### Helper script
You can use the lavalab-gen.sh helper script which will do all the above actions for you.

### boards.yaml
This file describe how the DUTs are connected and powered.
```
masters:
 - name:  lava-master	name of the master
    host: name		name of the host running lava-master (default to "local")
    webadmin_https:	Does the LAVA webadmin is accessed via https
    zmq_auth: True/False	Does the master requires ZMQ authentication.
    zmq_auth_key:		optional path to a public ZMQ key
    zmq_auth_key_secret:	optional path to a private ZMQ key
    slave_keys:			optional path to a directory with slaves public key. Usefull when you want to create a master without slaves nodes in boards.yaml.
    lava-coordinator:		Does the master should ran a lava-coordinator and export its port
    persistent_db: True/False	(default False) Is the postgres DB is persistent over reboot
    http_fqdn:			The FQDN used to access the LAVA web interface. This is necessary if you use https otherwise you will issue CSRF errors.
    healthcheck_url:		Hack healthchecks hosting URL. See hosting healthchecks below
    allowed_hosts:		A list of FQDN used to access the LAVA master
    - "fqdn1"
    - "fqdn2"
    loglevel:
      lava-logs: DEBUG/INFO/WARN/ERROR			(optional) select the loglevel of lava-logs (default to DEBUG)
      lava-slave: DEBUG/INFO/WARN/ERROR			(optional) select the loglevel of lava-slave (default to DEBUG)
      lava-master: DEBUG/INFO/WARN/ERROR		(optional) select the loglevel of lava-master (default to DEBUG)
      lava-server-gunicorn: DEBUG/INFO/WARN/ERROR	(optional) select the loglevel of lava-server-gunicorn (default to DEBUG)
    users:
    - name: LAVA username
      token: The token of this user 	(optional)
      password: Password the this user (generated if not provided)
      email:	email of the user	(optional)
      superuser: yes/no (default no)
      staff: yes/no (default no)
      groups:
      - name: 			Name of the group this user should join
    groups:
    - name: 			LAVA group name
      submitter: True/False	Can this group can submit jobs
    tokens:
    - username: The LAVA user owning the token below. (This user should be created via users:)
      token: The token for this callback
      description: The description of this token. This string could be used with LAVA-CI.
    slaveenv:			A list of environment to pass to slave
      - name: slavename		The name of slave (mandatory)
        env:
	- line1			A list of line to set as environment
	- line2
slaves:
  - name: lab-slave-XX		The name of the slave (where XX is a number)
    host: name			name of the host running lava-slave-XX (default to "local")
    zmq_auth_key:		optional path to a public ZMQ key
    zmq_auth_key_secret:	optional path to a private ZMQ key
    zmq_auth_master_key:	optional path to the public master ZMQ key. This option is necessary only if no master node exists in boards.yaml.
    dispatcher_ip: 		the IP where the slave could be contacted. In lava-docker it is the host IP since docker proxify TFTP from host to the slave.
    remote_master: 		the name of the master to connect to
    remote_address: 		the FQDN or IP address of the master (if different from remote_master)
    remote_rpc_port: 		the port used by the LAVA RPC2 (default 80)
    remote_user: 		the user used for connecting to the master
    remote_user_token:		The remote_user's token. This option is necessary only if no master node exists in boards.yaml. Otherwise lavalab-gen.py will get from it.
    remote_proto:		http(default) or https
    default_slave:		Does this slave is the default slave where to add boards (default: lab-slave-0)
    bind_dev:			Bind /dev from host to slave. This is needed when using some HID PDU
    use_nfs:			Does the LAVA dispatcher will run NFS jobs
    use_tap:			Does TAP netdevices could be used
    arch:			The arch of the worker (if not x86_64), only accept arm64
    host_healthcheck:		If true, enable the optional healthcheck container. See hosting healthchecks below
    lava-coordinator:		Does the slave should ran a lava-coordinator
    expose_ser2net:		Do ser2net ports need to be available on host
    expose_ports:		Expose port p1 on the host to p2 on the worker slave.
      - p1:p2
    extra_actions:		An optional list of action to do at end of the docker build
    - "apt-get install package"
    env:
      - line1			A list of line to set as environment (See /etc/lava-server/env.yaml for examples)
      - line2
    devices:			A list of devices which need UDEV rules
      - name:			The name of the device
        vendorid:		The VID of the UART (Formated as 0xXXXX)
        productid:		the PID of the UART (Formated as 0xXXXX)
        serial:			The serial number of the device if the device got one
        devpath:		The UDEV devpath to this device if more than one is present

boards:
  - name: devicename	Each board must be named by their device-type as "device-type-XX" (where XX is a number)
    type: the LAVA device-type of this device
    slave:		(optional) Name of the slave managing this device. Default to first slave found or default_slave if set.
    kvm: (For qemu only) Does the qemu could use KVM (default: no)
    uboot_ipaddr:	(optional) a static IP to set in uboot
    uboot_macaddr:	(Optional) the MAC address to set in uboot
    custom_option:	(optional) All following strings will be directly append to devicefile
    - "set x=1"
    tags:		(optional) List of tag to set on this device
    - tag1
    - tag2
    aliases:		(optional) List of aliases to set on the DEVICE TYPE.
    - alias1
    - alias2
    user:		(optional) Name of user owning the board (LAVA default is admin) user is exclusive with group
    group:		(optional) Name of group owning the board (no LAVA default) group is exclusive with user
# One of uart or connection_command must be choosen
    uart:
      idvendor: The VID of the UART (Formated as 0xXXXX)
      idproduct: the PID of the UART (Formated as 0xXXXX)
      serial: The serial number in case of FTDI uart
      devpath: the UDEV devpath to this uart for UART without serial number
      interfacenum:	(optional) The interfacenumber of the serial. (Used with two serial in one device)
      use_conmux:	True/False (Use conmux-console instead of ser2net)
      use_ser2net: 	True/False (Deprecated, ser2net is the default uart handler)
      ser2net_options	(optional) A list of ser2net options to add
        - option1
        - option2
      use_screen: 	True/False (Use screen via ssh instead of ser2net)
    connection_command: A command to be ran for getting a serial console
    pdu_generic:
      hard_reset_command: commandline to reset the board
      power_off_command: commandline to power off the board
      power_on_command: commandline to power on the board
```
Notes on UART:
* Only one of devpath/serial is necessary.
* screen usage is discouraged and should not be used, it was added as a workaround for some boards, but ser2net now can handle them.
* For finding the right devpath, you could use
```
udevadm info -a -n /dev/ttyUSBx |grep devpath | head -n1
```
* VID and PID could be found in lsusb. If a leading zero is present, the value must be given between double-quotes (and leading zero must be kept)
Example:
```
Bus 001 Device 054: ID 0403:6001 Future Technology Devices International, Ltd FT232 Serial (UART) IC
```
This device must use "0403" for idvendor and 6001 for idproduct.

Note on connection_command: connection_command is for people which want to use other custom way than ser2net to handle the console.

Examples: see [boards.yaml.example](boards.yaml.example)

### Generate
```
lavalab-gen.py
```

this script will generate all necessary files in the following locations:
```
output/host/lava-master/tokens/			This is where the callback tokens will be generated
output/host/lava-master/users/			This is where the users will be generated
output/host/lab-slave-XX/conmux/		All files needed by conmux
output/host/lab-slave-XX/devices/		All LAVA devices files
output/host/udev/99-lavaworker-udev.rules 	udev rules for host
output/host/docker-compose.yml			Generated from docker-compose.template
```

All thoses file (except for udev-rules) will be handled by docker.

You can still hack after all generated files.

#### udev rules
Note that the udev-rules are generated for the host, they must be placed in /etc/udev/rules.d/
They are used for giving a proper /dev/xxx name to tty devices. (where xxx is the board name)
(lavalab-gen.sh will do it for you)

### Building
To build all docker images, execute the following from the directory you cloned the repo:

```
docker-compose build
```

### Running
For running all images, simply run:
```
docker-compose up -d
```

## Proxy cache (Work in progress)
A squid docker is provided for caching all LAVA downloads (image, dtb, rootfs, etc...)<br/>
You have to uncomment a line in lava-master/Dockerfile to enable it.<br/>
For the moment, it is unsupported and unbuilded.

## Backporting LAVA patches
All upstream LAVA patches could be backported by placing them in lava-master/lava-patch/

## Backups / restore
For backupping a running docker, the "backup.sh" script could be used.
It will store boards.yaml + postgresql database backup + joboutputs.

For restoring a backup, postgresql database backup + joboutputs must be copied in master backup directory before build.

Example:
./backup.sh
This produce a backup-20180704_1206 directory
For restoring this backup, simply cp backup-20180704_1206/* output/local/master/backup/

## Upgrading from a previous lava-docker
For upgrading between two LAVA version, the only method is:
- backup data by running ./backup.sh on the host running the master (See Backups / restore)
- checkout the new lava-docker and your boards.yaml
- run lavalab-gen.sh
- copy your backup data in output/yourhost/master/backup directory
- build and run docker-compose

## Security
Note that this container provides defaults which are unsecure. If you plan on deploying this in a production enviroment please consider the following items:

  * Changing the default admin password (in tokens.taml)
  * Using HTTPS
  * Re-enable CSRF cookie (disabled in lava-master/Dockerfile)

## Non amd64 build
Since LAVA upstream provides only amd64 and arm64 debian packages, lava-docker support only thoses architectures.
For building an arm64 lava-docker, some little trick are necesssary:
- replace "baylibre/lava-xxxx-base" by "baylibre/lava-xxxx-base-arm64" for lava-master and lava-slave dockerfiles

For building lava-xxx-base images
- replace "bitnami/minideb" by "arm64v8/debian" on lava-master-base/lava-slave-base dockerfiles.

# How to ran NFS jobs
You need to se use_nfs: True on slave that will ran NFS jobs.
A working NFS server must be working on the host.
Furthermore, you must create a /var/lib/lava/dispatcher/tmp directory on the host and export it like:
/var/lib/lava/dispatcher/tmp 192.168.66.0/24(no_root_squash,rw,no_subtree_check)

## How to add custom LAVA patchs
You can add custom or backported LAVA patchs in lava-master/lava-patch
Doing the same for lava-slave will be done later.

## How to add/modify custom devices type
There are two way to add custom devices types.
* Copy a device type file directly in lava-master/device-types/
	If you have a brand new device-type, it is the simpliest way.
* Copy a patch addding/modifying a device-type in lava-master/device-types-patch/
	If you are modifying an already present (upstream) device-type, it is the best way.

## How to made LAVA slave use a proxy ?
Add env to a slave like:
slave:
  env:
  - "http_proxy: http://dns:port"

## How to use a board which uses PXE ?
All boards which uses PXE, could be used with LAVA via grub.
But you need to add a configuration in your DHCP server for that board.
This configuration need tell to the PXE to get GRUB for the dispatcher TFTP.
EXample for an upsquare and a dispatcher availlable at 192.168.66.1:
```
  	host upsquare {
		hardware ethernet 00:07:32:54:41:bb;
		filename "/boot/grub/x86_64-efi/core.efi";
		next-server 192.168.66.1;
	}
```

## How to host healthchecks
Healthchecks jobs needs externals ressources (rootfs, images, etc...).
By default, lava-docker healthchecks uses ones hosted on our github, but this imply usage of external networks and some bandwith.
For hosting locally healthchecks files, you can set healthcheck_host on a slave for hosting them.
Note that doing that bring some constraints:
- Since healthchecks jobs are hosted by the master, The healthcheck hostname must be the same accross all slaves.
- You need to set the base URL on the master via healthcheck_url
- If you have qemu devices, Since they are inside the docker which provides an internal DNS , you probably must use the container("healthcheck") name as hostname.
- In case of a simple setup, you can use the slave IP as healthcheck_url
- In more complex setup (slave sprayed on different site with different network subnets) you need to set a DNS server for having the same DNS availlable on all sites.

For setting a DNS server, the easiest way is to use dnsmasq and add in /etc/hosts "healtcheck ipaddressoftheslave"

Example:
One master and slave on DC A, and one slave on DC B.
Both slave need to have healthcheck_host to true and master will have healthcheck_url set to healthcheck:8080
You have to add a DNS server on both slave with an healthcheck entry.

## Bugs, Contact
The prefered way to submit bugs are via the github issue tracker
You can also contact us on #lava-docker on the freenode IRC network
