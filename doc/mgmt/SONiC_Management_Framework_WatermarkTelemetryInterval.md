# Watermark Telemetry
Watermark telemetry interval support in management framework.
# High Level Design Document
#### Rev 0.1

# Table of Contents
  * [List of Tables](#list-of-tables)
  * [Revision](#revision)
  * [About This Manual](#about-this-manual)
  * [Scope](#scope)




# Revision
| Rev |     Date    |       Author       | Change Description                |
|:---:|:-----------:|:------------------:|-----------------------------------|
| 0.1 | 09/30/2019  |   Adedamola Adetayo      | Initial version                   |

# About this Manual
This document provides general information about watermark telemetry show and configure interval in SONiC cli using the management framework.
# Scope
This document describes the high level design of watermark telemetry interval as well as unit test cases.  Reference:
https://github.com/Azure/sonic-utilities/blob/master/doc/Command-Reference.md#watermark-configuration-and-show



# 1 Feature Overview

This feature will enable users to configure watermark telemetry interval in SONiC management framework with CLI, REST or gNMI. It is used to configure the interval in seconds between the device's sample of a telemetry data source.


## 1.1 Requirements
### 1.1.1 Functional Requirements
N/A

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
Design watermark telemetry interval support using transformer.
### 1.2.2 Container
Changes will be made in the sonic-mgmt-framework container
Added files:
1. XML file for the CLI
2. Python script to handle CLI request (actioner)
3. Jinja template to render CLI output (renderer)
4. SONiC YANG model
### 1.2.3 SAI Overview
N/A
# 2 Functionality
## 2.1 Target Deployment Use Cases
N/A
## 2.2 Functional Description
N/A

# 3 Design
## 3.1 Overview
## 3.2 DB Changes
### 3.2.1 ConfigDB
This feature will enable users to make/show configuration changes to ConfigDB
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
Supported SONiC YANG attributes:
**sonic-watermark-telemetry.yang**
````
module: sonic-watermark-telemetry
    +--rw sonic-watermark-telemetry
       +--rw WATERMARK_TABLE
          +--rw interval?   uint64
````

### 3.6.2 CLI
#### 3.6.2.1 Configuration Commands

Set watermark telemetry interval:-
Default value is 120 

````
watermark telemetry interval <value>
````
````
sonic(config)# watermark telemetry interval 200

````

#### 3.6.2.2 Show Commands
````
sonic# show watermark telemetry interval
  
   Telemetry interval: 200 second(s)
````


#### 3.6.2.3 Debug Commands
N/A
#### 3.6.2.4 IS-CLI Compliance
N/A


### 3.6.3 REST API Support

**GET**

-`/sonic-watermark-telemetry:WATERMARK_TABLE/interval`

**PATCH**

-`/sonic-watermark-telemetry:WATERMARK_TABLE/interval`


# 4 Flow Diagrams
N/A
# 5 Error Handling
N/A
# 6 Serviceability and Debug
N/A
# 7 Warm Boot Support
N/A
# 8 Scalability
N/A

# 9 Unit Test
| Test Name | Test Description |
| :------ | :----- |
Set  interval via CLI | Validate watermark telemetry interval in ConfigDB
Show interval via CLI | Verify show watermark telemetry interval
Set  interval via REST | Validate watermark telemetry interval in ConfigDB
Show interval via REST | Verify show watermark telemetry interval
Set  interval via gNMI | Validate watermark telemetry interval in ConfigDB
Show interval via gNMI | Verify show watermark telemetry interval

