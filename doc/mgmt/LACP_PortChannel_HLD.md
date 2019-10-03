# LACP PortChannel
Openconfig support for LACP PortChannel interfaces

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
| 0.1 | 09/09/2019  |   Tejaswi Goel , Arthi Sivanantham     | Initial version                   |

# About this Manual
This document provides north bound interface details for LACP portchannels.

# Scope
This document covers the "configuration" and "show" commands supported for LACP PortChannel based on openconfig yang and Unit test cases. It does not include the protocol design or protocol implementation details.

# Definition/Abbreviation

### Table 1: Abbreviations
| **Term**                 | **Meaning**                         |
|--------------------------|-------------------------------------|
|          LAG             | Link aggregation                    |
|          LACP            | Link Aggregation Control Protocol   |

# 1 Feature Overview
Provide CLI, GNMI and REST management framework capabilities for CONFIG and GET support for the existing SONiC implemenatation of LACP feature.

## 1.1 Requirements
Provide management framework capabilities to handle:
- PortChannel creation and deletion
- Addition of ports to PortChannel
- Removal of ports from PortChannel
- Configure min-links, mtu, admin-status and ip address
- Show LACP PortChannel details

### 1.1.1 Functional Requirements
Provide management framework support to existing SONiC capabilities with respect to LACP PortChannels.

### 1.1.2 Configuration and Management Requirements
- IS-CLI style configuration and show commands
- REST API support
- gNMI Support

Details described in Section 3.

Configuration of LACP protocol parameters (LACP interval, mode, MAC, priority) using management framework is **not supported** due to the following limitations:

- No support for LACP protocol specific configurations in SONiC. For example, on creating port channels and adding members to it, “teammgrd” daemon (in SONIC code base):
1.	Reads user configuration from CONFIG DB
2.	Spawns an instance of libteamd (open source package for LACP protocol stack) with the given user configurations and a couple of default LACP options.

Now to support protocol specific configurations we would need to enhance:
1.	REDIS CONFIG DB PORTCHANNEL Table schema
2.	Code changes in “teammgrd” to read these protocol specific configs and to use “teamdctl” control utility to configure the above-mentioned configs to an already running "teamd" daemon.

- `teamd` supports only a few options (not LACP-specific) to be configured via `teamdctl` utility. This could be overcome by enhancing `teamd` to support configuration of LACP parameters.

Due to the above mentioned reasons, LACP **protocol specific configuration is not supported** in the initial release.

### 1.1.3 Scalability Requirements
key scaling factors -N/A
### 1.1.4 Warm Boot Requirements
N/A

## 1.2 Design Overview
### 1.2.1 Basic Approach
Provide transformer methods in sonic-mgmt-framework container for PortChannel configs and LACP show commands.

### 1.2.2 Container
All code changes will be done in management-framework container

### 1.2.3 SAI Overview
N/A

# 2 Functionality
## 2.1 Target Deployment Use Cases
Manage/configure/show LACP PortChannel interface via GNMI, REST and CLI interfaces
## 2.2 Functional Description
Provide CLI, GNMI and REST support LACP PortChannel related commands handling

# 3 Design
## 3.1 Overview
N/A

## 3.2 DB Changes
N/A
### 3.2.1 CONFIG DB
No changes to CONFIG DB. Will be populating PORTCHANNEL table and PORTCHANNEL_MEMBER table with user configuartions.

### 3.2.2 APP DB
Will be reading LAG_TABLE and LAG_MEMBER_TABLE for show commands.

### 3.2.3 STATE DB
### 3.2.4 ASIC DB
### 3.2.5 COUNTER DB

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
List of yang models required for PortChannel interface management.
1. **openconfig-if-aggregate.yang**
2. **openconfig-interfaces.yang**
3. **openconfig-lacp.yang**

**Note:** Currently LACP fallback not supported, so will be augmenting openconfig-if-aggregate.yang data model.

### 3.6.2 CLI
#### 3.6.2.1 Configuration Commands

**Create a PortChannel**

 `interface PortChannel \<channel-number>`

    SONiC(config)# interface PortChannel 1

  **Configure min-links**

  `minimum-links \<number>`

      SONiC(config)# interface PortChannel 1
      SONiC(conf-if-po1)# minimum-links 2

**Configure MTU**

`mtu \<number>`

    SONiC(config)# interface PortChannel 1
    SONiC(conf-if-po1)# mtu 1500

**Configure default MTU**

`no mtu`

    SONiC(config)# interface PortChannel 1
    SONiC(conf-if-po1)# no mtu

**Disable interface**

`shutdown`

    SONiC(config)# interface PortChannel 1
    SONiC(conf-if-po1)# shutdown

**Enable interface**

`no shutdown`

    SONiC(config)# interface PortChannel 1
    SONiC(conf-if-po1)# no shutdown

**Configure IP address**

`ip address \<ip-address with mask>`

    SONiC(config)# interface PortChannel 1
    SONiC(conf-if-po1)# ip address 1.2.3.4/16

**Remove IP address**

`no ip address \<ip-address>`

    SONiC(config)# interface PortChannel 1
    SONiC(conf-if-po1)# no ip address 1.2.3.4

**Add port member**

`channel-group \<channel-number>`

    SONiC(config)# interface Ethernet1
    SONiC(conf-if-Ethernet1)# channel-group 1

**Remove a port member**

`no channel-group`

    SONiC(config)# interface Ethernet1
    SONiC(conf-if-Ethernet1)# no channel-group

**Delete a PortChannel**

`no interface PortChannel \<channel-number>`

    SONiC(config)# no interface PortChannel 1

#### 3.6.2.2 Show Commands

**Show LACP information about all PortChannels**

`show lacp PortChannel`

    SONiC# show lacp PortChannel
    PortChannel1 admin up oper down mode active interval slow
    Actor System ID: Priority 65535 Address 90:b1:1c:f4:a8:7e
    Port Ethernet4
    Actor Port 5  Address 90:b1:1c:f4:a8:7e Key 0
    Partner Port 0  Address 00:00:00:00:00:00 Key 0
    Port Ethernet8
    Actor Port 5  Address 90:b1:1c:f4:a8:7e Key 0
    Partner Port 0  Address 00:00:00:00:00:00 Key 0
    PortChannel5 admin up oper down mode active interval slow
    Actor System ID: Priority 65535 Address 90:b1:1c:f4:a8:7e
    Port Ethernet12
    Actor Port 5  Address 90:b1:1c:f4:a8:7e Key 0
    Partner Port 0  Address 00:00:00:00:00:00 Key 0
    Port Ethernet16
    Actor Port 5  Address 90:b1:1c:f4:a8:7e Key 0
    Partner Port 0  Address 00:00:00:00:00:00 Key 0


**Show LACP information about a specific PortChannel**

`show lacp PortChannel interface PortChannel <port-channel id>`

    SONiC# show lacp PortChannel interface PortChannel 1
    PortChannel1 admin up oper down mode active interval slow
    Actor System ID: Priority 65535 Address 90:b1:1c:f4:a8:7e
    Port Ethernet4
    Actor Port 5  Address 90:b1:1c:f4:a8:7e Key 0
    Partner Port 0  Address 00:00:00:00:00:00 Key 0
    Port Ethernet8
    Actor Port 5  Address 90:b1:1c:f4:a8:7e Key 0
    Partner Port 0  Address 00:00:00:00:00:00 Key 0



#### 3.6.2.3 Debug Commands
N/A

#### 3.6.2.4 IS-CLI Compliance
N/A

### 3.6.3 REST API Support
| Command description | OpenConfig Command Path |
| :------ | :----- |
| Create/Delete a PortChannel | /openconfig-interfaces:interfaces/interface={name} |
| Set min-links  | /openconfig-interfaces:interfaces/interface={name}/openconfig-if-aggregate:aggregation/config/min-links |
| Set MTU/admin-status  | /openconfig-interfaces:interfaces/interface={name}/config/[admin-status|mtu] |
| Set IP | /openconfig-interfaces:interfaces/interface={name}/subinterfaces/subinterface[index=0]/openconfig-if-ip:ipv4/addresses/address={ip}/config |
| Add/Remove port member | /openconfig-interfaces:interfaces/interface={name}/openconfig-if-ethernet:ethernet/config/openconfig-if-aggregate:aggregate-id  |
| Get LACP PortChannel details | /openconfig-interfaces:interfaces/interface={name}/openconfig-if-aggregate:aggregation/state /openconfig-interfaces:interfaces/interface={name}/state/admin-status mtu]  /openconfig-lacp:lacp/interfaces/interface={name}/state /openconfig-lacp:lacp/interfaces/interface={name}/members/member={interface}/state|


# 4 Flow Diagrams
N/A

# 5 Error Handling
TBD

# 6 Serviceability and Debug
TBD

# 7 Warm Boot Support
N/A

# 8 Scalability
N/A

# 9 Unit Test
- Validate PortChannel creation via CLI, GNMI and REST
    - Verify error returned if PortChannel ID out of supported range
- Validate min links, mtu and admin-status config for PortChannel via CLI, GNMI and REST
    - Verify error returned if min-links value out of supported range
- Validate addition of ports to PortChannel via CLI, GNMI and REST
    - Verify error returned if port already part of other PortChannel
    - Validate MTU, speed and list of Vlans permitted on each member link are same as PortChannel
- Validate removal of ports from PortChannel via CLI, GNMI and REST
    - Verify error returned if PortChannel does not exist
    - Verify error returned if invalid interface given
- Validate PortChannel deletion via CLI, GNMI and REST
    - Verify error returned if PortChannel does not exist
- Validate show command's listed above using CLI, GNMI and REST

# 10 Internal Design Information
