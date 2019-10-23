# Authentication and Role-Based Access Control
# High Level Design Document
#### Rev 0.1

# Table of Contents
- [Revision History](#revision-history)
- [About this Manual](#about-this-manual)
- [Scope](#scope)
- [Definitions/Abbreviations](#definitions-abbreviations)
    + [Table 1: Abbreviations](#table-1--abbreviations)
- [1 Feature Overview](#1-feature-overview)
  * [1.1 Requirements](#11-requirements)
    + [1.1.1 Functional Requirements](#111-functional-requirements)
      - [1.1.1.1 NBI Authentication](#1111-nbi-authentication)
      - [1.1.1.2 CLI Authentication to REST server](#1112-cli-authentication-to-rest-server)
      - [1.1.1.3 Translib Enforcement of RBAC](#1113-translib-enforcement-of-rbac)
      - [1.1.1.4 Linux Groups](#1114-linux-groups)
    + [1.1.2 Configuration and Management Requirements](#112-configuration-and-management-requirements)
    + [1.1.3 Scalability Requirements](#113-scalability-requirements)
      - [1.1.3.1 REST Server](#1131-rest-server)
      - [1.1.3.2 gNMI Server](#1132-gnmi-server)
      - [1.1.3.3 Translib](#1133-translib)
    + [1.1.4 Warm Boot Requirements](#114-warm-boot-requirements)
  * [1.2 Design Overview](#12-design-overview)
    + [1.2.1 Basic Approach](#121-basic-approach)
    + [1.2.2 Container](#122-container)
    + [1.2.3 SAI Overview](#123-sai-overview)
- [2 Functionality](#2-functionality)
  * [2.1 Target Deployment Use Cases](#21-target-deployment-use-cases)
  * [2.2 Functional Description](#22-functional-description)
- [3 Design](#3-design)
  * [3.1 Overview](#31-overview)
  * [3.2 DB Changes](#32-db-changes)
    + [3.2.1 CONFIG DB](#321-config-db)
    + [3.2.2 APP DB](#322-app-db)
    + [3.2.3 STATE DB](#323-state-db)
    + [3.2.4 ASIC DB](#324-asic-db)
    + [3.2.5 COUNTER DB](#325-counter-db)
  * [3.3 Switch State Service Design](#33-switch-state-service-design)
    + [3.3.1 Orchestration Agent](#331-orchestration-agent)
    + [3.3.2 Other Process](#332-other-process)
  * [3.4 SyncD](#34-syncd)
  * [3.5 SAI](#35-sai)
  * [3.6 User Interface](#36-user-interface)
    + [3.6.1 Data Models](#361-data-models)
    + [3.6.2 CLI](#362-cli)
      - [3.6.2.1 Configuration Commands](#3621-configuration-commands)
        * [3.6.2.1.1 Authentication Methods](#36211-authentication-methods)
        * [3.6.2.1.2 User management](#36212-user-management)
      - [3.6.2.2 Show Commands](#3622-show-commands)
      - [3.6.2.3 Debug Commands](#3623-debug-commands)
      - [3.6.2.4 IS-CLI Compliance](#3624-is-cli-compliance)
    + [3.6.3 REST API Support](#363-rest-api-support)
- [4 Flow Diagrams](#4-flow-diagrams)
- [5 Error Handling](#5-error-handling)
  * [5.1 REST Server](#51-rest-server)
  * [5.2 gNMI server](#52-gnmi-server)
  * [5.3 CLI](#53-cli)
  * [5.4 Translib](#54-translib)
- [6 Serviceability and Debug](#6-serviceability-and-debug)
- [7 Warm Boot Support](#7-warm-boot-support)
- [8 Scalability](#8-scalability)
- [9 Unit Test](#9-unit-test)
- [10 Internal Design Information](#10-internal-design-information)

# Revision History
| Rev |     Date    |       Author       | Change Description                |
|:---:|:-----------:|:------------------:|-----------------------------------|
| 0.1 | 10/22/2019  |   Jeff Yin      | Initial version                   |

# About this Manual
This document provides a high-level design approach for authentication and RBAC in the SONiC Management Framework.
For authentication, this document describes how the CLI and programmatic interfaces (REST, gNMI) -- collectively referred to in this document as the northbound interfaces (NBIs) -- will authenticate users and the supported credentials and methods.
For authorization, this document describes a centralized authorization approach to be implemented in the Translib component of the SONiC Management Framework.

# Scope
This document covers the interfaces and mechanisms by which NBIs will authenticate users who wish to access and configure the SONiC system via the Management Framework. It will also cover RBAC enforcement.
This document will NOT extensively cover (or assumes the pre-existence of):
- Implementation of remote authentication and authorization (RADIUS, TACACS+, etc.)
- Local user management: CLIs and APIs for creating local users on the system (TBD)
- Public Key Infrastructure management: X.509v3 certificate installation, deletion, trust store management, etc.


# Definitions/Abbreviations

### Table 1: Abbreviations
| **Term**                 | **Meaning**                         |
|:--------------------------|:-------------------------------------|
| RBAC                      | Role-Based Access Control                   |
| AAA                       | Authentication, Authorization, Accounting |
| CLI   | Command-Line Interface |
| REST  | REpresentational State Transfer |
| gNMI  | gRPC Network Management Interface |
| NBI    | North-bound Interfaces (CLI, REST, gNMI) |



# 1 Feature Overview

## 1.1 Requirements

- The SONiC Management Framework must support authenticated access for the various supported northbound interfaces (NBIs): CLI, REST, and gNMI. Since the CLI works with the REST server, it must also be authenticated with the REST server.
- The NBIs must pass along the username info to Translib, so that Translib can enforce role-based access (RBAC).
- For RBAC, Linux Groups will facilitate authorization. Initially, two roles will be supported: read/write and read-only. Additionally, remotely-authenticated users who map to a defined role will be authenticated as a static global user on the system. These limitations should be revisited at a later date.




### 1.1.1 Functional Requirements

#### 1.1.1.1 NBI Authentication
A variety of authentication methods must be supported:
* **CLI** authentication is handled via the same mechanisms supported by SSH
  - Password-based Authentication
  - Public-key Authentication
* **REST** authentication
  - Password-based Authentication
  - Token-based authentication
  - Certificate-based Authentication
* **gNMI** Authentication
  - Password-based Authentication
  - Token-based authentication
  - Certificate-based Authentication
(Question from Jeff: When a user logs into the switch via SSH, will they be "dropped into" the KLISH shell? Can I make that assumption in this HLD?)

#### 1.1.1.2 CLI Authentication to REST server
Given that the CLI module works by issuing REST transactions to the Management Framework's REST server, the CLI module must also be able to authenticate with the REST server when REST authentication is enabled. This can be accomplished via several approaches:
1. **UNIX Socket** -- Since a user accessing the CLI has already been authenticated, the CLI module can communicate with the REST server via a UNIX socket. This UNIX socket is accessed as `root`, will not enforce any authentication methods, and is solely intended for use between CLI and REST server, so care must be taken to ensure that it is not remotely accessible by any indirect means. Additionally, the username of the CLI user must be shared between the CLI and the REST server, so that the REST server can further relay this information to Translib for RBAC enforcement. Note that this approach is still being researched, and (at first glance) a significant portion of custom code will need to be implemented.
2. **Self-Signed Certificates** -- As a user authenticates with SSH and is given access to the CLI shell, a script runs to generate a self-signed certificate associated with the user. Alternatively, the user's self-signed certificate may be generated when the user is "created" on the system. The user's information is embedded in the Subject field of the certificate, and then the certificate is installed in a static location (i.e., trust store). The CLI authenticates to the REST server with the same certificate, which is accepted because the REST server trusts all certificates installed in the static location. This approach is not as performant as the UNIX socket approach, but has the advantage of allowing the REST server to treat all incoming connections the same way, regardless of the client being a REST endpoint or the CLI module.

#### 1.1.1.3 Translib Enforcement of RBAC
REST and gNMI server modules must ensure that the username is passed to Translib so that it can enforce RBAC for a given user. Translib will be the centralized point of authorization.

Since Translib is the main authority on authorized operations, this means that the NBIs cannot render to the user what they are and are not allowed to do. The CLI, therefore, renders the entire command tree to a user, even the commands they are not authorized to execute.

(TODO/BRCM: feel free to elaborate more on the mechanisms of Translib and RBAC enforcement)

#### 1.1.1.4 Linux Groups
RBAC will be facilitated via Linux Groups:
- Local users
  - Local user with `Operator` role is added into `docker` group
  - Local user with `Admin` role is added into `admin` group and is a `sudoer`.
- Remote users
  - Remote user with `Operator` role mapped to a global `remote-user` user who is part of `docker` group
  - Remote user with `Admin` role mapped to a global `remote-user-su` user who is part of `admin` group and is a `sudoer`
  - In the future, this "global user" approach will be revisited so that remote users are authenticated with their own username so that their activities may be properly audited.


### 1.1.2 Configuration and Management Requirements

Configuration and Management of authentication/RBAC will initially be limited to toggling authentication methods for the REST and gNMI NBIs.

No UI will be initially developed for user management. Initially, users must be managed via Linux tools like `useradd`, `usermod`, `passwd`, etc.



### 1.1.3 Scalability Requirements
Adding authentication to NBIs will result in some performance overhead, especially when doing operations involving asymmetric cryptography. Care should be taken to leverage performance-enhancing features of a protocol wherever possible.

#### 1.1.3.1 REST Server
- Persistent HTTP connections can be used to preserve TCP sessions, thereby avoiding handshake overhead.
- TLS session resumption can be used to preserve the TLS session layer, thereby avoiding TLS handshake overhead and repeated authentication operations (which can involve expensive asymmetric cryptographic operations)
- Token-based authentication can be used to preserve sessions for users who have already authenticated with password-based authentication, so that they do not need to constantly re-use their passwords.
(TODO/DELL: What's the mechanism for this? JWT?)

#### 1.1.3.2 gNMI Server
- TLS session resumption can be used to preserve the TLS session layer, thereby avoiding TLS handshake overhead and repeated authentication operations (which can involve expensive asymmetric cryptographic operations)

#### 1.1.3.3 Translib
(TODO/BRCM: Fill this out if applicable)

### 1.1.4 Warm Boot Requirements
N/A

## 1.2 Design Overview
### 1.2.1 Basic Approach
The code will extend the existing Klish (CLI) and REST Server modules in the sonic-mgmt-framework repository. Klish will be extended to enable authentication to the REST server (depending on the ultimately chosen approach), and the REST Server will need to be extended to pass username data to the Translib.

The Translib code (also in sonic-mgmt-framework) will be extended to support RBAC via Linux Groups. It will receive username data from the REST/gNMI NBIs and perform the Group lookup for a given user.

The gNMI server, which currently exists in the sonic-telemetry repository, needs to support passing the username down to Translib. It also needs to be extended to enable the 3 authentication methods, although they may just be enabled via configuration.


### 1.2.2 Container
SONiC Management Framework, gNMI Telemetry containers

### 1.2.3 SAI Overview
N/A

# 2 Functionality
## 2.1 Target Deployment Use Cases
Enterprise networks that enforce authentication for their management interfaces.

## 2.2 Functional Description
This feature enables authentication and Role-Based Access Control (RBAC) on the REST and gNMI programmatic interfaces that are provided by the SONiC Management Framework and Telemetry containers. With respect to authentication, these programmatic interfaces will support password-based authentication, token-based authentication, and certificate-based authentication.

Since the Klish CLI in the management framework communicates with the REST server in the back-end, the solution will also be extended to support REST authentication.

RBAC will be enforced centrally in the management framework, so that users accessing the system through varying interfaces will be limited to the same, consistent set of operations and objects depending on their role. Users' roles will be mapped using Linux Groups.

Users and their groups will be managed via Linux tools and paradigms.

# 3 Design
## 3.1 Overview
(TODO/DELL: Draw a picture)

## 3.2 DB Changes
### 3.2.1 CONFIG DB
(TODO/DELL: Define ConfigDB changes. Mainly surrounding config of authentication methods for the REST/gNMI servers)

Note: Users will not initially be stored in the ConfigDB, so they won't be portable between systems.

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
Can be reference to YANG if applicable. Also cover gNMI here.

### 3.6.2 CLI
#### 3.6.2.1 Configuration Commands
##### 3.6.2.1.1 Authentication Methods
The authentication methods for REST and gNMI servers should be configurable to toggle them on/off.

(TODO/DELL: Fill in CLIs here)

##### 3.6.2.1.2 User management
No UI will be initially developed for user management. Initially, users must be managed via Linux tools like `useradd`, `usermod`, `passwd`, etc.

#### 3.6.2.2 Show Commands
N/A

#### 3.6.2.3 Debug Commands
N/A

#### 3.6.2.4 IS-CLI Compliance
(TODO/DELL)

The following table maps SONIC CLI commands to corresponding IS-CLI commands. The compliance column identifies how the command comply to the IS-CLI syntax:

- **IS-CLI drop-in replace**  – meaning that it follows exactly the format of a pre-existing IS-CLI command.
- **IS-CLI-like**  – meaning that the exact format of the IS-CLI command could not be followed, but the command is similar to other commands for IS-CLI (e.g. IS-CLI may not offer the exact option, but the command can be positioned is a similar manner as others for the related feature).
- **SONIC** - meaning that no IS-CLI-like command could be found, so the command is derived specifically for SONIC.

|CLI Command|Compliance|IS-CLI Command (if applicable)| Link to the web site identifying the IS-CLI command (if applicable)|
|:---:|:-----------:|:------------------:|-----------------------------------|
| | | | |
| | | | |
| | | | |
| | | | |
| | | | |
| | | | |
| | | | |

**Deviations from IS-CLI:** If there is a deviation from IS-CLI, Please state the reason(s).

### 3.6.3 REST API Support
N/A -- Allowing REST authentication to be configured via the REST API itself will introduce additional test complexities and/or non-deterministic behavior.


# 4 Flow Diagrams
N/A

# 5 Error Handling
## 5.1 REST Server
The REST server should return standard HTTP errors when authentication fails or if the user tries to access a forbidden resource or perform an unauthorized activity.

## 5.2 gNMI server
The gNMI server should return standard gRPC errors when authentication fails. (Question: Does gRPC have "standard" errors for authorization failures?)

## 5.3 CLI
Authentication errors will be handled by SSH. However, the CLI must gracefully handle authorization failures from the REST server (the authorization failure would originate from Translib of course). While the CLI will render all of the available commands to a user, the user will actually only be able to execute a subset of them. This limitation is a result of the design decision to centralize RBAC in Translib. Nevertheless, the CLI must inform the user when they attempt to execute an unauthorized command.

## 5.4 Translib
(TODO/BRCM: Describe Translib RBAC error handling)
Question/ALL: What happens if a user authenticates but is not part of one of the pre-defined groups? Perhaps they should not be allowed to do anything at all?

# 6 Serviceability and Debug
All operations performed by NBIs (CLI commands, REST/gNMI operations) should be logged/audited with usernames attached to the given operation(s) performed.
Initially, users who are remotely authenticated will share a common role-specific username, so there will be a limitation here.  

# 7 Warm Boot Support
N/A

# 8 Scalability
See previous section 1.1.3: Scalability Requirements

# 9 Unit Test

### Table 3: Test Cases
| **Test Case**                 | **Description**                         |
|:--------------------------|:-------------------------------------|
| REST with password | Authenticate to REST server with username/password and perform some operations |
| REST with token | Perform subsequent operations with token, ensure username/password are not re-prompted |
| REST with certs | Install certificate to SONiC switch, authenticate to REST server with said certificate, perform some operations |
| REST authorized RBAC | Perform authorized operations as both `Admin` and `Operator` via REST |
| REST unauthrozied RBAC | Attempt unauthorized operations as both `Admin` and `Operator` via REST |
| CLI with password | SSH to the system with username/password and execute some commands |
| CLI with RSA | SSH to the system with pubkey and execute some commands |
| CLI authorized RBAC | SSH to the system and perform authorized commands |
| CLI unauthorized RBAC | SSH to the system and perform unauthorized commands |
| RBAC no-group | Create a user and assign them to a non-predefined group; make sure they can't perform any operations |
| gNMI authentication | Test the same authentication methods as REST, but for gNMI instead |
| gNMI authorization | Test the same authorization as REST, but for gNMI instead |



# 10 Internal Design Information
(TODO/ALL: Remove)
