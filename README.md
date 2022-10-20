# Description

In this repository you can find the initial steps to run Tofino Switch (in model type of execution), connecting two hosts (h1 and h2). We implement the IPv4 forwarding based on Longest Prefix Match. 

Our setup is based on linux namespaces, running entirely in a Ubuntu (22.04) VM. The version of SDE used is 9.10.0.

You can find *.p4 files in "tna_ipv4_lpm" directory.

## Topology

![Topology](https://github.com/leandrocalmeida/TofinoModelNamespaces/blob/main/img/topo.jpg)

### Create Namespaces
	$ sudo ip netns add h1 \\Create a h1 namespace
	$ sudo ip netns add h2 \\Create a h2 namespace

	$ sudo ip link add veth0 type veth peer name veth1 \\Create veth pair between h1-TofinoSW
	$ sudo ip link add veth2 type veth peer name veth3 \\Create veth pair between TofinoSW-h2

	$ sudo ip link set veth0 netns h1 \\Attach veth0 to h1
	$ sudo ip link set veth2 netns h2 \\Attach veth2 to h2

	$ sudo ip link set veth1 up \\Bring interface veth1 up
	$ sudo ip link set veth3 up \\Bring interface veth3 up

### Host1 (run in other terminal - tab 1)

	$ sudo ip netns exec h1 bash \\Run bash in h1 namespace
	# ip link set veth0 up \\Bring interface veth0 up
	# ip address add 172.16.0.1/16 dev veth0 \\Assign IP address in h1
	# arp -s 172.16.0.2 YY:YY:YY:YY:YY:YY \\Create a new arp entry for h2. Change YY:YY:YY:YY:YY:YY for h2 Mac address

### Host2 (run in other terminal - tab 2)

	$ sudo ip netns exec h2 bash \\Run bash in h2 namespace
	# ip link set veth2 up \\Bring interface veth2 up
	# ip address add 172.16.0.2/16 dev veth2 \\Assign IP address in h2
	# arp -s 172.16.0.1 XX:XX:XX:XX:XX:XX \\Create a new arp entry for h1. Change XX:XX:XX:XX:XX:XX for h1 Mac address

### Tofino
#### Compile P4 code 
	$ cd $SDE/tools
	$./p4build -p /path-to-p4-code/tna_ipv4_lpm/tna_ipv4_lpm.p4 --with-tofino

	//The p4_build script will create two folders(build-dir and log-dir).

	$SDE/
	    |-build/p4-build/tofino/tna_ipv4_lpm/
	    |-logs/p4-build/tofino/tna_ipv4_lpm/
	    
#### Port map (create a file to mapping the veth in tofino ports)
	
	$ cd /path-to-p4-code/tna_ipv4_lpm/
	$ > ports.json \\Create a port map file. 

	Insert the content in the file

	{
	    "PortToIf" : [
		{ "device_port" :  1, "if" : "veth1"  },
		{ "device_port" :  2, "if" : "veth3"  }
	    ]
	}

#### Run Tofino (Model)
	$ mkdir /path-to-code/tna_ipv4_lpm/logs \\Create a log directory
	$ cd $SDE
	$ ./run_tofino_model.sh -p tna_ipv4_lpm -f /path-to-p4-code/tna_ipv4_lpm/portmapping.json --log-dir $SDE/path-to-p4-code/tna_ipv4_lpm/logs/
	
#### Run switch daemon (switchd)
	$ ./run_switchd.sh -p tna_ipv4_lpm
	bfshell> bfrt_python
	[1] bfrt.code.pipe.SwitchIngress.ipv4_lpm
	bfrt.code.pipe.SwitchIngress.ipv4_lpm> add_with_ipv4_forward(dst_addr='172.16.0.1', dst_addr_p_length=32, dst_mac_addr='XX:XX:XX:XX:XX:XX', dst_port=1)
	bfrt.code.pipe.SwitchIngress.ipv4_lpm> add_with_ipv4_forward(dst_addr='172.16.0.2', dst_addr_p_length=32, dst_mac_addr='YY:YY:YY:YY:YY:YY', dst_port=2)
	
#### Test connectivity

	h1
	# ping 172.16.0.2
	
    h2
	# ping 172.16.0.1

	Tofino
	\\See the log files created in $SDE/path-to-p4-code/tna_ipv4_lpm/logs/
	Eg.: $ tail -f model_20221012_132707.log





[def]: /img/topo.jpg
