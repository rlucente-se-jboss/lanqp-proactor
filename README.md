THIS IS A WORK IN PROGRESS

# Overview
LanQP enables tunnelling of packets over an AMQP router network.
The lanqp-proactor executable relies on the following command line
options:

| Option | Description |
| ------ | ----------- |
| -a <address> | The address of the AMQ Interconnect network connection in the form `host:port` |
| -d | Daemonize the process |
| -P <file> | The PID file for the daemonized process |
| -U <user> | The user for the daemon to run as |

The lanqp-proactor executable also relies on the following environment variables:

| Env Var | Description |
| ------- | ----------- |
| LANQP_IF_COUNT | Number of tun interfaces |
| LANQP_IFn_NAME | The tun interface name where n >= 0 (default is `lanq0`) |

# Prerequisites
Install Fedora 27 Server on three virtual machines with the minimal
installation profile.  Designate the virtual machines as A, B, and
C.  On A and C, install packages needed to build and run lanqp-proactor.

    sudo dnf -y update
    sudo dnf -y group install 'C Development Tools and Libraries'
    sudo dnf -y install \
	cmake git net-tools qpid-proton-c-devel tunctl nmap-ncat
    sudo dnf -y clean all
    sudo systemctl reboot

On virtual machine B, install the dispatch router.

    sudo dnf -y update
    sudo dnf -y install \
        qpid-dispatch-router qpid-dispatch-tools
    sudo dnf -y clean all
    sudo systemctl enable qdrouterd
    sudo firewall-cmd --permanent --add-port=5672/tcp
    sudo systemctl reboot

# Build and Install
On virtual machines A and C, build the lanqp-proactor application
to tunnel packets over AMQP.

    git clone https://github.com/rlucente-se-jboss/lanqp-proactor.git
    cd lanqp-proactor
    mkdir build
    cd build
    cmake ..
    make

# Test
## Determine Router IP Address
On virtual machine B, type the following command to determine the IP address.

    sudo ip a s

192.168.2.103

Create a tunnel device on virtual machine A.  Feel free to use an
alternative unprivileged user instead of `rlucente`.

    sudo tunctl -t lanq0 -n -u rlucente
    sudo ifconfig lanq0 10.244.1.1 netmask 255.255.0.0 up

Create a tunnel device on virtual machine C.  Feel free to use an
alternative unprivileged user instead of 'rlucente'.

    sudo tunctl -t lanq0 -n -u rlucente
    sudo ifconfig lanq0 10.244.1.2 netmask 255.255.0.0 up

Configure and start lanqp-proactor on both A and C which will bring
up the tun interfaces.

    export LANQP_IF_COUNT=1
    export LANQP_IF0_NAME=lanq0
    ./lanqp-proactor -a localhost:amqp -d

# Cleanup
To shut it all down and remove the tunnel devices:

    pkill lanqp
    sudo systemctl stop qdrouterd
    sudo ip tuntap del mode tun name lanq0

