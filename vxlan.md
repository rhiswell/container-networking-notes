# VXLAN

## Demo a VXLAN-based overlay network in Linux 

```bash
# First we create a vxlan device and then attach to a bridge
$ sudo ip link add vxlan10 type vxlan id 10 group 239.1.1.1 dev ib0
$ sudo ip link add br-vxlan10 type bridge
$ sudo ip link set vxlan10 master br-vxlan10
$ sudo ip link set vxlan10 up
$ sudo ip link set br-vxlan10 up
# Allow fowarding between vxlan device and bridge
$ sudo iptables -P FORWARD ACCEPT

# Create a pair of veths that are used to link container and host's network namespace
$ sudo ip link add vxlan10-veth1 type veth peer name vxlan10-veth0
$ sudo ip link set dev vxlan10-veth0 up
$ sudo ip link set vxlan10-veth0 master br-vxlan10

# Make docker run permenatly in background
$ sudo docker run -d --rm --privileged \
$	--network=none --name=guest0 ubuntu:14.04 tail -f /dev/null
# or
$ sudo docker run -d -t --rm --privileged --network=none --name=guest0 ubuntu:14.04
# alias docker-pid="sudo docker inspect --format '{{.State.Pid}}'"
$ sudo ip link set vxlan10-veth1 netns $(docker-pid guest0) 
$ sudo ip netns exec $(docker-pid guest0) ip addr add 192.168.233.140 dev vxlan10-veth1 
$ sudo ip netns exec $(docker-pid guest0) ip link set vxlan10-veth1 up

# Repeat above commands in another host then make a test
$ sudo docker exec guest0 ping -c3 192.168.233.141

# Quit and do post clearing
$ sudo docker stop guest0
$ sudo ip link del br-vxlan10
$ sudo ip link del vxlan10
```

## Refs

- How to create overlay networks using Linux Bridges and VXLANs. https://ilearnedhowto.wordpress.com/2017/02/16/how-to-create-overlay-networks-using-linux-bridges-and-vxlans/.
- VXLAN: Extending Networking to Fit the Cloud. https://www.usenix.org/publications/login/october-2012-volume-37-number-5/vxlan-extending-networking-fit-cloud.
