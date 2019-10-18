# Feature Name
Implement management support using CLI/REST/gNMI SONiC management framework interfaces for Error Handling Framework.

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
This document provides information about the management interfaces for Error Handling Framework.
# Scope
The  scope of this document is within the bounds of the functionality provided by the new SONiC Management Framework. It gives a High Level Design of the north-bound interfaces for managing Error Handling framework.
For reference to the new Management Framework refer to https://github.com/project-arlo/SONiC/blob/master/doc/mgmt/Management%20Framework.md.
This document also does not discuss the workings of the Error Handling feature. It can be referred at
https://github.com/Azure/SONiC/blob/b8816dfac04b17e0e7bc7a68361e4feab1ae47c5/doc/error-handling/error_handling_design_spec.md

# Definition/Abbreviation

### Table 1: Abbreviations
| **Term**                 | **Meaning**                         |
|--------------------------|-------------------------------------|
| NBI                      | North Bound Interface                    |

# 1 Feature Overview

The feature addresses the north-bound interfaces (CLI/REST/gNMI) for configuring  and fetching the data from the back-end to the user using SONiC managment framework.  The  implementation is contained within the Management Framework container.

Their is no openconfig data yang defined for this feature. A new SONiC yang will be introduced for the NBI.

The following operations will be exposed by the SONiC yang interface:
1. Get errors from Db.
2. Clear error tables in Db.

## 1.1 Requirements
The requirements for the feature are derived from the Error Handling Framework Specification.
https://github.com/Azure/SONiC/blob/b8816dfac04b17e0e7bc7a68361e4feab1ae47c5/doc/error-handling/error_handling_design_spec.md

### 1.1.1 Functional Requirements
Provide yang interface for REST and CLI commands for feature configuration and management.

### 1.1.2 Configuration and Management Requirements

- CLI commands

Mode: Show
1. show error-database table <table-name>
2. show error-database <CR>

Mode: Configuration
1. no error-database table <table-name>
2. no error-database

- REST support
- gNMI support


### 1.1.3 Scalability Requirements
key scaling factors
### 1.1.4 Warm Boot Requirements

## 1.2 Design Overview
### 1.2.1 Basic Approach
The 'show' command will fetch the error table information from ErrorDB(8). This is through yang show construct.
The 'clear error-database <table>'' is handle through yang rpc construct. It provides the mechanism to register a callback. The callback will send notification to ErrorDb subscribers, which in response will clear the tables.

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

A new sonic yang (sonic-error.yang) provides the interface for configuration and status.

module: sonic-error
    +--rw sonic-error
       +--rw ERROR_NEIGH_TABLE
       |  +--rw ERROR_NEIGH_TABLE_LIST* [ifname prefix]
       |     +--rw ifname       string
       |     +--rw prefix       inet:ip-address
       |     +--rw operation?   enumeration
       |     +--rw neigh?       ietf:mac-address
       |     +--rw family?      string
       |     +--rw rc?          string
       +--rw ERROR_ROUTE_TABLE
          +--rw ERROR_ROUTE_TABLE_LIST* [prefix]
             +--rw prefix       inet:ip-address
             +--rw operation?   enumeration
             +--rw nexthop?     string
             +--rw intf?        string
             +--rw rc?          string

  rpcs:
    +---x clear_database
       +---w input
       |  +---w tablename?   string
       +--ro output
          +--ro status?   string


### 3.6.2 CLI
#### 3.6.2.1 Configuration Commands
#### 3.6.2.2 Show Commands
sonic# show error-database

ERROR ROUTE TABLE
Route      : 20.20.20.0/24
Operation  :  create
----------------------------------
Route      : 30.20.20.0/24
Operation  :  create
----------------------------------
Route      : 31.20.20.0/24
Operation  :  create
----------------------------------

ERROR NEIGHBOR TABLE
----------------------------------
Interface  : Ethernet36
Prefix     : 2.2.2.2/24
Operation  : create
----------------------------------

#### 3.6.2.3 Debug Commands
#### 3.6.2.4 IS-CLI Compliance
The following table maps SONIC CLI commands to corresponding IS-CLI commands. The compliance column identifies how the command comply to the IS-CLI syntax:

- **IS-CLI drop-in replace**  – meaning that it follows exactly the format of a pre-existing IS-CLI command.
- **IS-CLI-like**  – meaning that the exact format of the IS-CLI command could not be followed, but the command is similar to other commands for IS-CLI (e.g. IS-CLI may not offer the exact option, but the command can be positioned is a similar manner as others for the related feature).
- **SONIC** - meaning that no IS-CLI-like command could be found, so the command is derived specifically for SONIC.

|CLI Command                      |Compliance|
|:-------------------------------:|:---------|
|  show error-database            |  SONIC   |
|  no error-database              |  SONIC   |

**Deviations from IS-CLI:** If there is a deviation from IS-CLI, Please state the reason(s).



### 3.6.3 REST API Support
Rest API support is through

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
1. Execute CLI 'show error-database'. Add data to ErrorDB|Routetable, NeighborTable. Verify all errors are fetched from DB and rendered on CLI.
2. Execute CLI 'show error-database table <tablename>'. Verify only specific table data is retrieved from database.
3. Execute CLI 'clear error-database'. Verify all table entries in ErrorDB are deleted.
4. Execute CLI 'clear error-database table <tablename>'. Verify table is delete from ErrorDb.
