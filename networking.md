# Container Networking demo

## Isolating containers with network namespaces

### Demo for connectivity between host and container using veth

- Create network namespace 
```bash
sudo ip netns add netns0
ip netns
``` 

- Enter into the newly created namespace
```bash
sudo nsenter --net=/var/run/netns/netns0 bash
```

- Create vEth pair and attach to the container (on host)
```bash
sudo ip link add veth0 type veth peer name ceth0
sudo ip link set ceth0 netns netns0
sudo ip link set veth0 up
sudo ip addr add 172.18.0.11/16 dev veth0
```

- on netns0
```bash
ip link set lo up
ip link set ceth0 upip addr add 172.18.0.10/16 dev ceth0
```

### Demo for interconnecting containers with bridge

- Create a second container
```bash
# From root namespace
$ sudo ip netns add netns1
$ sudo ip link add veth1 type veth peer name ceth1
$ sudo ip link set ceth1 netns netns1
$ sudo ip link set veth1 up
$ sudo ip addr add 172.18.0.21/16 dev veth1

$ sudo nsenter --net=/var/run/netns/netns1
$ ip link set lo up
$ ip link set ceth1 up
$ ip addr add 172.18.0.20/16 dev ceth1
```
Now in this case from netns1 we cannot reach the host and we cannot reach netns1 from host. However from netns0 we can reach veth1 but still cant reach netns1

- Recreate the 2 containers
```bash
sudo ip netns delete netns0
sudo ip netns delete netns1

# But if you still have some leftovers...
sudo ip link delete veth0
sudo ip link delete ceth0
sudo ip link delete veth1
sudo ip link delete ceth1
```

```bash
# On host vm
sudo ip netns add netns0
sudo ip link add veth0 type veth peer name ceth0
sudo ip link set veth0 up
sudo ip link set ceth0 netns netns0

sudo ip netns add netns1
sudo ip link add veth1 type veth peer name ceth1
sudo ip link set veth1 up
sudo ip link set ceth1 netns netns1

# On netns0
ip link set lo up
ip link set ceth0 up
ip addr add 172.18.0.10/16 dev ceth0


# On netns1
ip link set lo up
ip link set ceth1 up
ip addr add 172.18.0.20/16 dev ceth1
```

Make sure there is no new routes on the host:
```bash
ip route
```

- Create a bridge and attach veth0 and veth1 ends to bridge
```bash
sudo ip link set veth0 master br0
sudo ip link set veth1 master br0
```

- Assign IP address to bridge to get a route on the host routing table
```bash
sudo ip addr add 172.18.0.1/16 dev br0

$ ip route
# ... omitted lines ...
172.18.0.0/16 dev br0 proto kernel scope link src 172.18.0.1
```
