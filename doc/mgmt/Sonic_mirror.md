# Feature Name
Layer 3 Mirror Session
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
| Rev |     Date    |       Author       | Change Description                |
|:---:|:-----------:|:------------------:|-----------------------------------|
| 0.1 | 10/28/2019  |   Sucheta Mahara      | Initial version                   |

# About this Manual
This document provides information about configuring Mirror Sessions and ACL rules for mirrorring on an interface  using the management framework based on current capabilities of SONiC.

# Scope

This document cover mirror session configuration and show commands. It also covers ip mirror Access list configuration and show commands
.

# Definition/Abbreviation

### Table 1: Abbreviations


# 1 Feature Overview

This feature will allow the user to configure mirror session and ip mirror access list on an interface using SONiC Management Framework via CLI, REST or  gNMI. The underlying mirror session DB schema is already provided by mirror session support from SONiC.
The mirror sessions created can then be associated to ip mirror access list and ACL rules can be configured to mirror selected packet only.
The mirror access list can then be associated to interfaces for mirroring packets on that interface.

Note: Currently the backend doesn't support changing mirror session parameters once created.
Also certain parameters are required to create a mirror session as defined in the sonic yang for mirror session.


## 1.1 Requirements

### 1.1.1 Functional Requirements
Provide management framework support to existing SONiC capabilities with respect to mirror session and ip mirror access list.

### 1.1.2 Configuration and Management Requirements

1. CLI configuration/show support
2. REST API support
3. gNMI support

### 1.1.3 Scalability Requirements
N/A

### 1.1.4 Warm Boot Requirements
N/A

## 1.2 Design Overview
### 1.2.1 Basic Approach
Implement mirror session, ip mirror access list, ip mirror access group support using transformer in sonic-mgmt-framework.

### 1.2.2 Container
There will be changes in the sonic-mgmt-framework container only.
Following new additions will be done:--
1. XML file for the new CLI support
2. Python script to handle CLI request (actioner)
3. Jinja template to render CLI output (renderer)
No changes expected in sonic yang model for MIRROR_SESSION_TABLE or ACL_TABLE or ACL_RULE_TABLE.
Open-config acl yang will have a new ACL type added to create ACL mirror table type in config DB.
There will also be new extensions added to support ACL mirror as show below

```
identity ACL_MIRROR {
      base ACL_TYPE;
      description
          "Mirror-ACL with mirroring action.";
}


module: openconfig-acl
  +--rw acl
     +--rw config
     +--ro state
     |  +--ro counter-capability?   identityref
     +--rw acl-sets
     |  +--rw acl-set* [name type]
     |    +--rw acl-entries
            +--rw acl-entry* [sequence-id]
    NOTE: -     /*Below are the new extensions added to support ACL mirror */
                +--rw oc-acl-ext:config
     |           |  +--rw oc-acl-ext:session_id?   uint32
     |           +--ro oc-acl-ext:state
     |           |  +--ro oc-acl-ext:session_id?   uint32
     |           +--rw oc-acl-ext:ipv4
     |           |  +--rw oc-acl-ext:config
     |           |  |  +--rw oc-acl-ext:source-address?        oc-inet:ipv4-prefix
     |           |  |  +--rw oc-acl-ext:destination-address?   oc-inet:ipv4-prefix
     |           |  |  +--rw oc-acl-ext:dscp?                  oc-inet:dscp
     |           |  |  +--rw oc-acl-ext:protocol?              oc-pkt-match-types:ip-protocol-type
     |           |  |  +--rw oc-acl-ext:hop-limit?             uint8
     |           |  +--ro oc-acl-ext:state
     |           |     +--ro oc-acl-ext:source-address?        oc-inet:ipv4-prefix
     |           |     +--ro oc-acl-ext:destination-address?   oc-inet:ipv4-prefix
     |           |     +--ro oc-acl-ext:dscp?                  oc-inet:dscp
     |           |     +--ro oc-acl-ext:protocol?              oc-pkt-match-types:ip-protocol-type
     |           |     +--ro oc-acl-ext:hop-limit?             uint8
     |           +--rw oc-acl-ext:transport
     |              +--rw oc-acl-ext:config
     |              |  +--rw oc-acl-ext:source-port?        oc-pkt-match-types:port-num-range
     |              |  +--rw oc-acl-ext:destination-port?   oc-pkt-match-types:port-num-range
     |              |  +--rw oc-acl-ext:tcp-flags*          identityref
     |              +--ro oc-acl-ext:state
     |                 +--ro oc-acl-ext:source-port?        oc-pkt-match-types:port-num-range
     |                 +--ro oc-acl-ext:destination-port?   oc-pkt-match-types:port-num-range
     |                 +--ro oc-acl-ext:tcp-flags*          identityref

```



### 1.2.3 SAI Overview
N/A
# 2 Functionality
## 2.1 Target Deployment Use Cases
N/A
## 2.2 Functional Description
Provide CLI, gNMI and REST support for configuring  mirror session , ip mirror access list and ip mirror access group  in sonic management framework using transformer.
# 3 Design
## 3.1 Overview
Enhancing the SONiC management framework backend code to add support for mirror session ,ip mirror access list and ip mirror access group.
## 3.2 DB Changes
### 3.2.1 CONFIG DB
This feature will allow the user to make/show mirror session configuration changes to CONFIG DB.
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
N/A
### 3.3.2 Other Process
N/A
## 3.4 SyncD
N/A

## 3.5 SAI
N/A

## 3.6 User Interface
### 3.6.1 Data Models
https://github.com/project-arlo/sonic-mgmt-framework/blob/master/models/yang/sonic/sonic-mirror-session.yang
### 3.6.2 CLI

There will be three new CLIs added to support the full functionality in addition to show commands.

#### 3.6.2.1 Configuration Commands
All commands are executed in configuration-view:
```
sonic# configure terminal

sonic(config)#
```
#####  Add mirror session with the required parameters
`
monitor session <name> erpm source-ip <src-ip> destination-ip <dsp-ip>
`

```
sonic(config)# monitor session mirror1 erpm source-ip 1.1.1.1 destination-ip 2.2.2.2
```

#####  Add mirror session with required and optional  parameters
`
monitor session <session_name> erpm source-ip <src-ip> destination-ip <dsp-ip> ip-ttl <ttl> ip-dscp <dscp> queue < que> gre-protocol <id>
`

```
sonic(config)# mirror session add mir1 source-ip 1.1.1.1 destination-ip 2.2.2.2 ip-ttl 10
ip-dscp 10 queue 0 gre-protocol 0x6558
```
#####  Delete mirror session

`
no monitor session <name>
`

```
sonic(config)# no monitor session mirror1
```

##### Update mirror session : there is no backend support to modify a mirror session in orch-agent. This is an asked feature. The CLI to change mirror session will be provided if backend support is added.
The mirror orch-agent listens to change in neighbor information, next hop information in relation to reachability of  destination IP to update destination mac and destination port.
It also listens to the FDB entry change (one entry update) for destination MAC getting added on a member port (for vlan with ip) and LAG member addition ( for a lag with ip) to set correct destination MAC and
destination port in SAI if the VLAN and LAG have destination IP as a neighbor. Once a mirror session is created it's parameters cannot be altered.

```
mirror session mirror1

   list of configurable parameters based on backend Support
```

#### 3.6.2.2 Show Commands

#### show mirror sessions
`
show monitor session
`

```
sonic(config)# show monitor session

mirror session mir1
source-ip 1.1.1.1
destination-ip 2.2.2.2
ip-dscp 20
ip-ttl 10
gre-protocol 0x6558
queue 0
status  active
```
#### show mirror-access-group  sessions

`
 show ip mirror-access-group
`

```
sonic(config)# show ip mirror-access-group
Ingress IP mirror-access-list acl_mirror on Ethernet116
Ingress IP mirror-access-list acl_mirror on Ethernet112
```
#### show mirror-access-list

`
  show ip  mirror-access-list
`

```
sonic(config)# do show ip mirror-access-lists
ip mirror-access-list acl_mirror
     10 permit udp 0.0.0.0/0 0.0.0.0/0 dscp 10 session mir1
```

#### Create a access control list for mirror session to filter packets based on acl rules before mirroring.

`
ip mirror-access-list <name>  -------this creates entry of type mirror in ACL table
`

```
sonic(config)# ip mirror-access-list al-mirror
```

#### Create ACL rules for the mirror access list and associate a mirror session name to it.
This create ACL rules in ACL rule table and mirrors only the packet that are not filtered by the ACL rules.
The session id is required for mirror ip access-list.

`
seq <number> permit <other allowed ipv4 filters> capture <mirror session name>
`



```
sonic(config)# ip mirror-access-list al-mirror
sonic(config-ipv4-mirror-acl)# seq 2 permit tcp any src-eq 4096 any syn capture mir1
```


#### Delete ACL rules for mirror access list
`
no <sequence no>
`
```
sonic(config)# ip mirror-access-list al-mirror
sonic(config-ipv4-mirror-acl)# no seq 2
```

#### Delete a mirror ACL table and its rules

`
no ip mirror-access-list <mirror-access-list-name>
`

```
sonic(config)# no ip mirror-access-list al-mirror
```

#### Associate the mirror-access list with an interface

`
ip mirror-access-group <name>
`

```
sonic(config)# interface Ethernet 116
sonic(conf-if-Ethernet116)# ip mirror-access-group al-mirror
```

#### Delete a mirror-access list from an interface

`
no ip mirror-access-group <mirror-access-list-name>
`

```
sonic(config)# interface Ethernet 116
sonic(conf-if-Ethernet116)# no ip mirror-access-group al-mirror
```


#### 3.6.2.3 Debug Commands
#### 3.6.2.4 IS-CLI Compliance
This CLI is not a ISL command and is based on what sonic supports.

### 3.6.3 REST API Support

# 4 Flow Diagrams
N/A

# 5 Error Handling
N/A

# 6 Serviceability and Debug
/var/log/syslog can be checked for any failures.

# 7 Warm Boot Support
N/A

# 8 Scalability
N/A

# 9 Unit Test

#### Show mirror session configuration via CLI

#### add, delete a mirror session via CLI

#### Configuration via gNMI

#### Get configuration via gNMI

#### Configuration via REST (POST/PUT/PATCH)

#### Get configuration via REST (GET)

# 10 Internal Design Information
N/A
