# Syslog messages forwarding

# High Level Design Document
#### Rev 0.1

# Table of Contents
  * [List of Tables](#list-of-tables)
  * [Revision](#revision)
  * [About This Manual](#about-this-manual)
  * [Scope](#scope)
  * [Definition/Abbreviation](#definitionabbreviation)

# List of Tables
[Table 1: Abbreviations](#table-1-abbreviations)

# Revision
| Rev  |    Date    |    Author     | Change Description       |
| :--: | :--------: | :-----------: | ------------------------ |
| 0.1  | 05/08/2019 | Suresh Babu R | Initial version          |
| 0.2  | 05/08/2019 | Suresh Babu R | Updated source-interface |

# About this Manual
This document provides general information about the configuration of remote syslog servers using management framework
# Scope
This document describes the REST-API, KLISH, VRF and source-interface support for remote syslog servers based on OpenConfig YANG model.

# Definition/Abbreviation

### Table 1: Abbreviations
| **Term** | **Meaning**               |
| -------- | ------------------------- |
| VRF      | Virtual routng and  forwarding |

# 1 Feature Overview
By default syslog messages are not forwarded to any server. User need to configure remote syslog server to forward messages.
Configure switch using REST and KLISH CLI to forward  messages to remote syslog servers. Also provide optional source-interface and VRF to be used while sending out syslog messages.
If source-interface is not configured then route interface ip is used as source ip of syslog packets. In case source-interface has more than one ip then primary ip is used as source-ip.



## 1.1 Requirements

1.1.1 Functional Requirements

### 1.1.1 Functional Requirements

Provide configuration options for  remote syslog server, port, source-interface and VRF.

### 1.1.2 Configuration and Management Requirements 
Provide REST-API and KLISH CLI commands support to configure remote-server, remote-port, source-interface and VRF using management framework.

### 1.1.3 Scalability Requirements
MAX 8 syslog servers will be supported
### 1.1.4 Warm Boot Requirements

N/A

## 1.2 Design Overview
### 1.2.1 Basic Approach
Implement configuration of remote syslog server with optional source-interface and VRF

### 1.2.2 Container
Changes are part of management framework container, redis-db and hostcfgd

### 1.2.3 SAI Overview
N/A

# 2 Functionality
## 2.1 Target Deployment Use Cases
User can use this feature to forward syslog messages to remote server

## 2.2 Functional Description
User can forward syslog messages generated on the switch to centralized remote syslog server for better log analysis by tools and to save space on the switch. 

# 3 Design
## 3.1 Overview
Whenever user changes syslog configuration, CONFIG_DB notifies hostcfgd to start syslog configuration script which in turn writes syslog server configuration to /etc/rsyslog.conf file using jinja2 template and restart the syslog service.
## 3.2 DB Changes

### 3.2.1 CONFIG DB

**3.2.1.1 SYSLOG_SERVER Table Schema**

```
;Syslog server configuration in the system
;Key
host          = inet:host; syslog server's Domain name or IP address (IPv4 or IPv6)
;Attributes
remote-port        = 1*5DIGIT
source-interface     = ifname; source inetrface for out going syslog packets
vrf-name           = vrfName
```



### 3.2.2 APP DB

N/A

### 3.2.3 STATE DB

N/A

### 3.2.4 ASIC DB

N/A

### 3.2.5 COUNTER DB

N/A

## 3.3 Switch State Service Design
### 3.3.1 Orchestration Agent
### 3.3.2 Other Process 
N/A

## 3.4 SyncD
N/A

## 3.5 SAI
N/A

## 3.6 User Interface
### 3.6.1 Data Models
Following YANG models are used for syslog server configuration.

openconfig-system-logging.yang

openconfig-system-ext.yang

sonic-system-logging.yang

```diff
+--rw remote-servers
     |     +--rw remote-server* [host]
     |        +--rw host         -> ../config/host
     |        +--rw config
     |        |  +--rw host?                          oc-inet:host
     |        |  +--rw remote-port?                   oc-inet:port-number
     |        |  +--rw oc-sys-ext:source-interface?   -> /oc-if:interfaces/interface/name
     |        |  +--rw oc-sys-ext:vrf-name?           -> /oc-ni:network-instances/network-instance/name
     |        +--ro state
     |        |  +--ro host?                          oc-inet:host
     |        |  +--ro remote-port?                   oc-inet:port-number
     |        |  +--ro oc-sys-ext:source-interface?   -> /oc-if:interfaces/interface/name
     |        |  +--ro oc-sys-ext:vrf-name?           -> /oc-ni:network-instances/network-instance/name

```



### 3.6.2 CLI
#### 3.6.2.1 Configuration Commands

```
logging server <host|ip| ipv6> [ source-interface { Ethernet | Loopback | Management | PortChannel | Vlan } ] [ remote-port <vrport> ] [ vrf <vvrf> ]
sonic(config)# logging server  10.59.142.126
sonic(config)# logging server  10.59.143.28 source-interface Loopback 1 vrf Vrf1
sonic(config)# logging server  10.59.136.33 source-interface Ethernet 4
sonic(config)#
```



```
no logging server <host> { [ remote-port ] | [ source-interface ] | [ vrf ] } ]
sonic(config)# no logging server  10.59.143.28
sonic(config)#no logging server  10.59.143.28 source-interface
sonic(config)# no logging server  10.59.143.28 vrf
```



#### 3.6.2.2 Show Commands

```
sonic# show logging servers
--------------------------------------------------------------------------------
HOST            PORT      SOURCE-INTERFACE       VRF
--------------------------------------------------------------------------------
10.59.136.33    514       Loopback1      -
10.59.142.126   514       -               -
10.59.143.28    514       Ethernet4      Vrf1
sonic#
```



#### 3.6.2.3 Debug Commands

**When remote syslog server is configured to use VRF and VRF is not created then following syslog message is generated.**

````
May 11 05:29:33.751764 sonic ERR rsyslogd: create UDP socket bound to device failed: No such device [v8.1901.0]

May 11 05:29:33.751806 sonic ERR rsyslogd: No UDP socket could successfully be initialized, some functionality may be disabled.  [v8.1901.0]

````

**If remote syslog server is not reachable via give VRF them following syslog messages are generated.**

```
May 11 05:32:24.793627 sonic ERR rsyslogd: omfwd/udp: socket 9: sendto() error: Network is unreachable [v8.1901.0 try https://www.rsyslog.com/e/2354 ]

May 11 05:32:24.793671 sonic ERR rsyslogd: omfwd: socket 9: error 101 sending via udp: Network is unreachable [v8.1901.0 try https://www.rsyslog.com/e/2354 ]

```



#### 3.6.2.4 IS-CLI Compliance
KLISH Based CLI



### 3.6.3 REST API Support

RESTCONF-APIs are supported for following YANG models 

openconfig-system-logging.yang

openconfig-system-ext.yang

sonic-system-logging.yang

Use URI from "swagger UI" or "sonic-buildimage/src/sonic-mgmt-framework/build/restconf_md/openconfig-system.md"

### 3.6.4 Service and Docker Management

N/A

# 4 Flow Diagrams
N/A

# 5 Error Handling
N/A

# 6 Serviceability and Debug
N/A

# 7 Warm Boot Support
Configuration should be persistent across reboot.

# 8 Scalability
8 servers

# 9 Unit Test
Topology: D1 with remote syslog server.
1)Verify add/delete syslog server configuration(hostname, ip, ipv6) using KLISH CLI and make sure that /etc/rsyslog.conf is updated accordingly

2)Verify show command works fine.

3)Verify REST APIs for configuration and operational data

4)Verify GET REST-API

5)Verify with config reload

6)Verify syslog server add/delete/modify with remote-port

7)Verify syslog server add/delete/modify with source-interface

8)Verify syslog server add/delete/modify with  vrf ( mgmt and user defined)

9)Verify syslog server add/delete/modify with source-interface and vrf


# 10 Internal Design Information
Internal BRCM information to be removed before sharing with the community.

