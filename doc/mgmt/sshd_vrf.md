# SSH Over user VRF
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
| Rev  |    Date    |    Author     | Change Description |
|:---:|:-----------:|:-------------:| --------------------------- |
| 0.1  | 12/20/2019 | Suresh Babu R | Initial version    |
| 0.2  | 04/29/2020 | Suresh Babu R | updated REST and KLISH    |

# About this Manual
This document provides general information about supporting SSH connection to switch via user defined VRF.
# Scope
This document describes the high level description of SSH support via user defined VRF in SONiC. 

# Definition/Abbreviation

### Table 1: Abbreviations
| **Term** | **Meaning**               |
| -------- | ------------------------- |
| SSH      | secure shell              |
| VRF      | virtual router forwarding |

# 1 Feature Overview




## 1.1 Requirements

### 1.1.1 Functional Requirements

Allow SSH connection to switch via user defined VRF

### 1.1.2 Configuration and Management Requirements
- Provide REST, KLISH and Click-based CLI commands to enable SSH server on given VRF
- Provide REST, KLISH and Click-based show command to display list of VRFs associated with sshd server

### 1.1.3 Scalability Requirements
- Support maximum of 15 VRFs 

### 1.1.4 Warm Boot Requirements

- N/A

## 1.2 Design Overview
### 1.2.1 Basic Approach
- Implement REST APIs , KLISH and Click-based CLI to allow/block SSH connection over user defined VRF.


### 1.2.2 Container
Changes are part of REST, CLI, redis-db and aclmgrd.

### 1.2.3 SAI Overview
N/A

# 2 Functionality
## 2.1 Target Deployment Use Cases
User can use this feature to allow management traffic via specific VRFs
## 2.2 Functional Description


By default SSH connection to the switch is allowed via default-vrf and mgmt-vrf(if one configured). With setting of net.ipv4.tcp_l3mdev_accept=1 flag, we allow SSH connection to the switch via both user defined vrf and mgmt-vrf also.User generally do not want to allow management traffic( SSH) via all VRFs. User might want to allow on specific VRF. So for better user control and configuration, by default we block SSH connection to the switch on all user defined VRFs. We will provide REST APIs and CLI commands to allow SSH to switch  on specific VRF.



# 3 Design
## 3.1 Overview
![Design](images/ssh_vrf_image.png)

During bootup, aclmgrd daemon will program iptables rules to block SSH (port 22) on user defined VRFs (Vrf+).  Whenever user configures specific VRF  to allow SSH connection, redis db will notify aclmgrd about this and aclmgrd will program iptables rules to allow given VRF.

## 3.2 DB Changes
Describe changes to existing DBs or any new DB being added.
### 3.2.1 CONFIG DB

A new table SSH_SERVER_VRF being added in CONFIG_DB

```
;SSH server per VRF
;Key
key = SSH_SERVER_VRF|<vrf-name>
;Attributes
port = 1*5DIGIT ; only port 22 allowed
```

```
127.0.0.1:6379[4]> keys SSH*
1) "SSH_SERVER_VRF|VrfTest"
127.0.0.1:6379[4]>
```
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
### 3.3.2 Other Process 
N/A

## 3.4 SyncD
N/A

## 3.5 SAI
N/A

## 3.6 User Interface
### 3.6.1 Data Models
Openconfig yang file openconfig-system-terminal.yang will be extended for this.
Extension can be found in file openconfig-system-ext.yang

### 3.6.2 CLI
#### 3.6.2.1 Configuration Commands
##### 3.6.2.1.1 Click-commands
```
config ssh-server vrf add <vrfName>

config ssh-server vrf del <vrfName>
```
##### 3.6.2.1.1 KLISH Commands
```
sonic(config)# [no] ssh-server vrf <vrfName>
```  

#### 3.6.2.2 Show Commands
##### 3.6.2.2.1 Click-Commands
```
root@sonic:~# show ssh-server vrfs
VRFS     Status

-------  --------
VrfTest  enable
VrfRed   enable
root@sonic:~#
```
##### 3.6.2.2.2 KLISH
```
sonic# show ssh-server vrf all

            VRF Name                          Status
-------------------------------- --------------------------------
Vrf1                             enable
Vrf2                             enable

sonic#
sonic# show ssh-server vrfs <vrfName>
sonic# show ssh-server vrf Vrf1

            VRF Name                          Status
-------------------------------- --------------------------------
Vrf1                             enable

sonic#
```
#### 3.6.2.3 Debug Commands
```
root@sonic:~# iptables -S
-P INPUT ACCEPT
-P FORWARD ACCEPT
-P OUTPUT ACCEPT
-A INPUT -i VrfRed -p tcp -m tcp --dport 22 -j ACCEPT
-A INPUT -i VrfTest -p tcp -m tcp --dport 22 -j ACCEPT
-A INPUT -i mgmt+ -p tcp -m tcp --dport 22 -j REJECT --reject-with icmp-port-unreachable
-A INPUT -i Vrf+ -p tcp -m tcp --dport 22 -j REJECT --reject-with icmp-port-unreachable
-A INPUT -s 127.0.0.1/32 -i lo -j ACCEPT
root@sonic:~#

root@sonic:~# ip6tables -S
-P INPUT ACCEPT
-P FORWARD ACCEPT
-P OUTPUT ACCEPT
-A INPUT -i VrfRed -p tcp -m tcp --dport 22 -j ACCEPT
-A INPUT -i VrfTest -p tcp -m tcp --dport 22 -j ACCEPT
-A INPUT -i mgmt+ -p tcp -m tcp --dport 22 -j REJECT --reject-with icmp6-port-unreachable
-A INPUT -i Vrf+ -p tcp -m tcp --dport 22 -j REJECT --reject-with icmp6-port-unreachable
-A INPUT -s ::1/128 -i lo -j ACCEPT
root@sonic:~#
```
#### 3.6.2.4 IS-CLI Compliance
KLISH based cli

### 3.6.3 REST API Support

RESTCONF APIs are supported for following yang model
```yang
 grouping ssh-vrf-config {
      description
       "Configuration data for ssh server vrf";
        leaf vrf-name {
            type string {
                pattern "mgmt|Vrf[a-zA-Z0-9_-]+" {
                    error-message "Invalid VRF name for SSH server";
                    error-app-tag vrf-name-ssh-invalid;
                }
            }
            description
                "VRF name";
        }
        leaf port {
            default 22;
            type uint16;
            description
                "SSH port number";
        }
    }
     augment "/oc-sys:system/oc-sys:ssh-server" {
        container ssh-server-vrfs {
          description
           "Container for ssh server vrfs";
            list ssh-server-vrf {
                key "vrf-name";
                max-elements 15;
                description
                  "list of ssh server vrf";

                leaf vrf-name {
                    type leafref {
                        path "../config/vrf-name";
                    }
                }

                container config {
                    description
                      "Configuration data for ssh server vrf";
                    uses ssh-vrf-config;
                }
                container state {
                    config false;
                    description
                      "Operational state data for ssh server vrf";
                    uses ssh-vrf-config;
                }
            }
        }
    }
```    

### 3.6.4 Service and Docker Management

N/A

# 4 Flow Diagrams
N/A

# 5 Error Handling
N/A

# 6 Serviceability and Debug
N/A

# 7 Warm Boot Support
Configuration should be persistent across reboot.

# 8 Scalability
N/A

# 9 Unit Test
1)Verify ssh vrf add configuration and iptables rules are programmed accordingly

2)Verify ssh vrf del configuration and iptables rules are removed accordingly

3)Verfiy show ssh vrf configuration.

4)Verify all REST APIs for configuration and operational data

5)Verfiy KLISH commands both configuration and show.




