# Feature Name
Implement management support using CLI/REST/gNMI SONiC interfaces for Error Handling Framework.

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
The  scope of this document is within the bounds of the functionality provided by the SONiC Management Framework. It gives a high level design of the north-bound interfaces for managing Error Handling framework. <br>
For reference to the new Management Framework refer to https://github.com/project-arlo/SONiC/blob/master/doc/mgmt/Management%20Framework.md <br>
This document does not discuss the workings of the Error Handling feature. This can be referred at <br>
https://github.com/Azure/SONiC/blob/b8816dfac04b17e0e7bc7a68361e4feab1ae47c5/doc/error-handling/error_handling_design_spec.md

# Definition/Abbreviation

### Table 1: Abbreviations
| **Term**                 | **Meaning**                         |
|--------------------------|-------------------------------------|
| NBI                      | North Bound Interface               |

# 1 Feature Overview

The feature addresses the north-bound interfaces (CLI/REST/gNMI) for configuring  and fetching the data from the back-end to the user using SONiC managment framework. The  implementation is contained within the Management Framework container.

Their is no openconfig data yang defined for this feature. A new SONiC yang will be introduced for the NBI.

The following operations will be exposed by the SONiC yang interface:
1. Get errors from redis:ErrorDB.
2. Clear error tables in redis:ErrorDB.

## 1.1 Requirements
The requirements for the feature are derived from the Error Handling Framework Specification.
https://github.com/Azure/SONiC/blob/b8816dfac04b17e0e7bc7a68361e4feab1ae47c5/doc/error-handling/error_handling_design_spec.md

### 1.1.1 Functional Requirements
Provide yang interface for REST and CLI commands for feature configuration and management.

### 1.1.2 Configuration and Management Requirements

- CLI commands

Mode: Show
1. show error-database \[*tablename*\] <br>
  Parameter: *tablename*: ERROR_NEIGH_TABLE, ERROR_ROUTE_TABLE <br>

Mode: Configuration
1. no error-database \[*table-name*\] <br>
  Parameter: *tablename*: ERROR_NEIGH_TABLE, ERROR_ROUTE_TABLE <br>


REST support
gNMI support for show errors tables.


### 1.1.3 Scalability Requirements
Not Applicable
### 1.1.4 Warm Boot Requirements
Not Applicable

## 1.2 Design Overview
### 1.2.1 Basic Approach
The 'show' command will fetch the error table information from ErrorDB(8). This is through yang show construct.
The 'clear error-database [*tablename*]' is handle through yang rpc construct. It provides the mechanism to register a callback. The callback will send notification to ErrorDB subscribers, which in response will clear the tables.

### 1.2.2 Container
This feature is contained within the sonic-mgmt-framework container.

### 1.2.3 SAI Overview
Not Applicable.

# 2 Functionality
## 2.1 Target Deployment Use Cases
Not applicable
## 2.2 Functional Description
Wordy description

# 3 Design
## 3.1 Overview
Not applicable.
## 3.2 DB Changes
Not applicable.
## 3.3 Switch State Service Design
Not applicable.

## 3.4 SyncD
Not applicable.

## 3.5 SAI
Not applicable.

## 3.6 User Interface
### 3.6.1 Data Models

A new sonic yang (sonic-error.yang) provides the interface for configuration and status.
```
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
```

### 3.6.2 CLI
#### 3.6.2.1 Configuration Commands
#### 3.6.2.2 Show Commands
```
sonic# show error-database
  ERROR_NEIGH_TABLE  Show error neighbor table
  ERROR_ROUTE_TABLE  Show error route table
  <cr>

sonic# show error-database

------------------------------
ERROR_ROUTE_TABLE
------------------------------
Route      : 20.20.20.0/24
Operation  :  create
------------------------------
Route      : 30.20.20.0/24
Operation  :  create
------------------------------
Route      : 31.20.20.0/24
Operation  :  create
------------------------------
------------------------------
ERROR_NEIGH_TABLE
------------------------------
Interface  : Ethernet36
Prefix     : 2.2.2.2/24
Operation  : create
------------------------------

sonic#
sonic# show error-database ERROR_NEIGH_TABLE
------------------------------
ERROR_NEIGH_TABLE
------------------------------
Interface  : Ethernet36
Prefix     : 2.2.2.2/24
Operation  : create
------------------------------
sonic# show error-database ERROR_ROUTE_TABLE

------------------------------
ERROR_ROUTE_TABLE
------------------------------
Route      : 20.20.20.0/24
Operation  :  create
------------------------------
Route      : 30.20.20.0/24
Operation  :  create
------------------------------
Route      : 31.20.20.0/24
Operation  :  create
------------------------------


sonic(config)# no error-database
  ERROR_NEIGH_TABLE  Delete error neighbor table
  ERROR_ROUTE_TABLE  Delete error route table
  <cr>
```


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
Rest API support is through the yang interface.

# 4 Flow Diagrams
Provide flow diagrams for inter-container and intra-container interactions.

# 5 Error Handling
Not applicable.

# 6 Serviceability and Debug
Not applcable.

# 7 Warm Boot Support
Not applicable.

# 8 Scalability
Not applicable.

# 9 Unit Test
List unit test cases added for this feature including warm boot.
1. Execute CLI 'show error-database'. Add data to ErrorDB|Routetable, NeighborTable. Verify all errors are fetched from DB and rendered on CLI.
2. Execute CLI 'show error-database [*tablename*]'. Verify only specific table data is retrieved from database.
3. Execute CLI 'clear error-database'. Verify all table entries in ErrorDB are deleted.
4. Execute CLI 'clear error-database [*tablename*]'. Verify table is delete from ErrorDB.
