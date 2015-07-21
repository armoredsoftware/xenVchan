#VChan

VChan is a means of communication between two (and only two) virtual machines
that are on the **same** Xen instance.

In order to use VChan the Virtual Machines need to have the following setup:

* Yum repository needs to be setup
* Xen development and Xen runtime need to be installed
* The Xen gntalloc, evtchn, and gntdev kernel modules need to be running

## Setup

### Yum Repository Setup / Xen VM Installations 
In order to set up the yum repository you need to:
* Obtain the IP address for the VM:
  * > sudo xl console [domain id]
  * > ifconfig
* Call ./vmYumSetup.sh [ip address] from compute node (Dom-0). The script
will ask for the 'root' password on the virtual machines.
* Log into the VM using sudo xl console or ssh
* In the home directory:
  * > rpm -i epel-release-6.8.noarch.rpm
  * > yum -y install xen-devel-4.2.2
  * > yum -y install xen-runtime-4.2.2
  * > reboot
     * Note: this reboot will change the domID and network IP address of the VM

### Xen Kernel Modules 

Log into the VM using *sudo xl console* or using *ssh*. Execute the following:
* > modprobe xen_gntalloc
* > modprobe xen_evtchn
* > modprobe xen_gntdev


## General Information

A simplified way to look at VChan communication is to think of each of the
Virtual Machines running on a Xen Instance having a directory on Domain-0
(using XenStore). When two VMs want to communicate with each other, one VM
must act as the server calling the *server_init* function, while the other VM
will call the *client_init*. The *server_init* call must occur **before** the
*client_init* function call. Both init calls take the domain ID of the other
VM. 

The *server_init* function basically designates a file in its directory
on Dom-0 and says the VM at the given domain ID is allowed access to this
file for communication. If domain 2 does a *server_init* to communicate with
domain 3, but then does a *server_init* for domain 4 with the same file, then
domain 3 will no longer be able to communicate with domain 2 on that channel.
This enforces that a channel can only be used by two entities. 

The *client_init* function attempts to access a file in the directory of the 
other domain in order to communicate. If the *client_init* call happens before
the *server_init* then an error will occur since the permissions have not yet
been granted to the client.

The VChan channel acts as a ring so sends and receives do not get crossed.
In other words if you send out a message and immediately try to receive
you will not receive the message you sent out. This ring also handles the
queuing of messages. If a producer sends 5 separate messages before the
consumer attempts a send, the consumer will then be able to receive the
first message by calling the *receive* function, and each subsequent message
with an additional calling to the *receive* function.

The *vchan_[client/server]_init* functions found in common.c and the 
*[server/client]_init* functions in the VChanUtil Haskell library abstract
away the need to specify the path of the channel, and creates a unique path
for each domain-to-domain communication. The *send* and *receive* functions
do the chunking up of sends and receives since the underlying library only
allows a maximum of 1024 bytes of data to be sent at one time.

The speed of VChan communication is roughly the same speed as writing to a file. 

##Main Functions in Library
| Function | Description|
|----------|------------|
| getDomId | Function used for a VM to obtain their domain id|
| server_init | Function must be called first by one VM in order to start communication |
| client init | Function to be called after a server_init call to make connection |
| maybe_client_init | Attempts a client_init and returns Null on a fail |
| send | send message of a certain size on a given channel |
| receive | receive message on a channel, message and size is returned |
| createDebugLogger | necessary for c library to use for init functions, not used for Haskell library |
| libxenvchan_wait / ctrlWait | function to block until an event happens on the channel | 
| libxenvchan_data_ready / dataReady | returns the size of the data ready to be read from the VChan channel (best used after a wait or for checking a channel to see if it is empty or ready for a receive)|

