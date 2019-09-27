## cumulus-its-symmetric

### Summary:

  - Cumulus Linux 3.7.9
  - Underlying Topology Converter to 4.7.0
  - Tested against Vagrant 2.1.5 on Mac and Linux. Windows is not supported.
  - Tested against Virtualbox 5.2.32 on Mac 10.14
  - Tested against Libvirt 1.3.1 and Ubuntu 16.04 LTS

### Description:

This is an Ansible demo which configures client isolation using VRF and VXLAN to accomplish microsegmentation and multitenancy.

### Network Diagram:

![Network Diagram](https://github.com/chronot1995/cumulus-its-symmetric/blob/master/documentation/cumulus-its-symmetric.png)

### Install and Setup Virtualbox on Mac

Setup Vagrant for the first time on Mojave, MacOS 10.14.6

1. Install Homebrew 2.1.9 (This will also install Xcode Command Line Tools)

    https://brew.sh

2. Install Virtualbox (Tested with 5.2.32)

    https://www.virtualbox.org

I had to go through the install process twice to load the proper security extensions (System Preferences > Security & Privacy > General Tab > "Allow" on bottom)

3. Install Vagrant (Tested with 2.1.5)

    https://www.vagrantup.com

### Install and Setup Linux / libvirt demo environment:

First, make sure that the following is currently running on your machine:

1. This demo was tested on a Ubuntu 16.04 VM w/ 4 processors and 32Gb of Diagram

2. Following the instructions at the following link:

    https://docs.cumulusnetworks.com/cumulus-vx/Development-Environments/Vagrant-and-Libvirt-with-KVM-or-QEMU/

3. Download the latest Vagrant, 2.1.5, from the following location:

    https://www.vagrantup.com/

### Initializing the demo environment:

1. Copy the Git repo to your local machine:

    ```
    git clone https://github.com/chronot1995/cumulus-its-symmetric/
    ```

2. Change directories to the following

    ```
    cumulus-its-symmetric
    ```

3a. Run the following for Virtualbox:

    ```
    ./start-vagrant-vbox-poc.sh
    ```

3b. Run the following for Libvirt:
    ```
    ./start-vagrant-libvirt-poc.sh
    ```

### Running the Ansible Playbook

1a. SSH into the Virtualbox oob-mgmt-server:

    ```
    cd vx-vbox-simulation
    vagrant ssh oob-mgmt-server
    ```

1a. SSH into the Libvirt oob-mgmt-server:

    ```
    cd vx-libvirt-simulation  
    vagrant ssh oob-mgmt-server
    ```

2. Copy the Git repo unto the oob-mgmt-server:

    ```
    git clone https://github.com/chronot1995/cumulus-its-symmetric
    ```

3. Change directories to the following

    ```
    cumulus-its-symmetric/automation
    ```

4. Run the following:

    ```
    ./provision.sh
    ```

This will run the automation script and configure the environment.

5. Optional: Run the following playbook in the automation directory if you'd like to simulate a firewall on fw01:

    ```
    ansible-playbook break-server-ping.yml
    ```

This will put an ICMP block on the fw01 interfaces that connect to border01. You will be able to ping within the same VRF, as it doesn't go through the firewall, but you won't be able to ping between VRFs. You still will be able to SSH between VRFs, as a further test.

To disable the ICMP block, simply run the following:

    ```
    ansible-playbook fix-server-pings.yml
    ```

### Troubleshooting

Helpful VRF-enabled NCLU troubleshooting commands:

- net show route
- net show route vrf <name>
- net show bgp summary
- net show bgp vrf <name> summary
- net show interface | grep -i UP
- net show lldp

Helpful Linux troubleshooting commands:

- ip route
- ip link show
- ip address <interface>

The BGP Summary command for a VRF will show if each switch has formed an IPv4 neighbor relationship within that VRF:

```
ccumulus@border01:mgmt-vrf:~$ net show bgp vrf RED summary

show bgp vrf RED ipv4 unicast summary
=====================================
BGP router identifier 10.111.111.2, local AS number 65444 vrf-id 40
BGP table version 5
RIB entries 8, using 1216 bytes of memory
Peers 1, using 19 KiB of memory

Neighbor           V         AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd
fw01(10.111.111.1) 4      65555     756     749        0    0    0 00:37:12            3

Total number of neighbors 1
```

The more general command will show all of the peers across all the VRFs:

```
cumulus@border01:mgmt-vrf:~$ net show bgp vrf
Type  Id     routerId          #PeersVfg  #PeersEstb  Name                                  L3-VNI     Rmac
DFLT  0      10.4.4.4                  2           2  Default                               0          00:00:00:00:00:00
 VRF  40     10.111.111.2              1           1  RED                                   104001     44:39:39:ff:41:03
 VRF  43     10.222.222.2              1           1  BLUE                                  104002     44:39:39:ff:42:03
```

One should see all of the routes in the default VRF with the following:

```
cumulus@border01:mgmt-vrf:~$ net show route bgp
RIB entry for bgp
=================
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR,
       > - selected route, * - FIB route

B>* 10.1.1.1/32 [20/0] via fe80::4638:39ff:fe00:7, swp3, 00:41:06
B>* 10.2.2.2/32 [20/0] via fe80::4638:39ff:fe00:7, swp3, 00:41:07
B>* 10.3.3.3/32 [20/0] via fe80::4638:39ff:fe00:7, swp3, 00:41:07
B>* 10.5.5.5/32 [20/0] via 10.100.100.1, vlan100, 00:41:06
```

One can also view the VNIs that are established on all of the switches:

```
cumulus@leaf01:mgmt-vrf:~$ net show evpn vni
VNI        Type VxLAN IF              # MACs   # ARPs   # Remote VTEPs  Tenant VRF
22         L2   vni22                 2        7        1               BLUE
11         L2   vni11                 2        6        1               RED
104001     L3   L3VNI_RED             1        1        n/a             RED
104002     L3   L3VNI_BLUE            2        2        n/a             BLUE
```
```
cumulus@leaf02:mgmt-vrf:~$ net show evpn vni
VNI        Type VxLAN IF              # MACs   # ARPs   # Remote VTEPs  Tenant VRF
22         L2   vni22                 2        7        1               BLUE
11         L2   vni11                 2        7        1               RED
104001     L3   L3VNI_RED             1        1        n/a             RED
104002     L3   L3VNI_BLUE            1        1        n/a             BLUE
```

```
cumulus@border01:mgmt-vrf:~$ net show evpn vni
VNI        Type VxLAN IF              # MACs   # ARPs   # Remote VTEPs  Tenant VRF
104001     L3   L3VNI_RED             1        1        n/a             RED
104002     L3   L3VNI_BLUE            1        1        n/a             BLUE
```

Since this is a Symmetric design, the border will contain the RED and BLUE L3 VNIs.

The key to this design is the "Type 5" routes being propagated from the fw01 switch into the EVPN fabric as a Type 5 / default route:


```
cumulus@leaf01:mgmt-vrf:~$ net show bgp evpn route type prefix
BGP table version is 4, local router ID is 10.1.1.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
Origin codes: i - IGP, e - EGP, ? - incomplete
EVPN type-2 prefix: [2]:[ESI]:[EthTag]:[MAClen]:[MAC]:[IPlen]:[IP]
EVPN type-3 prefix: [3]:[EthTag]:[IPlen]:[OrigIP]
EVPN type-5 prefix: [5]:[ESI]:[EthTag]:[IPlen]:[IP]

   Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 10.111.111.2:2
*> [5]:[0]:[0]:[0]:[0.0.0.0]
                    10.4.4.4                               0 65333 65444 65555 i
*> [5]:[0]:[0]:[24]:[10.100.100.0]
                    10.4.4.4                               0 65333 65444 65555 i
*> [5]:[0]:[0]:[24]:[10.111.111.0]
                    10.4.4.4                               0 65333 65444 i
*> [5]:[0]:[0]:[32]:[10.5.5.5]
                    10.4.4.4                               0 65333 65444 65555 i
Route Distinguisher: 10.222.222.2:3
*> [5]:[0]:[0]:[0]:[0.0.0.0]
                    10.4.4.4                               0 65333 65444 65555 i
*> [5]:[0]:[0]:[24]:[10.100.100.0]
                    10.4.4.4                               0 65333 65444 65555 i
*> [5]:[0]:[0]:[24]:[10.222.222.0]
                    10.4.4.4                               0 65333 65444 i
*> [5]:[0]:[0]:[32]:[10.5.5.5]
                    10.4.4.4                               0 65333 65444 65555 i

Displayed 8 prefixes (8 paths) (of requested type)
```

In the below, server01 is tracing to server02, which is on the same leaf, but in a different VRF:

```
cumulus@server01:~$ traceroute 192.168.22.111
traceroute to 192.168.22.111 (192.168.22.111), 30 hops max, 60 byte packets
 1  192.168.11.11 (192.168.11.11)  0.436 ms  0.478 ms  0.661 ms
 2  10.111.111.2 (10.111.111.2)  2.816 ms  2.794 ms  2.765 ms
 3  10.111.111.1 (10.111.111.1)  3.436 ms  3.405 ms  3.376 ms
 4  10.222.222.2 (10.222.222.2)  6.018 ms  5.998 ms  5.971 ms
 5  192.168.22.22 (192.168.22.22)  7.258 ms  7.235 ms  7.168 ms
 6  192.168.22.111 (192.168.22.111)  7.080 ms  4.863 ms  4.836 ms
```
The above traffic goes through 10.111.111.1, which is fw01. This highlights VRF tenant separation.

The next output is an intra-VRF traceroute between server01 and server03. This traceroute takes the VXLAN tunnel directly from server01 to server03:

```
cumulus@server01:~$ traceroute 192.168.11.222
traceroute to 192.168.11.222 (192.168.11.222), 30 hops max, 60 byte packets
 1  192.168.11.222 (192.168.11.222)  3.357 ms  3.317 ms  3.274 ms
```

This is an inter-VRF communication from server01 in the RED VRF on leaf01 to server04 in the BLUE VRF  on leaf02:

```
cumulus@server01:~$ traceroute 192.168.22.222
traceroute to 192.168.22.222 (192.168.22.222), 30 hops max, 60 byte packets
 1  192.168.11.11 (192.168.11.11)  0.594 ms  0.547 ms  0.515 ms
 2  10.111.111.2 (10.111.111.2)  7.015 ms  6.994 ms  6.956 ms
 3  10.111.111.1 (10.111.111.1)  6.912 ms  6.868 ms  6.739 ms
 4  10.222.222.2 (10.222.222.2)  6.680 ms  6.735 ms  6.705 ms
 5  192.168.22.22 (192.168.22.22)  9.664 ms  9.635 ms  9.602 ms
 6  192.168.22.222 (192.168.22.222)  9.555 ms  4.656 ms  4.627 ms
```

### Errata

1. To shutdown the demo, run the following command from the vx-simulation directory:

    ```vagrant destroy -f```

2. This topology was configured using the Cumulus Topology Converter found at the following URL:

    https://github.com/CumulusNetworks/topology_converter

3. The following command was used to run the Topology Converter within the appropriate vx-sim directory:

```
     ./topology_converter.py cumulus-its-symmetric.dot -c --provider=virtualbox
     ./topology_converter.py cumulus-its-symmetric.dot -c --provider=libvirt
```

After the above command is executed, the following configuration changes are necessary:

4. Within "<vx-sim>/helper_scripts/auto_mgmt_network/OOB_Server_Config_auto_mgmt.sh"

The following stanza:

echo " ### Creating cumulus user ###"
useradd -m cumulus

Will be replaced with the following:

echo " ### Creating cumulus user ###"
useradd -m cumulus -m -s /bin/bash

The following stanza:

    #Install Automation Tools
    puppet=0
    ansible=1
    ansible_version=2.6.3

Will be replaced with the following:

    #Install Automation Tools
    puppet=0
    ansible=1
    ansible_version=2.8.5

Add the following ```echo``` right before the end of the file.

    echo " ### Adding .bash_profile to auto login as cumulus user"
    echo "sudo su - cumulus" >> /home/vagrant/.bash_profile
    echo "exit" >> /home/vagrant/.bash_profile
    echo "### Adding .ssh_config to avoid HostKeyChecking"
    printf "Host * \n\t StrictHostKeyChecking no\n" >> /home/cumulus/.ssh/config

    echo "############################################"
    echo "      DONE!"
    echo "############################################"
