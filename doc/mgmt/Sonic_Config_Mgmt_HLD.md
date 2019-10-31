# Feature Name
Implement management support using CLI/REST/gNOI interfaces for Configuration Management Operations.

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
| 0.1 | 10/30/2019  |  Bhavesh Nisar     | Initial version                   |

# About this Manual
This document provides information on the management interfaces for Configuration Management Operations.
# Scope
The scope of this document is limited to the northbound interfaces supported by the new management framework, i.e., CLI, gNOI and REST. The functional behavior of these operations is not modified and can be referred in the SONiC Documents.

# Definition/Abbreviation

### Table 1: Abbreviations
| **Term**                 | **Meaning**                         |
|--------------------------|-------------------------------------|
| NBI                      | North Bound Interface               |

# 1 Feature Overview

The feature addresses the equivalent of the existing cLick CLI commands.  The  implementation is contained within the Management Framework container.
Their is no openconfig data yang defined for this feature. A new SONiC yang will be introduced for the NBI.

## 1.1 Requirements

### 1.1.1 Functional Requirements
Not Applicable

### 1.1.2 Configuration and Management Requirements

Operations:
1. Save running configuration.  
   CLI Command : write memory  
   EXEC Level  
   Parameter: filename  
   Default: /etc/sonic/config_db.json  

2. Copy config to redis:configDB and reload.  
   CLI Command : copy-to-running-config  
   EXEC Level  
   Parameter: filename  
   Default: /etc/sonic/config_db.json  

3. Update config to redis:configDB.  
   CLI Command: update-to-running-config  
   EXEC Level  
   Parameter: filename  
   Default: /etc/sonic/config_db.json  

The click CLI additionally supports 'config load_mgmt' and 'config load_minigraph' commands. These operations are SONiC specific. The new sonic management framework will handle the management interface related configuration through the NBI interface. The current json format of configDB replaces the xml format of minigraph.

### 1.1.3 Scalability Requirements
Not Applicable

### 1.1.4 Warm Boot Requirements
Not Applicable

## 1.2 Design Overview
### 1.2.1 Basic Approach
The operations are invoked via RPC construct of the yang interface. The sonic management framework callback is defined in the sonic-annotation.yang file. The operations are executed on the host service via the dBus framework.

### 1.2.2 Container
This feature is contained within the sonic-mgmt-framework container.

### 1.2.3 SAI Overview
Not Applicable.

# 2 Functionality
## 2.1 Target Deployment Use Cases
Not Applicable

## 2.2 Functional Description
Not Applicable

# 3 Design
## 3.1 Overview
Not Applicable

## 3.2 DB Changes
Describe changes to existing DBs or any new DB being added.
### 3.2.1 CONFIG DB
### 3.2.2 APP DB
### 3.2.3 STATE DB
### 3.2.4 ASIC DB
### 3.2.5 COUNTER DB

## 3.3 Switch State Service Design
### 3.3.1 Orchestration Agent
### 3.3.2 Other Process
Not Applicable

## 3.4 SyncD
Not Applicable

## 3.5 SAI
Not Applicable

## 3.6 User Interface
### 3.6.1 Data Models

A new sonic yang (sonic-config-mgmt.yang) provides the interface for configuration and status.

```
module: sonic-config-mgmt

rpcs:
  +---x save_config
  |  +---w input
  |  |  +---w file_path?   string
  |  +--ro output
  |     +--ro status?   string
  +---x reload_config
  |  +---w input
  |  |  +---w file_path?   string
  |  +--ro output
  |     +--ro status?   string
  +---x load_config
  |  +---w input
  |  |  +---w file_path?   string
  |  +--ro output
  |     +--ro status?   string

```

### 3.6.2 CLI
#### 3.6.2.1 Configuration Commands
As described in section 1.1.2

#### 3.6.2.2 Show Commands
Not Applicable

#### 3.6.2.3 Debug Commands
Not Applicable

#### 3.6.2.4 IS-CLI Compliance
The following table maps SONIC CLI commands to corresponding IS-CLI commands. The compliance column identifies how the command comply to the IS-CLI syntax:

- **IS-CLI drop-in replace**  – meaning that it follows exactly the format of a pre-existing IS-CLI command.
- **IS-CLI-like**  – meaning that the exact format of the IS-CLI command could not be followed, but the command is similar to other commands for IS-CLI (e.g. IS-CLI may not offer the exact option, but the command can be positioned is a similar manner as others for the related feature).
- **SONIC** - meaning that no IS-CLI-like command could be found, so the command is derived specifically for SONIC.

|       CLI Command       | Compliance   | click CLI                    | Deviation
|:-----------------------:|:-------------|------------------------------|---------------
| write memory            | IS-CLI-like  |  config save                 |
| copy-to-running-config  |              |  config reload               | This incorporates copy and restart of services.
| update-to-running-config|              |  config load                 | This does an update to running config.

**Deviations from IS-CLI:** If there is a deviation from IS-CLI, Please state the reason(s).


### 3.6.3 REST API Support
Rest API is supported through the sonic-config-mgmt.yang.

# 4 Flow Diagrams
Provide flow diagrams for inter-container and intra-container interactions.

# 5 Error Handling
Provide details about incorporating error handling feature into the design and functionality of this feature.

# 6 Serviceability and Debug
Logging, counters, stats, trace considerations. Please make sure you have incorporated the debugging framework feature. e.g., ensure your code registers with the debugging framework and add your dump routines for any debug info you want to be collected.

# 7 Warm Boot Support
Describe expected behavior and any limitation.

# 8 Scalability
Describe key scaling factor and considerations.

# 9 Unit Test
List unit test cases added for this feature including warm boot.
CLI test cases
1. Execute 'write memory'. Default path applied. Verify config_db.json file.
2. Execute 'write memory' with filepath. Verify config saved in the given filepath.
3. Execute 'copy-to-running-config'. Verify default config loaded in redis:configDB with reload.
4. Execute 'update-to-running-config' with filepath. Verify config loaded in redis:configDB with reload.
