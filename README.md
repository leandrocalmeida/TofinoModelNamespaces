# Description

# Topology

	h1(veth0) <--> (veth1:1)TofinoSW(veth3:2) <--> (veth2)h2  

### Create Namespaces
	$ sudo ip netns add h1 \\Create a h1 namespace
	$ sudo ip netns add h2 \\Create a h2 namespace

	$ sudo ip add veth0 type veth peer name veth1 \\Create veth pair between h1-TofinoSW
	$ sudo ip add veth2 type veth peer name veth3 \\Create veth pair between TofinoSW-h2

	$ sudo ip link set veth0 netns h1 \\Attach veth0 to h1
	$ sudo ip link set veth2 netns h2 \\Attach veth2 to h2

	$ sudo ip link set veth1 up \\Bring interface veth1 up
	$ sudo ip link set veth3 up \\Bring interface veth3 up

### Host1 (run in other terminal - tab 1)

	$ sudo ip netns exec h1 bash \\Run bash in h1 namespace
	# ip link set veth0 up \\Bring interface veth0 up
	# ip address add 172.16.0.1/16 dev veth0 \\Assign IP address in h1
	# arp -a 172.16.0.2 YY:YY:YY:YY:YY:YY \\Create a new arp entry for h2. Change YY:YY:YY:YY:YY:YY for h2 Mac address

### Host2 (run in other terminal - tab 2)

	$ sudo ip netns exec h2 bash \\Run bash in h2 namespace
	# ip link set veth2 up \\Bring interface veth2 up
	# ip address add 172.16.0.2/16 dev veth0 \\Assign IP address in h2
	# arp -a 172.16.0.1 XX:XX:XX:XX:XX:XX \\Create a new arp entry for h1. Change XX:XX:XX:XX:XX:XX for h1 Mac address

### Tofino
#### Compile P4 code 
	$ cd $SDE/tools
	$./p4build -p /path-to-p4-code/code.p4 --with-tofino

	//The p4_build script will create two folders(build-dir and log-dir).

	$SDE/
	    |-build/p4-build/tofino/code/
	    |-logs/p4-build/tofino/code/
	    
#### Port map (create a file to mapping the veth in tofino ports)
	
	$ cd /path-to-p4-code/
	$ > ports.json \\Create a port map file. 

	Insert the content in the file

	{
	    "PortToIf" : [
		{ "device_port" :  1, "if" : "veth1"  },
		{ "device_port" :  2, "if" : "veth3"  }
	    ]
	}

#### Run Tofino (Model)
	$ mkdir /path-to-code/logs \\Create a log directory
	$ cd $SDE
	$ ./run_tofino_model.sh -p code -f /path-to-p4-code/portmapping.json --log-dir $SDE/path-to-p4-code/logs/
	
#### Run switch daemon (switchd)
	$ ./run_switchd.sh -p code
	bfshell> bfrt_python
	[1] bfrt.code.pipe.SwitchIngress.ipv4_lpm
	bfrt.code.pipe.SwitchIngress.ipv4_lpm> add_with_ipv4_forward(dst_addr='172.16.0.1', dst_addr_p_length=32, dst_mac_addr='XX:XX:XX:XX:XX:XX', dst_port=1)
	bfrt.code.pipe.SwitchIngress.ipv4_lpm> add_with_ipv4_forward(dst_addr='172.16.0.2', dst_addr_p_length=32, dst_mac_addr='YY:YY:YY:YY:YY:YY', dst_port=2)
	
#### Test connectivity

	h1
    ```
	# ping 172.16.0.2
	```
	
    h2
    ```
	# ping 172.16.0.1
	```

	Tofino
	\\See the log files created in $SDE/path-to-p4-code/logs/
	Eg.: 
    ```
    $ tail -f model_20221012_132707.log
    ```



