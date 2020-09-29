# Management CLI

# High Level Design Document
# Table of Contents
- [1 Feature Overview](#1-Feature-Overview)
    - [1.1 Target Deployment Use Cases](#11-Target-Deployment-Use-Cases)
    - [1.2 Requirements](#12-Requirements)
    - [1.3 Design Overview](#13-Design-Overview)
        - [1.3.1 Basic Approach](#131-Basic-Approach)
        - [1.3.2 Container](#132-Container)
        - [1.3.3 SAI Overview](#133-SAI-Overview)
- [2 Functionality](#2-Functionality)
- [3 Design](#3-Design)
    - [3.1 Overview](#31-Overview)
        - [3.1.1 Service and Docker Management](#311-Service-and-Docker-Management)
        - [3.1.2 Packet Handling](#312-Packet-Handling)
    - [3.2 DB Changes](#32-DB-Changes)
        - [3.2.1 CONFIG DB](#321-CONFIG-DB)
        - [3.2.2 APP DB](#322-APP-DB)
        - [3.2.3 STATE DB](#323-STATE-DB)
        - [3.2.4 ASIC DB](#324-ASIC-DB)
        - [3.2.5 COUNTER DB](#325-COUNTER-DB)
        - [3.2.6 ERROR DB](#326-ERROR-DB)
    - [3.3 Switch State Service Design](#33-Switch-State-Service-Design)
        - [3.3.1 Orchestration Agent](#331-Orchestration-Agent)
        - [3.3.2 Other Processes](#332-Other-Processes)
    - [3.4 SyncD](#34-SyncD)
    - [3.5 SAI](#35-SAI)
    - [3.6 User Interface](#36-User-Interface)
        - [3.6.1 Data Models](#361-Data-Models)
        - [3.6.2 CLI](#362-CLI)
        - [3.6.2.1 Configuration Commands](#3621-Configuration-Commands)
        - [3.6.2.2 Show Commands](#3622-Show-Commands)
        - [3.6.2.3 Exec Commands](#3623-Exec-Commands)
        - [3.6.3 REST API Support](#363-REST-API-Support)
        - [3.6.4 gNMI Support](#364-gNMI-Support)
     - [3.7 Warm Boot Support](#37-Warm-Boot-Support)
     - [3.8 Upgrade and Downgrade Considerations](#38-Upgrade-and-Downgrade-Considerations)
     - [3.9 Resource Needs](#39-Resource-Needs)
- [4 Flow Diagrams](#4-Flow-Diagrams)
- [5 Error Handling](#5-Error-Handling)
- [6 Serviceability and Debug](#6-Serviceability-and-Debug)
- [7 Scalability](#7-Scalability)
- [8 Platform](#8-Platform)
- [9 Limitations](#9-Limitations)
- [10 Unit Test](#10-Unit-Test)
- [11 Internal Design Information](#11-Internal-Design-Information)
    - [11.1 IS-CLI Compliance](#111-IS-CLI-Compliance)
    - [11.2 Broadcom Packaging](#112-Broadcom-SONiC-Packaging)
    - [11.3 Broadcom Silicon Considerations](#113-Broadcom-Silicon-Considerations)    
    - [11.4 Design Alternatives](#114-Design-Alternatives)
    - [11.5 Broadcom Release Matrix](#115-Broadcom-Release-Matrix)

# List of Tables
[Table 1: Abbreviations](#table-1-Abbreviations)

# Revision
| Rev |     Date    |       Author       | Change Description                |
|:---:|:-----------:|:------------------:|-----------------------------------|
| 0.1 | <09/24/2020>|   Eric Seifert     | Initial version                   |

# About this Manual
This document provides comprehensive functional and design information about the Management CLI feature implementation in SONiC.

# Definition/Abbreviation

### Table 1: Abbreviations
| **Term**                 | **Meaning**                         |
|--------------------------|-------------------------------------|
| REST                     | Representational state transfer     |
| gNMI                     | Google Network Managemnt Interface  |
| JWT                      | Java Web Token                      |

# 1 Feature Overview

A CLI is needed to manage the options for running the management services, REST and telemetry. The CLI will include setting the client authentication modes, token validity, token refresh interval and service operational state. This also requires the creation of a sonic YANG model to contain these fields which will be named sonic-mgmt-svcs. With the model defined, these fields will also be configurable via REST and gNMI interfaces. Along with the CLI and YANG model, changes will be required to REST server, sonic-buildimage and sonic-utilities.

## 1.1 Target Deployment Use Cases

This is a general feature that applies to all deployments.

## 1.2 Requirements

1. Overview
  The CLI is needed to facilitate configuring the management services in a cleaner and more consistant manner than manually editing the fields in the redis config DB.
2. Functionality
  * Sonic model will contain the fields defined above for REST and gNMI and map to the existing redis fields. The model will be called sonic-mgmt-svcs.
  * A CLI (XML and actioner) will be created to modify these fields via the CLI. 
  * The fields can also be modified via REST and gNMI.
  * Services will be restarted when their configurations are changed.
3. Interfaces
  * The CLI will be available on the Management interface(s) only
4. Configurability
  * This feature is for the configuration of mgmt services.
5. User Interfaces
  * The main interface is the CLI, but is also available via REST and gNMI.
6. Serviceability
  N/A
7. Scaling
  N/A
8. Warm Boot/ISSU
  N/A
9. Platforms
  * All platforms
10. Feature Interactions/Exclusions
  * Changing REST configuration will require restarting the CLI.
  * Changing telemetry config will result in telemetry being temporarily unavailable.
11. Limitations
    * This is available via the Management interface(s) only.
    * After making a configuration change to REST or gNMI, those services will be restarted and will not be available until they have fully restarted.
    * Changing REST configuration will require restarting the CLI.
    * Disabling REST services only shuts down the external port, the service must still run for the CLI to function.
    * Disabling Telemetry service will shut down main telemetry process, but Dialout-Telemetry process may still run.
    * There is not currently a CLI for setting host and CA certificates. This will be addressed in a later HLD when PKI is available.

2 Functionality

2.0 Top level managment CLI.
2.0.1 - Sub-command for either rest-server or telemetry-server.
2.1.0 - Server Specific commands are the same for both REST and telemetry.
2.1.1 - Port command taking valid range for tcp ports 1-65535
2.1.2 - token validity in seconds
2.1.3 - token refresh-interval in seconds
2.1.4 - enable boolean to enable the service



## 1.3 Design Overview
### 1.3.1 Basic Approach

The model will be called sonic-mgmt-svcs.yang and include the following fields for both REST and gNMI:

    1) port
    2) jwt_valid
    3) jwt_refresh
    4) enable
    5) client_auth

The CLI actioner script will send the appropriate HTTP requests to update the fields as desired.

The REST server needs to be modified to include a command line option to enable/disable the external facing port, with the default being enabled. This option will be provided by the "enable" field above. When enable is false, the external port will be disabled and REST will only be available via the unix socket for use by the CLI.

The Host Services script needs to be modified to check the "enable" field of telemetry. If the telemetry "enable" field is changed to false, instead of restarting the service as it usually does for configuration changes, it should shut down the telemetry service. The telemetry container will still be running and the dialout-telemetry process will still be running, but the telemetry process will be stopped. If the "enable" field is set to true, host services will then start the service.

The telemetry startup script will also need to be modified to check the "enable" key to either enable or disable telemetry service in the container.


### 1.3.2 Container

No new containers are introduced for this feature. Existing Mgmt containers are updated.

### 1.3.3 SAI Overview

No new or existing SAI services are required


# 2 Functionality

* A sonic model will be created to map to existing redis DB keys for the fields to allow for backwards compatibility. 
* A CLI XML template with an actioner script will be created to configure the appropriate model fields.


# 3 Design
## 3.1 Overview

This feature will involve the mgmt-framework and telemetry containers as well as the host-services on the host. It will be used to modify configuration held in the config DB in redis.

### 3.1.1 Service and Docker Management

No new dockers will be introduced. Host services will be used to restart and/or stop/start the REST and telemetry services running in their respective containers.

### 3.1.2 Packet Handling
N/A

## 3.2 DB Changes
All of the fields except "enable" already exist in the CONFIG DB. However, they were not mapped to a sonic yang, and the format of some of the fields is changed. This requires a DB migration script. 

  * PORT - <uint16> 1-65535 : Default 443 for REST, 8080 for Telemetry
  * jwt_valid - <uint64> 1-2^64-1 : Default: 3600 (Seconds)
  * jwt_refresh - <uint64> 1-2^64-1 : Default: 900 (Seconds)
  * enable - <boolean> - Default: true
  * client_auth - <enum> - Default: PASSWORD_JWT


### 3.2.1 CONFIG DB

  * REST_SERVER|default port - Already exists and will not change
  * REST_SERVER|default client_auth - Already exists, but format is changing from string with comma separated values to enum of valid client auth modes.
  * REST_SERVER|default jwt_valid - Already exists and will not change.
  * REST_SERVER|default jwt_refresh - Already exists and will not change.
  * REST_SERVER|default enable - Needs to be added, will be a "bool" type.

  * TELEMETRY|gnmi port - Already exists and will not change
  * TELEMETRY|gnmi client_auth - Already exists, but format is changing from string with comma separated values to enum of valid client auth modes.
  * TELEMETRY|gnmi jwt_valid - Already exists and will not change.
  * TELEMETRY|gnmi jwt_refresh - Already exists and will not change.
  * TELEMETRY|gnmi enable - Needs to be added, will be a "bool" type.

## 3.3 Switch State Service Design
N/A

### 3.6.2 CLI

This feature uses the KLISH CLI only.
  

#### 3.6.2.1 Configuration Commands
sonic(config)# management
  rest       Authentication for REST
  telemetry  Authentication for telemetry
 
sonic(config)# management rest
  client-authentication 
  enable (default)
  port
  token-refresh          Configure the seconds before JWT expiry that the token can be refreshed
  token-validity         Configure the seconds JWT token is valid for
 
sonic(config)# management rest client-authentication
  certificate               Certificate-based authentication only
  jwt-certificate           Certificate and token-based authentication
  none                      Disable all forms of authentication
  password                  Password-based authentication only
  password-certificate      Password and certificate authentication
  password-jwt              Password and token-based authentication (default)
  password-jwt-certificate  Enable all forms of authentication
 
sonic(config)# management telemetry
  client-authentication 
  enable (default)
  port
  token-refresh          Configure the seconds before JWT expiry that the token can be refreshed
  token-validity         Configure the seconds JWT token is valid for
 
sonic(config)# management telemetry client-authentication
  certificate               Certificate-based authentication only
  jwt-certificate           Certificate and token-based authentication
  none                      Disable all forms of authentication
  password                  Password-based authentication only
  password-certificate      Password and certificate authentication
  password-jwt              Password and token-based authentication (default)
  password-jwt-certificate  Enable all forms of authentication
 
sonic(config)# no management
  rest       Set rest management authentication to default (password-jwt)
  telemetry  Set rest management authentication to default (password-jwt)
 
sonic(config)# no management rest
  client-authentication
  enable (Shuts down external REST server port)
  port (Reset listening port to default)
  token-refresh   Set REST token refresh to default (900s)
  token-validity  Set REST token valid to default (3600s)
  <cr>
 
sonic(config)# no management telemetry
  client-authentication
  enable (Shuts down gNMI server and dial-in functionality)
  port (Reset listening port to default)
  token-refresh   Set telemetry token refresh to default (900s)
  token-validity  Set telemetry token valid to default (3600s)
  <cr>
  
#### 3.6.2.2 Show Commands

sonic# show management rest

REST:
Client Auth: password-jwt
Enable: True
Port: 443
token-refresh: 900
token-validity: 3600

sonic# show management telemetry

Telemetry:
Client Auth: password-jwt
Enable: True
Port: 443
token-refresh: 900
token-validity: 3600

#### 3.6.2.3 Exec Commands
N/A

### 3.6.3 REST API Support

The sonic-mgmt-svcs YANG model will be available via the REST API as all other models are. The following URLs are provided.

#### REST Server

/restconf/data/sonic-management:sonic-management/REST_SERVER/REST_SERVER_LIST=default/port

/restconf/data/sonic-management:sonic-management/REST_SERVER/REST_SERVER_LIST=default/client_auth

/restconf/data/sonic-management:sonic-management/REST_SERVER/REST_SERVER_LIST=default/jwt_refresh

/restconf/data/sonic-management:sonic-management/REST_SERVER/REST_SERVER_LIST=default/jwt_valid

/restconf/data/sonic-management:sonic-management/REST_SERVER/REST_SERVER_LIST=default/enable


#### Telemetry Server

/restconf/data/sonic-management:sonic-management/TELEMETRY/TELEMETRY_LIST=gnmi/port

/restconf/data/sonic-management:sonic-management/TELEMETRY/TELEMETRY_LIST=gnmi/client_auth

/restconf/data/sonic-management:sonic-management/TELEMETRY/TELEMETRY_LIST=gnmi/jwt_refresh

/restconf/data/sonic-management:sonic-management/TELEMETRY/TELEMETRY_LIST=gnmi/jwt_valid

/restconf/data/sonic-management:sonic-management/TELEMETRY/TELEMETRY_LIST=gnmi/enable


### 3.6.4 gNMI Support

gNMI Will support the same paths as REST but using the gNMI path/key format, i.e:

sonic-management:sonic-management/REST_SERVER/REST_SERVER_LIST[server=default]/port

sonic-management:sonic-management/REST_SERVER/REST_SERVER_LIST[server=default]/client_auth

sonic-management:sonic-management/REST_SERVER/REST_SERVER_LIST[server=default]/jwt_refresh

sonic-management:sonic-management/REST_SERVER/REST_SERVER_LIST[server=default]/jwt_valid

sonic-management:sonic-management/REST_SERVER/REST_SERVER_LIST[server=default]/enable

sonic-management:sonic-management/TELEMETRY/TELEMETRY_LIST[server=gnmi]/port

sonic-management:sonic-management/TELEMETRY/TELEMETRY_LIST[server=gnmi]/client_auth

sonic-management:sonic-management/TELEMETRY/TELEMETRY_LIST[server=gnmi]/jwt_refresh

sonic-management:sonic-management/TELEMETRY/TELEMETRY_LIST[server=gnmi]/jwt_valid

sonic-management:sonic-management/TELEMETRY/TELEMETRY_LIST[server=gnmi]/enable



## 3.7 Warm Boot Support
N/A

## 3.8 Upgrade and Downgrade Considerations

The client_auth field will need to be migrated from the old format of a string with comma separated values to the new enum based field. A migrator function will be added to the migration script to handle this.

The existing fields are a string with comma separated values (CSV). Since these values can be in any order, and the "none" option can invalidate the other modes, this must be considered during the migration.

Below is a list of valid migrations:

- password -> PASSWORD
- password,jwt -> PASSWORD_JWT
- none -> NONE
- password,jwt,cert -> PASSWORD_JWT_CERT
- password,cert -> PASSWORD_CERT
- jwt,cert -> JWT_CERT
- cert -> CERT



## 3.9 Resource Needs

No significant increases in resources needed.

# 4 Flow Diagrams
N/A

# 5 Error Handling

The model should prevent invalid configurations from being created, such as invalid client_auth modes (like JWT alone) which is handled by using an ENUM of valid modes. For PORT field, valid ports are between 1-65535, 0 should not be allowed and should result in an error. Both jwt_refresh and jwt_valid should not allow 0 either. This is acheived by the YANG range statement. In the case an invalid value is specified on the CLI, the CLI will display the error.

In the case that the user specifies a port that is already in use, this will cause the service to not restart successfully. In this case the service should revert back to the default port.

# 6 Serviceability and Debug

N/A

# 7 Scalability
N/A

# 8 Platform

  * All Platforms

# 9 Limitations
    * This is available via the Management interface(s) only.
    * After making a configuration change to REST or gNMI, those services will be restarted and will not be available until they have full restarted.
    * Disabling REST services only shuts down the external port, the service must still run for the CLI to function. 
    * Disabling Telemetry service will shut down main telemetry process, but Dialout-Telemetry process may still run.
    * There is not currently a CLI for setting host and CA certificates. This will be addressed in a later HLD when PKI is available.

# 10 Unit Test

CLI and REST/gNMI tests will be created to test the setting/unsetting of the fields.

* Change port and verify service is available on new port.
* Change client_auth and verify new authentication method is enabled.
* Disable service and verify service is not available.

# 11 Internal Design Information
N/A

## 11.1 IS-CLI Compliance
N/A

## 11.2 Broadcom SONiC Packaging
This will be part of the sonic-mgmt-framework package.

## 11.3 Broadcom Silicon Considerations
N/A

## 11.4 Design Alternatives
N/A

## 11.5 Broadcom Release Matrix
3.1.1

