# Feature Name
Implement management support using CLI/REST/gNMI SONiC interfaces for Configuration Management Operations.

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
| 0.1 | 10/14/2019  |  Bhavesh Nisar      | Initial version                  |

# About this Manual
This document provides information on the management interfaces for Configuration Management Operations.
# Scope
The scope of this document is limited to the northbound interfaces supported by the new management framework interface. These are the CLI, gNMi and REST interface. The backend functional behavior of the operations is not modified and can be referred in the SONic Documents.


# Definition/Abbreviation

### Table 1: Abbreviations
| **Term**                 | **Meaning**                         |
|--------------------------|-------------------------------------|
| NBI                      | North Bound Interface                    |

# 1 Feature Overview

The feature addresses the equivalent of the existing cLick CLI commands.  The  implementation is contained within the Management Framework container.
Their is no openconfig data yang defined for this feature. A new SONiC yang will be introduced for the NBI.

## 1.1 Requirements

### 1.1.1 Functional Requirements

### 1.1.2 Configuration and Management Requirements

CLI Commands
1. Command : config reload
   Attributes: file path
   Validation failure message: Invalid value for "filename": Path "/etc/config/config_db2.json" does not exist.
   Prompt message: Clear current config and reload config from the file /etc/sonic/config_db.json? [y/N]:
   Help: Clear current configuration and import a previous saved config DB dump
         file.


2. Command : config save
   Attributes: file Path
   Prompt Message: Existing file will be overwritten, continue? [y/N]
   Help: Export current config DB to a file on disk.

3. Command: config load
   Attributes: file Path
   Validation: Invalid value for "filename": Path "/etc/config/config_db2.json"  does not exist.
   Prompt message: Load config from the file /etc/sonic/config_db.json? [y/N]:
   Help: Import a previous saved config DB dump file.

4. Command: config load_mgmt_config
   Attributes: file path
   Validation: Error: Invalid value for "filename": Path "hello" does not exist.
   Invalid value for "filename": Path "/etc/sonic/device_desc.xml" does not  exist.
   Help: Reconfigure hostname and mgmt interface based on device description file.

5. Command: config load_minigraph
   Validation: IOError: Error reading file '/etc/sonic/minigraph.xml': failed to load external entity "/etc/sonic/minigraph.xml"
   Prompt Message: Reload config from minigraph? [y/N]: y
   Help: Reconfigure based on minigraph.


### 1.1.3 Scalability Requirements
key scaling factors
### 1.1.4 Warm Boot Requirements

## 1.2 Design Overview
### 1.2.1 Basic Approach


### 1.2.2 Container
This feature is contained within the sonic-mgmt-framework container.

### 1.2.3 SAI Overview
Not Applicable.

# 2 Functionality
## 2.1 Target Deployment Use Cases
Wordy description, with diagrams if possible
## 2.2 Functional Description
Wordy description

# 3 Design
## 3.1 Overview

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
Describe changes to other process within SwSS if applicable.

## 3.4 SyncD
Describe changes to syncd if applicable.

## 3.5 SAI
Describe [new/existing] SAI APIs used by this feature.

## 3.6 User Interface
### 3.6.1 Data Models

A new sonic yang (sonic-config-mgmt.yang) provides the interface for configuration and status.

module: sonic-config-mgmt

rpcs:
  +---x save_config
  |  +---w input
  |  |  +---w file_path?   string
  |  +--ro output
  |     +--ro status?   string
  +---x load_config
  |  +---w input
  |  |  +---w file_path?   string
  |  +--ro output
  |     +--ro status?   string
  +---x reload_config
  |  +---w input
  |  |  +---w file_path?   string
  |  +--ro output
  |     +--ro status?   string
  +---x load_mgmt_config
  |  +---w input
  |  |  +---w file_path?   string
  |  +--ro output
  |     +--ro status?   string
  +---x load_minigraph
     +---w input
     |  +---w file_path?   string
     +--ro output
        +--ro status?   string


### 3.6.2 CLI
#### 3.6.2.1 Configuration Commands
#### 3.6.2.2 Show Commands


#### 3.6.2.3 Debug Commands
#### 3.6.2.4 IS-CLI Compliance
The following table maps SONIC CLI commands to corresponding IS-CLI commands. The compliance column identifies how the command comply to the IS-CLI syntax:

- **IS-CLI drop-in replace**  – meaning that it follows exactly the format of a pre-existing IS-CLI command.
- **IS-CLI-like**  – meaning that the exact format of the IS-CLI command could not be followed, but the command is similar to other commands for IS-CLI (e.g. IS-CLI may not offer the exact option, but the command can be positioned is a similar manner as others for the related feature).
- **SONIC** - meaning that no IS-CLI-like command could be found, so the command is derived specifically for SONIC.

|       CLI Command       | Compliance | click CLI
|:-----------------------:|:-----------|-----------
| config save             | SONIC      |  config save
| config load             | SONIC      |  config load
| config reload           | SONIC      |  config reload
| config load-mgmt-config | SONIC      |  config load_mgmt_config
| config load-minigraph   | SONIC      |  config load_minigraph

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
1. Execute command 'config save'. Default path applied. Verify configdb.json file.
2. Execute 'config save' with filepath. Verify config saved in the given filepath.
2. Execute 'config load'. Verify config loaded in config Db.
3. Execute 'config load' with filepath. Verify config loaded in config Db.
3. Execute 'config reload'. Verify configdb reset with new loaded config from configDb.json.
4. Execute 'config load-mgmt-config'. Verify mgmt-config loaded into Db.
5. Execute 'config load-mgmt-config' with filepath. Verify mgmt-config loaded into Db
5. Execute 'config load minigraph'. Verify config loaded into Db.
