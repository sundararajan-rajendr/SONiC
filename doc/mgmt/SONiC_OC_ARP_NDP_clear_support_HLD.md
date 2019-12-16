# High Level Design Document
#### Rev 0.1

# Table of Contents
  * [List of Tables](#list-of-tables)
  * [Revision](#revision)
  * [About This Manual](#about-this-manual)
  * [Scope](#scope)
  * [Definition/Abbreviation](#definitionabbreviation)

# List of Tables

# Revision
| Rev |     Date    |       Author       | Change Description                |
|:---:|:-----------:|:------------------:|-----------------------------------|
| 0.1 | 11/21/2019  |   Syed Obaid Amin  | Initial version                   |

# About this Manual
This document provides general information about 'clear ip arp' and 'clear ipv6 neighbors' commands for the SONiC subsystem via the Management Framework infrastructure.
# Scope
Covers CLI commands and northbound interface for clearing ARP and neighbors table.

# Definition/Abbreviation
### Table 1: Abbreviations
| **Term**                 | **Meaning**                         |
|--------------------------|-------------------------------------|
| ARP                      | Address Resolution Protocol         |
| NDP                      | Neighbor Discovery Protocol         |


# 1 Feature Overview
Provide Management Framework functionality to clear ARP and neighbors table.

## 1.1 Requirements

### 1.1.1 Functional Requirements

Provide a Management Framework based implementation of the "clear ip arp" and "clear ipv6 neighbors" commands. Match the functionality currently provided for this command via a Click-based host interface.

### 1.1.2 Configuration and Management Requirements
- IS-CLI style implementation of  the "clear ip arp" and "clear ipv6 neighbors" command
- REST API support
- gNMI Support

### 1.1.3 Scalability Requirements
N/A
### 1.1.4 Warm Boot Requirements
N/A
## 1.2 Design Overview
### 1.2.1 Basic Approach
In SONiC the neighbors' information is stored in NEIGH_TABLE of APPL_DB. The neighbor_sync process resolves mac address for neighbors using libnl and stores that information in NEIGH_TABLE. Also any static neighbor entry, made using Linux tools like "ip or arp", is also reflected in NEIGH_TABLE.
The 'clear ip arp' or 'clear ipv6 neighbors' command will therefore clear the neighbors cache using Linux tools, which eventually will clear the NEIGH_TABLE as well.

### 1.2.2 Container
The user interface (front end) portion of this feature is implemented within the Management Framework container.

### 1.2.3 SAI Overview
N/A (non-hardware feature)

# 2 Functionality
## 2.1 Target Deployment Use Cases
# 3 Design
## 3.1 Overview
The command causes invocation of an RPC sent from the management framework to a process in the host that clears the Linux neighbors cache using following commands:

For ipv4:
```
All entries:
 sudo ip -4 -s -s neigh flush all

Specific entry:
 sudo ip -4 neigh del <ip> dev <interface name>
```

For ipv6:
```
All entries:
 sudo ip -6 -s -s neigh flush all

Specific entry:
 sudo ip -6 neigh del <ip> dev <interface name>
```
NOTE: To execute these commands, the docker image should be running in the *privileged* mode.

This triggers neighbor sync process of SONiC and eventually the corresponding entries in NEIGH_TABLE get deleted.

Without any argument the command will flush the complete neighbor table. However, a user can specify interface or IP to clear corresponding neighbor entries only.
Help information and syntax details are provided if the command is preceded with '?'.

## 3.2 DB Changes
N/A
### 3.2.1 CONFIG DB
### 3.2.2 APP DB
### 3.2.3 STATE DB
### 3.2.4 ASIC DB
### 3.2.5 COUNTER DB

## 3.3 Switch State Service Design
N/A
### 3.3.1 Orchestration Agent
### 3.3.2 Other Process

## 3.4 SyncD
N/A

## 3.5 SAI
N/A

## 3.6 User Interface
### 3.6.1 Data Models
The following Sonic Yang model is used for implementation of this feature:

```
module: sonic-arp-ndp
    +--rw sonic-arp-ndp
       +--ro NEIGH_TABLE
          +--ro NEIGH_TABLE_LIST* [ifname ip]
             +--ro ifname    union
             +--ro ip        inet:ip-prefix
             +--ro neigh?    yang:mac-address
             +--ro family?   enumeration

  rpcs:
    +---x clear_arp_ndp
       +---w input
       |  +---w family?   enumeration
       |  +---w (option)?
       |     +--:(all)
       |     |  +---w all?      boolean
       |     +--:(ip)
       |     |  +---w ip?       inet:ip-prefix
       |     +--:(ifname)
       |        +---w ifname?   union
       +--ro output
          +--ro response?   string
```

### 3.6.2 CLI
#### 3.6.2.1 Configuration Commands
##### 3.6.2.1.1 `clear ip arp`
Syntax:

`
clear ip arp [interface { Ethernet <port> | PortChannel <id> | Vlan <id> | Management <id> }] [<A.B.C.D>]
`

The command returns a non-empty string in case of any error; for e.g. if the interface is not found or the IP address is not available in ARP or NDP table.

Syntax Description:

|    Keyword    | Description |
|:-----------------:|:-----------:|
| interface Ethernet/PortChannel/VLAN/Management*| This option clears the ARP entries matching the interface.
| A.B.C.D | This options clears the ARP entries matching the particular IP

*The Management interface translates to "eth" internally. For e.g. "clear ip arp inerface Management 0" will flush the entries learnt on interface "eth0".

Command Mode: User EXEC
Example:
```
sonic# clear ip arp
sonic#

sonic# clear ip arp 192.168.1.1
sonic#

sonic# clear ipv6 neighbors Ethernet 0
sonic#
```

##### 3.6.2.1.2 `clear ipv6 neighbors`
Syntax:

`
clear ipv6 neighbors [interface { Ethernet <port> | PortChannel <id> | Vlan <id> | Management <id> }] [<A::B>]
`

Syntax Description:

|    Keyword    | Description |
|:-----------------:|:-----------:|
| interface Ethernet/PortChannel/VLAN/Management*| This option clears the neighbors' entries matching the interface.
| A::B | This options clears the neighbors' entries matching the particular IPv6 address.

Command Mode: User EXEC

Example:
```
sonic# clear ipv6 neighbors
sonic#

sonic# clear ipv6 neighbors 20::2
sonic#

sonic# clear ipv6 neighbors Ethernet 0
sonic#
```

#### 3.6.2.2 Show Commands
N/A
#### 3.6.2.3 Debug Commands
N/A

#### 3.6.2.4 IS-CLI Compliance
|CLI Command|Compliance|IS-CLI Command (if applicable)| Link to the web site identifying the IS-CLI command (if applicable)|
|:---:|:-----------:|:------------------:|-----------------------------------|
|clear ip arp |IS-CLI drop-in replace | | |
|clear ip arp interface { Ehternet/PortChannel/Vlan/Management} |IS-CLI drop-in replace | | |
|clear ip arp ``<A.B.C.D>``  | IS-CLI drop-in replace | | |
| | | | |
|clear ipv6 neighbors |IS-CLI drop-in replace | | |
|clear ipv6 neighbors interface { Ethernet/PortChannel/Vlan/Management} |IS-CLI drop-in replace | | |
|clear ipv6 neighbors ``<A::B>``  | IS-CLI drop-in replace | | |

### 3.6.3 REST API Support
REST API support is provided. The REST API corresponds to the SONiC Yang model described in section 3.6.1.

# 4 Flow Diagrams

# 5 Error Handling
N/A

# 6 Serviceability and Debug
N/A

# 7 Warm Boot Support
N/A

# 8 Scalability
Refer to section 1.1.3

# 9 Unit Test
The following test cases will be tested using CLI/REST/gNMI management interfaces.
#### ARP test cases:
1) Verify whether "clear ip arp" command clears all the ARP entries

2) Verify whether "clear ip arp interface { ethernet/port-channel/vlan }" clears the ARPs learnt on the particular interface

3) Verify whether "clear ip arp <A.B.C.D> " clears the ARP entry matching the particular IP.

#### NDP test cases:
1) Verify whether "clear ipv6 neighbors" command clears all the neighbors entries

2) Verify whether "clear ipv6 neighbors interface {Ethernet/PortChannel/Vlan/Management}" clears the neighbor's learnt on the particular interface

3) Verify whether "clear ipv6 neighbors <A::B>" clears the neighbor entry matching the particular IP.

# 10 Internal Design Information
N/A
