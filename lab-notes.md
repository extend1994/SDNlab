# Lab 1 - Creating SDN network topologies in Mininet

## Mininet
> Mininet creates a realistic virtual network, running real kernel, switch and application code, on a single machine

### Operations
* simple test
  ```shell
  # sudo mn --test <mininet_command>
  sudo mn --test pingall
  ```
* clean and exit
  ```shell
  sudo mn --clean
  sudo mn -c
  ```
* start an xterm for every host and switch
  ```shell
  sudo mn -x
  ```
* Change topology
  ```shell
  sudo mn --topo single,3 # Three hosts connect to a switch
  sudo mn --topo linear,3 # Three hosts connect to respective switchs which connect to each other
  ```
* Link properties
  ```shell
  sudo mn --link=tc,bw=10,delay=10ms
  ```
  Use `--link=tc` to use ***Traffic Control Link***.

  [Available parameters](http://mininet.org/api/classmininet_1_1link_1_1TCIntf.html):
  * `bw` - expressed as a number in Mbit
  * `delay` - expressed as a string with units in place (e.g. '5ms', '100us', '1s')
  * `lost` - expressed as a percentage (between 0 and 100)
  * `max_queue_size` - expressed in packets
  * `jitter` - similar to `delay`

### Commands
```shell
mininet> help
mininet> nodes
mininet> net
mininet> dump

# Interact with hosts and switchs
mininet> <node> <command>
mininet> h1 ifconfig
mininet> h1 ping -c 2 h2
mininet> xterm h1
```

### Create Custom Topologies
> Mininet provides python API to setup topologies or create testing scripts
A simple one is in mininet VM: `mininet/custom/topo-2sw-2host.py`
* addHost( \<host name> ) - add host
* addSwitch( \<switch name> ) - add switch
* addLink( \<node1>, \<node2>[,\<link properties>] ) - add link
```shell
# --custom is to use custom topology
# --topo is to use specified topology from dictionary "topo" in the script
# --link is to enable link properties
sudo mn --custom mininet/custom/topo-2sw-2host.py --topo mytopo [--link=tc]
```

### Mininet Python testing
```python
from mininet.net import Mininet
from mininet.topo import Topo
from mininet.link import TCLink
from mininet.util import dumpNodeConnections

# Create Mininet object
net = Mininet(topo=topo, link=TCLink,,controller=None)
# Use remote controller
net.addController(<controller_name>, controller=RemoteController, ip=<controller_IP>)
# Start Mininet
net.start()
# Dump host connections
dumpNodeConnections(net.hosts)
# Stop Mininet
net.stop()
# Get net nodes by their names
net.get('h1','h2')
# pingall test
net.pingAll()
# iperf test
net.iperf((h1,h2))

# Run a command and display output on the specific code
# <host>.cmdPrint(<command>)
h1.cmdPrint('ping -c 1' + h2.IP())
# Run a command and store output in a file
# <host>.popen('<command> > <filename>', shell=True)
h2.popen('ping -c 3' + h1.IP() '> file', shell=True)

from mininet.log import setLoglevel
"""
Setup the log level
setLogLevel(<option>), options are
info, warning, critical, error, debug, output
"""
setLogLevel('info')
```

## iPerf (iPerf -> iPerf2 -> iPerf3)
> iPerf3 is a tool for active measurements of the maximum achievable bandwidth on IP networks.
> Support tuning of various parameters related to timing, buffers and protocols (TCP, UDP, SCTP with IPv4 and IPv6), TCP protocol is the default one.

Need server and client sides to send data to each other to do bandwidth testing.
**Defautly**, iPerf3 get users' result on **uploading**.
iPerf public servers: https://iperf.fr/iperf-servers.php

```shell
# Be the server in iperf env
iperf3 -s

# Be a client in iperf env, host can be ip or hostname
iperf3 -c <host>
```

### Functionalities:
* Both server/client side
  * `-p` - Specify server port to listen on/connect to
  * `-f` - Format (bits/bytes)
  * `-i` - Bandwidth report interval
  * `-t` - Total time in secs to transmit (default 10s)
  * `-J` - Output in JSON format
* Server specific
* Client specific
  * `-u` - Use UDP connection
  * `-P` - More than one connection to server (parallel)
  * `-R` - Reverse the default connection to let server send and client receive
  * `-4`,`-6` - Only user IPv4 or IPv6
  * `-F` - Send a file rather than regular TCP packets.

# Lab 2: Using Open vSwitch as an OpenFlow switch for SDN Research

## Open vSwitch set up for the lab
```shell
sudo ovs-vsctl add-br ovs-br0
sudo ovs-vsctl add-port ovs-br0 ens37
sudo ovs-vsctl add-port ovs-br0 ens38
sudo ovs-vsctl set-controller ovss-br0 tcp:192.168.230.129:6633
sudo ovs-vsctl show # see all settings
```

# Lab 3: Writing SDN Application Modules over Ryu
## Lab goal
<img src="https://bit.ly/2HNpVsA" width="80%">
在 fat tree topology 中，將所有送往非指定 host 的 UDP packets 全部導向指定 host，<br>
在本次 lab 中，指定 host 為 Pod 0 的第一個 host。<br>

## How to run the environment
```shell
# Start Ryu controller
ryu run path/to/ryu/ryu/app/simple_switch_stp_13.py
sudo python 0656517.py
```

## simple_switch.py analysis

* `eth = pkt.get_protocol(ethernet.ethernet)`<br>
  Get Ethernet information, e.g. MAC
* `dpid = datapath.id`<br>
  **dpid** 為 source message 的來源 switch ID
* `self.logger.info("packet in %s %s %s %s", dpid, src, dst, msg.in_port)` <br>
  **in_port** 是收 message 用的 port
* `out_port = self.mac_to_port[dpid][dst]`<br>
  告訴 switch 要從哪個 **out_port** 送出 packet
* `out_port = ofproto.OFPP_FLOOD`<br>
  **OFPP_FLOOD** 讓 switch 把 packet 送往每個 switch output port
* `actions = [datapath.ofproto_parser.OFPActionOutput(out_port)]`<br>
  **OFPActionOutput(out_port)** 是一個讓 switch 把 packet 從 **out_port** 送出的 action
* `self.add_flow(datapath, msg.in_port, dst, src, actions)`<br>
  Add flow
* `out = datapath.ofproto_parser.OFPPacketOut(...)`<br>
  Create "PacketOut" message

Also see https://gist.github.com/aweimeow/d3662485aa224d298e671853aadb2d0f

## Examine flow entries

1. Via wireshark
2. `ovs-ofctl dump-flows <bridge>` or related instructions

## Discovery

When one wants to match UDP packet, one must first specify its ethertype,<br>
see [OpenFlow spec v1.3](https://bit.ly/1eyrkxY) page 30 for more details

# References
1. [Mininet introduction]( https://github.com/mininet/mininet/wiki/Introduction-to-Mininet#wiki-working)
2. [Mininet topology](http://www.routereflector.com/2013/11/mininet-as-an-sdn-test-platform/)
3. [Mininet Python API](http://mininet.org/api/annotated.html)
4. [All Mininet test](http://mininet.org/api/classmininet_1_1net_1_1Mininet.html)
5. [Mininet source repository resolution](https://www.sdnlab.com/11495.html)
6. [Ryu Flow Match Structure/Fields](http://ryu.readthedocs.io/en/latest/ofproto_v1_3_ref.html#ryu.ofproto.ofproto_v1_3_parser.OFPMatch)
7. Ryu Action Structures: [ActionOutput](http://ryu.readthedocs.io/en/latest/ofproto_v1_3_ref.html#ryu.ofproto.ofproto_v1_3_parser.OFPActionOutput), [ActionSetField](http://ryu.readthedocs.io/en/latest/ofproto_v1_3_ref.html#ryu.ofproto.ofproto_v1_3_parser.OFPActionSetField)
8. [Ryu Packet library API Reference](http://ryu.readthedocs.io/en/latest/library_packet_ref.html)
