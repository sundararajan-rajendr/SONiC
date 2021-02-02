# SONiC telemetry in dial-out mode

# High Level Design Document
# Table of Contents
- [List of Tables](#list-of-tables)
- [Revision](#revision)
- [About this Manual](#about-this-manual)
- [Scope](#scope)
- [Definition/Abbreviation](#definitionabbreviation)
- [Table 1: Abbreviations](#table-1-abbreviations)
- [1 Feature Overview](#1-Feature-Overview)
    - [1.1 Target Deployment Use Cases](#11-Target-Deployment-Use-Cases)
    - [1.2 Requirements](#12-Requirements)
    - [1.3 Design Overview](#13-Design-Overview)
        - [1.3.1 Basic Approach](#131-Basic-Approach)
        - [1.3.2 Services Provided](#132-Services-Provided)
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
- [4 Configuration](#4-Configuration)
  - [4.1 Manual redis Configuration](#4-Manual-redis-Configuration)
  - [4.2 Openconfig Telemetry Configuration](#4.2-Openconfig-Telemetry-Configuration)
- [5 Error Handling](#5-Error-Handling)
- [6 Testing](#6-Testing)
  - [6.1 Unit Tests](#61-Unit-Tests)
  - [6.2 Functional Tests](#62-Functional-Tests)


# List of Tables
[Table 1: Abbreviations](#table-1-Abbreviations)

# Revision
| Rev |     Date    |       Author       | Change Description                |
|:---:|:-----------:|:------------------:|-----------------------------------|
| 0.1 | <02/02/2021>|   Eric Seifert     | Update Format, add OC-Telemetry   |
| 0.0 | <08/21/2018>|   Jipan Yang       | Initial version                   |

# About this Manual
This document provides comprehensive functional and design information about the Dialout Telemetry feature implementation in SONiC described in this specification https://github.com/openconfig/reference/issues/42

# Definition/Abbreviation

### Table 1: Abbreviations
| **Term**                 | **Meaning**                         |
|--------------------------|-------------------------------------|
| protobuf                 | Protocol Buffers                    |
| TLS                      | Transport Layer Security            |
| OC                       | OpenConfig                          |
| gRPC                     | gRPC Remote Procedure Calls         |
| YANG                     | Yet Another Next Generation         |

# 1 Feature Overview

Dial-out telemetry allows for persistent streaming telemetry configurations to be stored on the target. Telemetry connections are initiated from the target to the collector. These configurations activate automatically if the target reboots or telemetry service is restarted. The format of the streaming telemetry messages to the collector are the same as those in the gNMI dial-in telemetry, namely a series of SubscribeResponse protobuf messages with a list of updates described in the gnmi protobuf specification: https://github.com/openconfig/gnmi/blob/master/proto/gnmi/gnmi.proto

The dial-out service is configured either directly in the redis DB, or via the [openconfig-telemetry](https://github.com/openconfig/public/blob/master/release/models/telemetry/openconfig-telemetry.yang) model using the management framework that translates to a sonic-telemetry model to store in the redis database. The Openconfig Telemetry YANG model is configurable with gNMI and REST interfaces.

The stored subscriptions will be read by the telemetry process on startup and gNMI subscriptions will be configured for them. The telemetry process will also subscribe for updates for configuration changes to this model and will update the gNMI subscriptions as needed.

The subscription modes supported for dial-out are periodic samples, defined by the sample interval in the persistent subscription, or when the sample interval is set to 0, subscription responses are sent out only when the field changes (if supported).

## 1.1 Target Deployment Use Cases

- To simplify collector design by persisting subscriptions on target so that collector does not have to maintain state.
- No need to monitor and reconfigure targets when they reboot.
- No need to expose a service to the outside world, reducing risk.
- Firewall/Nat service sits between network device and telemetry collectors, and the collectors cannot initiate connections.

## 1.2 Requirements

*Fill out with detailed, immutably numbered requirements from which test cases can be generated. A structured numbering scheme is used using the following sections. Some sections may be omitted according to the needs of the feature: -*

1. Telemetry process will open gRPC(TCP) connection to collector(s) configured for each subscription and maintain the connection until the subscription is removed.
2. If the collector closes the connection, or the connection was never successfully established, the telemetry process will periodically retry to establish the connection indefinitely.
3. The telemetry process will support encrypted (TLS) and un-encrypted connections.
4. The dial-out mode will support both sample and ON_CHANGE subscriptions (on supported fields)
5. Allow configuration of openconfig-telemetry model via REST and gNMI.
6. Telemetry process will respond to updated configuration automatically.
7. Telemetry process will load configuration on startup.
8. The telemetry container will recover when restarted and the telemetry connections will be dropped until the container is back up. Telemetry updates will be missed during this time.
9. Configuration of OC Telemetry will return an error on invalid configuration.
10. Dial-out telemetry will support both JSON_IETF and Protobuf encodings.
11. Dial-out will only support gRPC protocol.

## 1.3 Design Overview

### 1.3.1 Basic Approach
System telemetry in SONiC supports both dial-in mode and dial-out mode. The DB client takes care of retrieving data from SONiC redis databases, non-DB client serves data outside of redis databases and translib data client serves YANG models. gRPC dial-out client is described in this document. Dial-out telemetry server is implemented as a separate process in the telemetry container, along with the main telemetry process. The dial-out telemetry process reuses much of the code from the telemetry process such as the DB clients described above, as well as the subscription mechanism from translib (in the tranlib DB client use case). The architecture is simple, with each dial-out subscription running in it's own client (goroutine/thread) to serve that connection. Sample based subscriptions are served by a timer, while ON_CHANGE type subscriptions are served by the translib Subscribe API (for translib DB client only).
![SOFTWARE ARCHITECTURE](img/dial_in_out.png)

### 1.3.2 Services Provided
gNMIDialout service is defined for telemetry in dial-out mode. It has one streaming RPC: Publish. The message from client to collector reuses [SubscribeResponse](https://github.com/openconfig/gnmi/blob/f6185680be3b63e2b17e155f06bfc892f74fc3e7/proto/gnmi/gnmi.proto#L216) from gNMI spec, while the PublishResponse message is optional and skipped by default. The dial-out service was defined here: https://github.com/openconfig/reference/issues/42

```
// gNMIDialOut defines a service which is used by a target system (typically a
// network element) to initiate connections to a client (collector). The server
// is implemented at the collector, such that the target can initiate connections
// to the collector, based on a configured set of telemetry subscriptions.
service gNMIDialOut {
  // Publish allows the target to send telemetry updates (in the form of
  // SubscribeResponse messages, which have the same semantics as in the
  // gNMI Subscribe RPC, to a client. The client may optionally return the
  // PublishResponse message in response to the dial-out connection from the
  // target as acknowledgment to the SubscribeResponse message
  //
  // The configuration of subscriptions associated with the publish RPC may
  // be through the OpenConfig telemetry configuration and operational state
  // model:
  // https://github.com/openconfig/public/blob/master/release/models/telemetry/openconfig-telemetry.yang
  rpc Publish(stream SubscribeResponse) returns (stream PublishResponse);
}

message PublishResponse {
  int64 timestamp = 1;          // Timestamp in nanoseconds since Epoch.
  Path prefix = 2;              // Prefix used for paths in the message.
  // An alias for the path specified in the prefix field.
  // Reference: gNMI Specification Section 2.4.2
  string alias = 3;
  repeated Path path = 4;     // Paths for which the notifications have been received.
}
```

# 2 Functionality

  - Be compatible with openconfig-telemetry model, and configurable with gNMI and REST interfaces.
  - Provide periodic sample and on-change subscriptions.
  - Support same data clients (DB, non-DB & translib) and subscription paths as dial-in mode.
  - Automatically establish an resume telemetry connections as needed.

# 3 Design

## 3.1 Design Overview

The two main concerns for dial-out telemetry are configuration and operation. To configure the dial-out mode, there are two methods. The original method uses a redis DB table as described below. Also supported is the openconfig-telemetry YANG model via the management framework. When the dial-out telemetry is started, it will read both of these configuration tables and create the clients for each with their respective subscription paths and settings.

### 3.1.1 Service and Docker Management

dialout_client_cli is the program running inside SONiC system in the telemetry container to collect telemetry data based on the configuration and stream data to collectors. The service has been integrated as part of SONiC. The dialout_client_cli is started automatically in the telemetry container.

In case some development testing is wanted, it may be manually started with command "/usr/sbin/dialout_client_cli -insecure -logtostderr -v 1".

dialout_server_cli is the testing program prepared for verifying the dialout service.

Below is one example testing scenario:
* dialout_client_cli has been started on SONiC, but the collectors are not up.

```
root@sonic:/# /usr/sbin/dialout_client_cli  -insecure -logtostderr -v 1
I0212 01:13:20.625697     608 dialout_client_cli.go:43] Starting telemetry publish client
I0212 01:13:50.627810     608 dialout_client.go:239] Dialout connection for {30.57.186.214:8081} failed: HS_RDMA, cs.conTryCnt Dial to ({30.57.186.214:8081}, timeout 30s): context deadline exceeded%!(EXTRA uint64=1)
I0212 01:14:20.628098     608 dialout_client.go:239] Dialout connection for {30.57.185.39:8081} failed: HS_RDMA, cs.conTryCnt Dial to ({30.57.185.39:8081}, timeout 30s): context deadline exceeded%!(EXTRA uint64=2)
```

* start dialout_server_cli on 30.57.185.39:8081

```
root@ASW:/tmp# ./dialout_server_cli -allow_no_client_auth -logtostderr -port 8081 -insecure -v 2
I0212 09:21:19.068612   29112 dialout_server.go:65] Created Server on localhost:8081
I0212 09:21:19.068748   29112 dialout_server_cli.go:90] Starting RPC server on address: localhost:8081
```

* dialout_client_cli is able to connect to collector on  30.57.185.39:8081

```
I0212 01:21:29.414704     630 dialout_client.go:242] Dialout service connected to {30.57.185.39:8081} successfully for HS_RDMA
```

* dialout_server_cli will receive the telemetry data

```
root@ASW-A2-16-A02.NA62:/tmp# ./dialout_server_cli -allow_no_client_auth -logtostderr -port 8081 -insecure -v 2
I0212 09:21:19.068612   29112 dialout_server.go:65] Created Server on localhost:8081
I0212 09:21:19.068748   29112 dialout_server_cli.go:90] Starting RPC server on address: localhost:8081
== subscribeResponse:
update: <
  timestamp: 1518398489415679977
  prefix: <
    target: "COUNTERS_DB"
  >
  update: <
    path: <
      elem: <
        name: "COUNTERS"
      >
      elem: <
        name: "Ethernet*"
      >
      elem: <
        name: "SAI_PORT_STAT_PFC_7_RX_PKTS"
      >
    >
    val: <
      json_ietf_val: "{\"Ethernet0\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet1\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet10\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet11\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet12\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet13\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet14\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet15\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet16\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet17\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet18\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet19\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet2\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet20\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet21\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet22\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet23\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet24\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet25\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet26\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet27\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet28\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet29\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet3\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet30\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet31\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet32\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet33\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet34\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet35\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet36\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet37\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet38\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet39\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet4\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet40\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet41\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet42\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet43\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet44\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet45\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet46\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet47\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet48\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet5\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet52\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet56\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet6\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet60\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet64\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet68\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet7\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet8\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"},\"Ethernet9\":{\"SAI_PORT_STAT_PFC_7_RX_PKTS\":\"0\"}}"
    >
  >
>
```

### 3.1.2 Packet Handling

The dial-out telemetry proces will initiate the connections via gRPC protocol. The gRPC protocol uses TCP connections that can either be unencrypted or encrypted with TLS. The user configuration will determine the destination IP and ports that are used. The source-ip is configurable in the dial-out configuration as well.


# 4 Configuration
The configuration of dial-out telemetry in SONiC is implemented with reference to openconfig [telemetry yang model](https://github.com/openconfig/public/blob/master/release/models/telemetry/openconfig-telemetry.yang)

## 4.1 Manual redis Configuration
There are three categories of configuration:
* Global
  * encoding:  It may be one of `JSON_IETF`, `ASCII`, `BYTES`  and `PROTO`.  Default value is JSON_IETF.
  * src_ip: Source ip address of the connection from device, if not specified, the device management IP will be used.
  * retry_interval: When connection to collector is down, how long dialout client should wait before retry. 30 seconds by default.
  * unidirectional: Whether to make the Publish RPC one directly only, no PublishResponse is expected by default.
* DestinationGroup
  * dst_addr: Multiple IP address plus port number of the collectors may be specified. dialout client will try the next one in a DesistinationGroup if current one got disconnected due to failure.
  Number of DestinationGroups is not limited.
* Subscription
  * dst_group: The DestinationGroup to be used by this subscription.
  * path_target: The DB target for this subscription
  * paths:  The list of paths subscribed to in this instance of subscription.
  * report_type: May be one of "periodic", "stream" or "once". "periodic" is the default value
  * report_interval:  How frequent the data for all paths should be sent to collector, in millisecond, default value is "5000".

One example configuration:
```
{
  "TELEMETRY_CLIENT": {
        "Global": {
            "encoding": "JSON_IETF",
            "retry_interval": "30",
            "src_ip": "30.57.185.38",
            "unidirectional": "true"
        },
        "DestinationGroup_HS": {
            "dst_addr": "30.57.186.214:8081,30.57.185.39:8081"
        },
        "Subscription_HS_RDMA": {
            "dst_group": "HS",
            "path_target": "COUNTERS_DB",
            "paths": "COUNTERS/Ethernet*,COUNTERS_PORT_NAME_MAP",
            "report_interval": "5000",
            "report_type": "periodic"
        }
    }
}
```

## 4.2 Openconfig Telemetry Configuration
Another way to configure dial-out telemetry is using the openconfig-telemetry YANG model via the REST or gNMI interfaces. The openconfig model is shown below however, only persistent-subscriptions are supported, not dynamic-subscriptions.

Openconfig Telemetry YANG Model:
```
module: openconfig-telemetry
  +--rw telemetry-system
     +--rw sensor-groups
     |  +--rw sensor-group* [sensor-group-id]
     |     +--rw sensor-group-id    -> ../config/sensor-group-id
     |     +--rw config
     |     |  +--rw sensor-group-id?   string
     |     +--ro state
     |     |  +--ro sensor-group-id?   string
     |     +--rw sensor-paths
     |        +--rw sensor-path* [path]
     |           +--rw path      -> ../config/path
     |           +--rw config
     |           |  +--rw path?             string
     |           |  +--rw exclude-filter?   string
     |           +--ro state
     |              +--ro path?             string
     |              +--ro exclude-filter?   string
     +--rw destination-groups
     |  +--rw destination-group* [group-id]
     |     +--rw group-id        -> ../config/group-id
     |     +--rw config
     |     |  +--rw group-id?   string
     |     +--ro state
     |     |  +--ro group-id?   string
     |     +--rw destinations
     |        +--rw destination* [destination-address destination-port]
     |           +--rw destination-address    -> ../config/destination-address
     |           +--rw destination-port       -> ../config/destination-port
     |           +--rw config
     |           |  +--rw destination-address?   oc-inet:ip-address
     |           |  +--rw destination-port?      uint16
     |           +--ro state
     |              +--ro destination-address?   oc-inet:ip-address
     |              +--ro destination-port?      uint16
     +--rw subscriptions
        +--rw persistent-subscriptions
        |  +--rw persistent-subscription* [name]
        |     +--rw name                  -> ../config/name
        |     +--rw config
        |     |  +--rw name?                     string
        |     |  +--rw local-source-address?     oc-inet:ip-address
        |     |  +--rw originated-qos-marking?   oc-inet:dscp
        |     |  +--rw protocol?                 identityref
        |     |  +--rw encoding?                 identityref
        |     +--ro state
        |     |  +--ro name?                     string
        |     |  +--ro id?                       uint64
        |     |  +--ro local-source-address?     oc-inet:ip-address
        |     |  +--ro originated-qos-marking?   oc-inet:dscp
        |     |  +--ro protocol?                 identityref
        |     |  +--ro encoding?                 identityref
        |     +--rw sensor-profiles
        |     |  +--rw sensor-profile* [sensor-group]
        |     |     +--rw sensor-group    -> ../config/sensor-group
        |     |     +--rw config
        |     |     |  +--rw sensor-group?         -> ../../../../../../../sensor-groups/sensor-group/config/sensor-group-id
        |     |     |  +--rw sample-interval?      uint64
        |     |     |  +--rw heartbeat-interval?   uint64
        |     |     |  +--rw suppress-redundant?   boolean
        |     |     +--ro state
        |     |        +--ro sensor-group?         -> ../../../../../../../sensor-groups/sensor-group/config/sensor-group-id
        |     |        +--ro sample-interval?      uint64
        |     |        +--ro heartbeat-interval?   uint64
        |     |        +--ro suppress-redundant?   boolean
        |     +--rw destination-groups
        |        +--rw destination-group* [group-id]
        |           +--rw group-id    -> ../config/group-id
        |           +--rw config
        |           |  +--rw group-id?   -> ../../../../../../../destination-groups/destination-group/group-id
        |           +--ro state
        |              +--ro group-id?   -> ../../../../../../../destination-groups/destination-group/group-id
        +--rw dynamic-subscriptions
           +--ro dynamic-subscription* [id]
              +--ro id              -> ../state/id
              +--ro state
              |  +--ro id?                       uint64
              |  +--ro destination-address?      oc-inet:ip-address
              |  +--ro destination-port?         uint16
              |  +--ro sample-interval?          uint64
              |  +--ro heartbeat-interval?       uint64
              |  +--ro suppress-redundant?       boolean
              |  +--ro originated-qos-marking?   oc-inet:dscp
              |  +--ro protocol?                 identityref
              |  +--ro encoding?                 identityref
              +--ro sensor-paths
                 +--ro sensor-path* [path]
                    +--ro path     -> ../state/path
                    +--ro state
                       +--ro path?             string
                       +--ro exclude-filter?   string

```

The oenconfig model is provided by the mgmt framework transformer. The transformer however uses another model to actually store the data in the redis DB. This model called sonic-telemetry is below which directly maps to the openconfig model is below.

Sonic Telemetry Yang Model:

```
module: sonic-telemetry
  +--rw sonic-telemetry
     +--rw DIALOUT_SENSOR_GROUP
     |  +--rw DIALOUT_SENSOR_GROUP_LIST* [sensor_group_id]
     |     +--rw sensor_group_id    string
     +--rw DIALOUT_SENSOR_PATH
     |  +--rw DIALOUT_SENSOR_PATH_LIST* [sensor_group_id path]
     |     +--rw sensor_group_id    -> ../../../DIALOUT_SENSOR_GROUP/DIALOUT_SENSOR_GROUP_LIST/sensor_group_id
     |     +--rw path               string
     +--rw DIALOUT_DESTINATION_GROUP
     |  +--rw DIALOUT_DESTINATION_GROUP_LIST* [group_id]
     |     +--rw group_id    string
     +--rw DIALOUT_DESTINATION
     |  +--rw DIALOUT_DESTINATION_LIST* [group_id destination_address destination_port]
     |     +--rw group_id               -> ../../../DIALOUT_DESTINATION_GROUP/DIALOUT_DESTINATION_GROUP_LIST/group_id
     |     +--rw destination_address    inet:ip-address
     |     +--rw destination_port       uint16
     +--rw DIALOUT_PERSISTENT_SUBSCRIPTION
     |  +--rw DIALOUT_PERSISTENT_SUBSCRIPTION_LIST* [name]
     |     +--rw name                      string
     |     +--rw local_source_address?     inet:ip-address
     |     +--rw originated_qos_marking?   uint8
     |     +--rw protocol?                 enumeration
     |     +--rw encoding?                 enumeration
     +--rw DIALOUT_SUBSCR_SENSOR_PROFILE
     |  +--rw DIALOUT_SUBSCR_SENSOR_PROFILE_LIST* [name sensor_group]
     |     +--rw name                  -> ../../../DIALOUT_PERSISTENT_SUBSCRIPTION/DIALOUT_PERSISTENT_SUBSCRIPTION_LIST/name
     |     +--rw sensor_group          -> ../../../DIALOUT_SENSOR_GROUP/DIALOUT_SENSOR_GROUP_LIST/sensor_group_id
     |     +--rw sample_interval?      uint64
     |     +--rw heartbeat_interval?   uint64
     |     +--rw suppress_redundant?   boolean
     +--rw DIALOUT_SUBSCR_DESTINATION_GROUP
        +--rw DIALOUT_SUBSCR_DESTINATION_GROUP_LIST* [name group_id]
           +--rw name        -> ../../../DIALOUT_PERSISTENT_SUBSCRIPTION/DIALOUT_PERSISTENT_SUBSCRIPTION_LIST/name
           +--rw group_id    -> ../../../DIALOUT_DESTINATION_GROUP/DIALOUT_DESTINATION_GROUP_LIST/group_id
```

Example Openconfig Telemetry Configuration:

```
$ curl -s -k https://100.94.162.19/restconf/data/openconfig-telemetry:telemetry-system -u admin:sonicadmin | jq .
{
  "openconfig-telemetry:telemetry-system": {
    "destination-groups": {
      "destination-group": [
        {
          "config": {
            "group-id": "dg1"
          },
          "destinations": {
            "destination": [
              {
                "config": {
                  "destination-address": "10.20.0.2",
                  "destination-port": 1234
                },
                "destination-address": "10.20.0.2",
                "destination-port": 1234,
                "state": {
                  "destination-address": "10.20.0.2",
                  "destination-port": 1234
                }
              }
            ]
          },
          "group-id": "dg1",
          "state": {
            "group-id": "dg1"
          }
        }
      ]
    },
    "sensor-groups": {
      "sensor-group": [
        {
          "config": {
            "sensor-group-id": "sg1"
          },
          "sensor-group-id": "sg1",
          "sensor-paths": {
            "sensor-path": [
              {
                "config": {
                  "path": "/openconfig-acl:acl"
                },
                "path": "/openconfig-acl:acl",
                "state": {
                  "path": "/openconfig-acl:acl"
                }
              }
            ]
          },
          "state": {
            "sensor-group-id": "sg1"
          }
        }
      ]
    },
    "subscriptions": {
      "persistent-subscriptions": {
        "persistent-subscription": [
          {
            "config": {
              "local-source-address": "10.20.0.1",
              "name": "sub1"
            },
            "destination-groups": {
              "destination-group": [
                {
                  "config": {
                    "group-id": "dg1"
                  },
                  "group-id": "dg1",
                  "state": {
                    "group-id": "dg1"
                  }
                }
              ]
            },
            "name": "sub1",
            "sensor-profiles": {
              "sensor-profile": [
                {
                  "config": {
                    "sample-interval": "0",
                    "sensor-group": "sg1"
                  },
                  "sensor-group": "sg1",
                  "state": {
                    "sample-interval": "0",
                    "sensor-group": "sg1"
                  }
                }
              ]
            },
            "state": {
              "local-source-address": "10.20.0.1",
              "name": "sub1"
            }
          }
        ]
      }
    }
  }
}

```

# 5 Error Handling

Error handling happens in two locations: Configuration and Run-time. During configuration of the openconfig-telemetry model, translib transformer will perform validity checks on the models data and return an error if it is invalid. During run-time, if destination collectors are not reachable, dial-out process will periodically re-try to establish the connection with appropriate log messages. In the case of an invalid path, there will be a log message indicating that the path could not be retrieved.

# 6 Testing

## 6.1 Unit Tests

Testing is provided by golang unit tests in the dial_out_client_test.go file.

![Test Topology](img/dialout.png)
```
bogon:sonic-telemetry jipanyang$ go test -v ./dialout/dialout_client/
=== RUN   TestGNMIDialOutPublish
=== RUN   TestGNMIDialOutPublish/DialOut_to_first_collector_in_stream_mode_and_synced
=== RUN   TestGNMIDialOutPublish/DialOut_to_second_collector_in_stream_mode_upon_failure_of_first_collector
--- PASS: TestGNMIDialOutPublish (26.22s)
    --- PASS: TestGNMIDialOutPublish/DialOut_to_first_collector_in_stream_mode_and_synced (0.52s)
    --- PASS: TestGNMIDialOutPublish/DialOut_to_second_collector_in_stream_mode_upon_failure_of_first_collector (7.53s)
PASS
ok    github.com/jipanyang/sonic-telemetry/dialout/dialout_client 26.233s
```

## 6.2 Functional Tests

As part of functional testing in the mgmt framework, a spytest test API is being utilized that uses the dialout_server_cli as a test collector. Translib DB client is exercised using a variety of openconfig paths to verify end-to-end functionality of the dial-out telemetry.



