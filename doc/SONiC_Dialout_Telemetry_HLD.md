# Dial-out Telemetry HLD

#### Rev 0.1

[TOC]


# About This Manual

This document is a general overview of the Dial-out feautre defined here https://github.com/openconfig/reference/issues/42

# 1 Introduction and Scope

Dial-out telemetry allows for peristent streaming telemetry configurations to be stored on the target. These configurations are activiate automatically if the target reboots or telemetry service is restarted. The format of the streaming telemetry messages to the collector are the same as those in the gNMI dial-in telemetry, namely a series of SubscribeResponse prtobuf messages with a list of updates.

The storage model of these persistent subscriptions is the [openconfig-telemetry](https://github.com/openconfig/public/blob/master/release/models/telemetry/openconfig-telemetry.yang) model. This YANG model is configurable with CLI as well as gNMI and REST. The stored subscriptions will be read by the telemetry process on startup and gNMI subscriptions will be configured for them. The telemetry process will also subscribe for updates for configuration changes to this model and will update the gNMI subscriptions as needed.

# 2 Feature Requirements

## 2.1 Functional Requirements
1. Telemetry process will open gRPC(tcp) connection to collector(s) configured for each subscription and maintain the connection until the subscription is removed.
2. If the other end closes the connection, or the connection was never successfully established, the telemetry process will periodically retry to establish the connection indefinitely.
3. The telemetry process will support encrypted and un-encrypted connections.
4. The dial-out mode will support the same subscription options as the dial-in mode.


## 2.2 Configuration and Management Requirements
1. Allow configuration of openconfig-telemetry model via all NBIs
2. Telemetry process will respond to updated configuration automatically.
3. Telemetry process will load configuration on startup.

## 2.3 Scalability Requirements
1. The telemetry process will support ten subscriptions.

## 2.4 Warm Boot Requirements
The telemetry container will be restarted and thus the telemetry connections will be dropped until the container is back up. Telemetry updates will be missed during this time.

# 3 Feature Description

## 3.1 Target Deployment use cases

- To simplify collector design by persisting subscriptions on target so that collector does not have to maintain state.
- No need to monitor and reconfigure targets when they reboot.

## 3.2 Functional Description

The role of dial-out telemetry is the same as dial-in, but the connection is originated at the target instead of the collector. The protocol used for both is the same except that the in the dial-out case, the target will begin by calling the Publish RPC on the collector and sending SubscribeResponse messages to the collector without the need for the collector to first send a Subscribe message. Also, the collector may optionally respond with a PublishResponse message to alter the subscription. The Publish RPC and PublishResponse message are the only new additions for dial-out.

The configuration will store the list of subscription paths representing YANG model paths, collector addresses and ports, encoding, protocol, QoS and source IP. The telemetry process will a