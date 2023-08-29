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
ip link set ceth0 up
ip addr add 172.18.0.10/16 dev ceth0
```

### Demo for interconnecting containers with bridge

- Create a second container
```bash
# From root namespace
sudo ip netns add netns1
sudo ip link add veth1 type veth peer name ceth1
sudo ip link set ceth1 netns netns1
sudo ip link set veth1 up
sudo ip addr add 172.18.0.21/16 dev veth1

# From netns1 
ip link set lo up
ip link set ceth1 up
ip addr add 172.18.0.20/16 dev ceth1
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
sudo ip link add br0 type bridge
sudo ip link set br0 up

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

### Demo for reaching out to the outside world (IP routing and masquerading)

At present in netns0 we are unable to reach host vm's ip 10.200.100.20 as there is no routei for that defined. The host cannot reach the containers as well. To establish connection between host and container we need to assign ip to the bridge

```bash
sudo ip addr add 172.18.0.1/16 dev br0
```

Once we assigned the IP address to the bridge interface, we got a route on the host routing table.

The container probably also got an ability to ping the bridge interface, but they still cannot reach out to host's ens18. We need to add the default route for containers:

```bash
#Add the route in both containers

ip route add default via 172.18.0.1
```

- Now that we have connected containers with host vm, lets connect the containers to outside world. For that ensure packet forwarding is enabled

```bash
cat /proc/sys/net/ipv4/ip_forward
```

- To communicate with outside world we need to add a new rule to the nat table of the POSTROUTING chain asking to masquerade all the packets originated in 172.18.0.0/16 network, but not by the bridge interface.
```bash
sudo iptables -t nat -A POSTROUTING -s 172.18.0.0/16 ! -o br0 -j MASQUERADE
```

### Letting the outside world reach out to containers (port publishing)

- Lets start an http server inside one container
```bash
python3 -m http.server --bind 172.18.0.10 5000
```

Now we will be able to access 172.18.0.10:5000 from the host however to access it from outside world we need to publish the container's port 5000 on the host's ens18 interface
```bash
# External traffic
sudo iptables -t nat -A PREROUTING -d 10.0.2.15 -p tcp -m tcp --dport 5000 -j DNAT --to-destination 172.18.0.10:5000

# Local traffic (since it doesn't pass the PREROUTING chain)
sudo iptables -t nat -A OUTPUT -d 10.0.2.15 -p tcp -m tcp --dport 5000 -j DNAT --to-destination 172.18.0.10:5000
```

