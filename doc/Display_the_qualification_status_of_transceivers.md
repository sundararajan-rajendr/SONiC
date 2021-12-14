# Feature Name
Display the qualification status of transceivers

# High Level Design Document
# Table of Contents

<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Feature Name](#feature-name)
- [High Level Design Document](#high-level-design-document)
- [Table of Contents](#table-of-contents)
- [List of Tables](#list-of-tables)
- [Revision](#revision)
- [About this Manual](#about-this-manual)
- [Definition/Abbreviation](#definitionabbreviation)
		- [Table 1: Abbreviations](#table-1-abbreviations)
- [1 Feature Overview](#1-feature-overview)
	- [1.1 Target Deployment Use Cases](#11-target-deployment-use-cases)
	- [1.2 Requirements](#12-requirements)
	- [1.3 Design Overview](#13-design-overview)
		- [1.3.1 Basic Approach](#131-basic-approach)
		- [1.3.2 Container](#132-container)
		- [1.3.3 SAI Overview](#133-sai-overview)
- [2 Functionality](#2-functionality)
- [3 Design](#3-design)
	- [3.1 Overview](#31-overview)
		- [3.1.1 Service and Docker Management](#311-service-and-docker-management)
		- [3.1.2 Packet Handling](#312-packet-handling)
	- [3.2 DB Changes](#32-db-changes)
		- [3.2.1 CONFIG DB](#321-config-db)
		- [3.2.2 APP DB](#322-app-db)
		- [3.2.3 STATE DB](#323-state-db)
		- [3.2.4 ASIC DB](#324-asic-db)
		- [3.2.5 COUNTER DB](#325-counter-db)
		- [3.2.6 ERROR DB](#326-error-db)
	- [3.3 Switch State Service Design](#33-switch-state-service-design)
		- [3.3.1 Orchestration Agent](#331-orchestration-agent)
		- [3.3.2 Other Processes](#332-other-processes)
	- [3.4 SyncD](#34-syncd)
	- [3.5 SAI](#35-sai)
	- [3.6 User Interface](#36-user-interface)
		- [3.6.1 Data Models](#361-data-models)
		- [3.6.2 CLI](#362-cli)
			- [3.6.2.1 Configuration Commands](#3621-configuration-commands)
			- [3.6.2.2 Show Commands](#3622-show-commands)
			- [3.6.2.3 Exec Commands](#3623-exec-commands)
		- [3.6.3 REST API Support](#363-rest-api-support)
		- [3.6.4 gNMI Support](#364-gnmi-support)
	- [3.7 Warm Boot Support](#37-warm-boot-support)
	- [3.8 Upgrade and Downgrade Considerations](#38-upgrade-and-downgrade-considerations)
	- [3.9 Resource Needs](#39-resource-needs)
- [4 Flow Diagrams](#4-flow-diagrams)
- [5 Error Handling](#5-error-handling)
- [6 Serviceability and Debug](#6-serviceability-and-debug)
- [7 Scalability](#7-scalability)
- [8 Platform](#8-platform)
- [9 Security and Threat Model](#9-security-and-threat-model)
- [10 Limitations](#10-limitations)
- [11 Unit Test](#11-unit-test)
- [12 Internal Design Information](#12-internal-design-information)
	- [12.1 IS-CLI Compliance](#121-is-cli-compliance)
	- [12.2 SONiC Packaging](#122-sonic-packaging)
	- [12.3 Broadcom Silicon Considerations](#123-broadcom-silicon-considerations)
	- [12.4 Design Alternatives](#124-design-alternatives)
	- [12.5 Release Matrix](#125-release-matrix)

<!-- /TOC -->

# List of Tables
[Table 1: Abbreviations](#table-1-Abbreviations)

# Revision
| Rev |     Date    |       Author           | Change Description                |
|:---:|:-----------:|:----------------------:|-----------------------------------|
| 0.1 | 11/17/2021  |Sundararajan Rajendran  | Initial version                   |
| 0.2 | 12/02/2021  |Sundararajan Rajendran  | Updating API details              |
| 0.3 | 12/09/2021  |Sundararajan Rajendran  | Data Model and CLI details        |

# About this Manual
This document provides comprehensive functional and design information about the "Display the qualification status of transceivers" feature implementation in SONiC.

# Definition/Abbreviation

### Table 1: Abbreviations
| **Term**                 | **Meaning**                         |
|--------------------------|-------------------------------------|
| PMON                     | Platform Monitor                    |
| XCVRD                    | Transmit Receiver Daemon            |
| API                      | Application Programming Interface   |
| REST                     | Representational State Transfer     |
| gNMI                     | gRPC Network Management Interface   |
| CLI                      | Command Line Interface              |

# 1 Feature Overview
Display the qualification status of transceivers. Based on the media type, this feature helps for identifying and displaying whether a media is vendor-qualified or not.
By this feature, Vendor platforms will be able to recognize certified and supported transceivers. 

## 1.1 Target Deployment Use Cases
NA

## 1.2 Requirements

1 Overview 
Based on the media type, this feature help us for identifying and displaying whether a media is vendor-qualified or not.

2 Functionality 
This feature helps for identifying and displaying whether a media is vendor-qualified or not. Vendor platforms will be able to recognize certified and supported transceivers.

The Media qualification can be done based on various parameters such as vendor name, part number, and/or vendor-specific product specifications encoding present in the custom info space in the EEPROM.
 
3 Interfaces 
NA

4 Configuration 
NA

5 User Interfaces 
The feature is managed through the SONiC Management Framework, including full support for Klish, REST and gNMI  

6 Serviceability 
UI show commands are provided to show the transceiver qualified detail.

7 Scaling 
NA

8 Warm Boot/ISSU 
NA

9 Platforms 
Media qualification is supported on all SONiC platforms  

10 Feature Interactions/Exclusions 
A Media qualification is typically associated with PMON/XCVRD

•	YANG extension for displaying the vendor qualification status, and

•	A platform-specific extension API for determining the vendor-qualification status.

	i) The "get_transceiver_info_dict" API will update the transceiver information by reading the EEPROM content of the media, which inturn calls the "get_static_info" API to update the media qualification status by invoking "is_qualified" API.

11 Limitations 
NA

## 1.3 Design Overview

### 1.3.1 Basic Approach
The Media qualification can be done based on various parameters such as vendor name, part number, and/or vendor-specific product specifications.

### 1.3.2 Container
No new containers are introduced for this feature. Existing PMON, Management framework and Redis containers are updated.

### 1.3.3 SAI Overview
NA

# 2 Functionality
This feature helps for identifying and displaying whether a media is vendor-qualified or not. Vendor platforms will be able to recognize certified and supported transceivers.

The Media qualification can be done based on various parameters such as vendor name, part number and/or vendor-specific product specification encoding present in the custom info space in the EEPROM.

# 3 Design

## 3.1 Overview

![](https://github.com/sundararajan-rajendr/SONiC/blob/master/images/Design_Overview.png)

### 3.1.1 Service and Docker Management
No new service or docker is introduced. PMON, management framework dockers and XCVRD daemons are used.

### 3.1.2 Packet Handling
NA

## 3.2 DB Changes

### 3.2.1 CONFIG DB
NA

### 3.2.2 APP DB
NA

### 3.2.3 STATE DB
"is_qualified" field type in TRANSCEIVER_INFO is modified as Boolean.

### 3.2.4 ASIC DB
NA

### 3.2.5 COUNTER DB
NA

### 3.2.6 ERROR DB
NA

## 3.3 Switch State Service Design

### 3.3.1 Orchestration Agent
NA

### 3.3.2 Other Processes
NA

## 3.4 SyncD
NA

## 3.5 SAI
NA

## 3.6 User Interface

### 3.6.1 Data Models

New attribute to denote media qualification state.
```
sonic-transceiver.yang:

module: sonic-transceiver
    +--rw sonic-transceiver
       +--rw TRANSCEIVER_INFO
          +--rw TRANSCEIVER_INFO_LIST* [ifname]
             +--rw ifname                 -> /prt:sonic-port/PORT/PORT_LIST/ifname
             +--rw type?                  string
             +--rw type_abbrv_name?       string
             +--rw manufacturename?       string
             +--rw modelname?             string
             +--rw hardwarerev?           string
             +--rw serialnum?             string
             +--rw media_type?            string
             +--rw power_class?           string
             +--rw revision_compliance?   string
             +--rw is_qualified?          boolean


container TRANSCEIVER_INFO {
   list TRANSCEIVER_INFO_LIST {
       .
       .
       leaf is_qualified {
           type boolean;
           description "Indicates if the component is
                qualified to be used by the system vendor.";
                  
      }
  }

openconfig-platform-ext.yang :

augment /oc-pf:components/oc-pf:component/oc-transceiver:transceiver/oc-transceiver:state {
        .
        .  
        leaf qualified {
           type boolean;
            description
             "Indicates if the component is qualified to be used by the system vendor.";
        }
   }

```

### 3.6.2 CLI

#### 3.6.2.1 Configuration Commands
NA

#### 3.6.2.2 Show Commands
```
In Click:
Usage: show interfaces transceiver summary [OPTIONS] [INTERFACENAME]

  Show interface transceiver summary information

Example:
show interfaces transceiver summary
-------------------------------------------------------------------------------------------------------------------
Interface    Name                                    Vendor           Part No.         Serial No.        Qualified
-------------------------------------------------------------------------------------------------------------------
Ethernet0    QSFP+ 40GBASE-CR4-DAC-1.0M              Amphenol         616760001        CN0TCPM237G21E0   True
Ethernet4    N/A                                     N/A              N/A              N/A               False
Ethernet8    QSFP+ 40GBASE-CR4-DAC-1.0M              Amphenol         616760001        CN0TCPM23731JW6   False
Ethernet12   N/A                                     N/A              N/A              N/A               False
Ethernet16   N/A                                     N/A              N/A              N/A               False
Ethernet20   N/A                                     N/A              N/A              N/A               False
Ethernet24   QSFP+ 40GBASE-CR4-DAC-1.0M              Amphenol         616760001        CN0TCPM237O27YA   False
Ethernet28   N/A                                     N/A              N/A              N/A               False
Ethernet32   QSFP+ 40GBASE-CR4-DAC-1.0M              Amphenol         599690001        APF11510011VTY    False
```

```
In Klish:
Usage: show interface transceiver [Eth <slot/port[/subport]-slot/port[/subport]> [summary]] [summary]

show interface transceiver summary
-------------------------------------------------------------------------------------------------------------------
Interface    Name                                    Vendor           Part No.         Serial No.        Qualified
-------------------------------------------------------------------------------------------------------------------
Ethernet0    QSFP+ 40GBASE-CR4-DAC-1.0M              Amphenol         616760001        CN0TCPM237G21E0   True
Ethernet4    N/A                                     N/A              N/A              N/A               False
Ethernet8    QSFP+ 40GBASE-CR4-DAC-1.0M              Amphenol         616760001        CN0TCPM23731JW6   False
Ethernet12   N/A                                     N/A              N/A              N/A               False
Ethernet16   N/A                                     N/A              N/A              N/A               False
Ethernet20   N/A                                     N/A              N/A              N/A               False
Ethernet24   QSFP+ 40GBASE-CR4-DAC-1.0M              Amphenol         616760001        CN0TCPM237O27YA   False
Ethernet28   N/A                                     N/A              N/A              N/A               False
Ethernet32   QSFP+ 40GBASE-CR4-DAC-1.0M              Amphenol         599690001        APF11510011VTY    False

show interface transceiver Eth 1/21

Eth1/21
---------------------------------------------------------------------
Attribute                :  Value/State
---------------------------------------------------------------------
cable-length(m)          :  0
connector-type           :  LC
date-code                :  2019-04-21
display-name             :  SFP28 25GBASE-SR-NOF
form-factor              :  SFP28
max-module-power(Watts)  :  2.5
max-port-power(Watts)    :  2.5
qualified                :  True
present                  :  PRESENT
serial-no                :  CN07919194K04K9
vendor                   :  DELL
vendor-oui               :  00-17-6A
vendor-part              :  W4GPP
vendor-rev               :  A0

show interface transceiver Eth 1/21 summary

-------------------------------------------------------------------------------------------------------------------
Interface    Name                                    Vendor           Part No.         Serial No.        Qualified
-------------------------------------------------------------------------------------------------------------------
Eth1/21      SFP28 25GBASE-SR-NOF                    DELL             W4GPP            CN07919194K04K9   True
```

#### 3.6.2.3 Exec Commands
NA

### 3.6.3 REST API Support
New attribute url
```
/openconfig-platform:components/component={name}/openconfig-platform-transceiver:transceiver/state/openconfig-platform-ext:qualified"
```

### 3.6.4 gNMI Support

(https://github.com/project-arlo/sonic-mgmt-common/blob/dell_sonic_share/models/yang/sonic/sonic-transceiver.yang)

## 3.7 Warm Boot Support
NA

## 3.8 Upgrade and Downgrade Considerations
In case of upgrade, "is_qualified" field type in TRANSCEIVER_INFO of STATE_DB is modified as "True/False" from "Yes/No" and Viceversa for downgrade.,

## 3.9 Resource Needs
Media(DAC/Optics)

# 4 Flow Diagrams
![](https://github.com/sundararajan-rajendr/SONiC/blob/master/images/Flow_Diagram.png)

# 5 Error Handling
NA

# 6 Serviceability and Debug
   1. sonic-db-cli STATE_DB HGETALL "TRANSCEIVER_INFO"
   2. hexdump commands can be used to dump the EEPROM content.

# 7 Scalability
NA

# 8 Platform
All SONiC infrastructure (e.g., xcvrd) and interfaces (management framework and show-commands) will account for vendor platforms that do not implement the transceiver qualification API.

# 9 Security and Threat Model
NA

# 10 Limitations
NA

# 11 Unit Test
  1. sonic-db-cli STATE_DB HGETALL "TRANSCEIVER_INFO"
  2. show interface transceiver summary
  3. show interfaces transceiver summary
  4. show interface transceiver Eth <>
  5. show interface transceiver Eth <> summary

# 12 Internal Design Information

## 12.1 IS-CLI Compliance

|CLI Command                                 |Compliance    |IS-CLI Command     | Link to the web site identifying the IS-CLI command (if applicable)|
|:------------------------------------------:|:------------:|:-----------------:|-----------------------------------|
|show interfaces transceiver summary         | IS-CLI       |                   |                                   |
|show interface transceiver summary          | IS-CLI       |                   |                                   |
|show interface transceiver Eth <>           | IS-CLI       |                   |                                   |
|show interface transceiver Eth <> summary   | IS-CLI       |                   |                                   |

## 12.2 SONiC Packaging
Enterprise

## 12.3 Broadcom Silicon Considerations
NA

## 12.4 Design Alternatives
NA

## 12.5 Release Matrix
Sonic_4.0.0
