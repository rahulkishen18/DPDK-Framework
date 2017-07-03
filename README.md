# DPDK-Framework
Brief overview of OVS-DPDK installation and packet transfer between a virtual and real host.

Specifications of the Equipment used:

1)Processor - Intel(R) Core(TM) i5-6200 CPU @ 2.30 GHz 4GB RAM
2)4 GB RAM
3)OS=Ubuntu 14.04
4)Kernel Version=3.19.0-25-generic(can be checked with the command uname -r)
5)eglibc 2.19(can be checked with ldd --version)
6)The NIC used for this purpose is I219 Intel e1000e.

OVS with DPDK and creation of an OVS bridge for communication between virtual and real host: 

export DPDK_DIR=/home/test/Downloads/dpdk-17.05
export OVS_DIR=/home/test/Downloads/ovs

pkill ovs
pkill ovsdb-server
rm -rf /usr/local/var/run/openvswitch
rm -rf /usr/local/etc/openvswitch/
rm -f /usr/local/etc/openvswitch/conf.db
mkdir -p /usr/local/etc/openvswitch
mkdir -p /usr/local/var/run/openvswitch
$OVS_DIR/ovsdb/ovsdb-tool create /usr/local/etc/openvswitch/conf.db ./vswitchd/vswitch.ovsschema
$OVS_DIR/ovsdb/ovsdb-server --remote=punix:/usr/local/var/run/openvswitch/db.sock --remote=db:Open_vSwitch,Open_vSwitch,manager_options --pidfile --detach

modprobe kvm-intel

sleep 1

echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
mount -t hugetlbfs hugetlbfs /mnt/huge
mount -t hugetlbfs none /mnt/huge_2mb -o pagesize=2MB
modprobe uio
insmod $DPDK_DIR/build/kmod/igb_uio.ko
dpdk-devbind --bind=igb_uio eth0

modprobe openvswitch

export OVS_DIR=/home/test/Downloads/ovs
cd $OVS_DIR

./vswitchd/ovs-vswitchd --detach
echo $OVS_DIR

$OVS_DIR/utilities/ovs-vsctl emer-reset

$OVS_DIR/utilities/ovs-vsctl --no-wait init
echo 'init done'
$OVS_DIR/utilities/ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-lcore-mask=0xf
echo 'Open_vSwitch done'
$OVS_DIR/utilities/ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-socket-mem=1024
./utilities/ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-socket-mem=1024
$OVS_DIR/utilities/ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-init=true
echo 'Open_vSwitch Config done'

ifconfig br0 down
sleep 1
$OVS_DIR/utilities/ovs-vsctl --no-wait del-br br0

$OVS_DIR/utilities/ovs-vsctl --no-wait add-br br0 -- set bridge br0 datapath_type=netdev
echo 'br0 Created'
$OVS_DIR/utilities/ovs-vsctl --no-wait add-port br0 vhost-user1 -- set Interface vhost-user1 type=dpdkvhostuser
$OVS_DIR/utilities/ovs-vsctl --no-wait add-port br0 vhost-user2 -- set Interface vhost-user2 type=dpdkvhostuser
$OVS_DIR/utilities/ovs-vsctl show

Creation of KVM:

ovs-vsctl add-port br0 wlan0
ifconfig wlan0 0
ifconfig br0 10.32.240.6 netmask 255.255.255.0
route add default gw 10.32.240.1 br0
ovs-vsctl del-port br0 tap0
kvm -m 512 -net nic,macaddr=00:00:00:00:cc:10 -net tap,script=/etc/ovs-ifup,downscript=/etc/ovs-ifdown cirros-0.3.5-x86_64-disk.img


Here we download the image of the kvm as cirros-0.3.5-x86_64-disk.img and allocate a RAM of 512 MB for the virtual host.

Pinging the real host for packets:

1)Log in to the virtual host
2)sudo ifconfig eth0 10.32.240.7/24
3)ping 10.32.240.6

Result:

We see a substantial increase in packet ping rate from about 8ms to 0.4ms.
This is done with OVS with DPDK enabled.

If the same thing is repeated with:
$OVS_DIR/utilities/ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-init=false

then,

then DPDK would be disabled and if we check we get the pinging rate as about 2-3ms.

Hence we conclude that OVS-DPDK together results in a substantial increase in packet ping rate than normal ping to a gateway or with just OVS with DPDK disabled.

