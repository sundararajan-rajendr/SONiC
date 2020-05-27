
## Feature Name
**Thermal Policy on Broadcom Supported Platforms**

## High Level Design Document
**Rev 0.1**

## Table of Contents
 * [List of Tables](#list-of-tables)
 * [Revision](#revision)
 * [About This Manual](#about-this-manual)
 * [Scope](#scope)
 * [Definition/Abbreviation](#definitionabbreviation)
 * [Requirements Overview](#requirements-overview)
    * [Functional Requirements](#functional-requirements)
	* [Scalability Requirements](#scalability-requirements)
 * [Supported Platforms](#supported-platforms)
 * [Thermal Policy Specification](#thermal-policy-specification)
    * [Rules](#rules)
 * [Serviceability and DEBUG](#serviceability-and-debug)
    * [Syslogs](#syslogs)
    * [Debug](#debug)
    * [Debug CLIs](#debug-clis)
 * [Unit Test](#unit-test)


# List of Tables
[Table 1: Abbreviations](#table-1-abbreviations)

# Revision
| Rev |     Date    |       Author       | Change Description                |
|:---:|:-----------:|:------------------:|-----------------------------------|
| 0.1 | 05/05/2020  |  Systems Infra Team     | Initial version                   |

# About this Manual
Thermal policy for a platform dictates how the platform would react to temperature changes in its operating environment. Usually the policy is driven by the ambient temperature and operational status of the Fans. The ambient temperature is determined by the onboard temperature sensors which are accessible by CPU \(using I2C/PCI\) or BaseBoard Management Controller \(BMC\). This document gives the details of thermal policy implementation on platforms where it is managed by CPU.

# Scope
This document gives the details of platform specific thermal policy implementation, correspinding logs and various actions taken for platform safety.  


# Definition/Abbreviation
### Table 1: Abbreviations
| **Term**                 | **Meaning**                         |
|--------------------------|-------------------------------------|
| ODM                      | Original Design Manufacturer        |
| OEM                      | Original Equipment Manufacturer        |
| PDDF                      | Platform Driver Development Framework         |
| PSU                      | Power Supply Unit |
| I2C                      | Inter-integrated Circuit communication protocol |
| SysFS                    | Virtual File System provided by the Linux Kernel |
| BMC                      | Baseboard Management Controller |


## 1 Requirements Overview
ODM needs to provide a monitoring service which would read the temperature from various temperature sensors, determines the fan speed or other actions using the thermal policy, and then implements those actions in runtime. This monitoring service is dependent on the platform APIs provided by the device object classes. This dependency is not only limited to reading temperature sensors, but also managing fans and overall system. The device object classes are either provided by ODM themselves or by PDDF. 

### 1.1	Functional Requirements
Functional requirements include capabilities of the monitoring service via platform APIs to 
 - Read the temperature from temperature sensors
 - Read the fan speed or duty cycle
 - Manage fan duty cycle
 - Manage the system with respect to thermal policy actions e.g. reset or shutdown the system

### 1.3 Scalability Requirements
NA

## 2 Supported Platforms

In Buzznik+ release, thermal policy is supported on Accton platform AS4630-54PE. Following is the platform configuration,

**_AS4630-54PE_**

| **Platform**                | **Number of Fans**          | **Number of temperature Sensors**      | **Sensor Type**    |
|-----------------------------|-----------------------------|----------------------------------------|--------------------|
| (Accton TD3-X5) AS4630-54PE |           3                 | 3 (Onboard-0x48, CPU-0x4b and FAN-0x4a)|   LM77 and LM75    |

## 3 Thermal Policy Specification
Thermal monitoring service polls for temperature of various temp sensors and fan status. The required action is decided based on the thermal policy rules.

**_AS4630-54PE_**

Target Fan duty cycle is decided by summation of the temperature values of the three temp sensors mentioned above. If one or more fan fails, remaining fans would operate on 100% duty cycle. Maximum speed for FAN1 and FAN2 is 14400 and for FAN3 is 13600 RPM.

### 3.1 Rules
**_AS4630-54PE_**

|      **Rule (temperature values are in celcius)**     |  **Thermal Level**|          **Action**            |  **FAN speed**  |
|-------------------------------------------------------|-------------------|--------------------------------|-----------------|
| LM77(0x48)+LM75(0x4B)+LM75(0x4A)  <  140              |      -            |   set fan duty cycle to 50%    |    6300+-10%    |
| LM77(0x48)+LM75(0x4B)+LM75(0x4A)  >  140              |      -            |   set fan duty cycle to 62%    |    8400+-10%    |
| LM77(0x48)+LM75(0x4B)+LM75(0x4A)  >  150              |      -            |   set fan duty cycle to 75%    |    10500+-10%   |
| LM77(0x48)+LM75(0x4B)+LM75(0x4A)  >  160              |     High          |   set fan duty cycle to 87%    |    12600+-10%   |
| if any fan fails                                      |      -            |   set fan duty cycle to 100%   |    14000+-10%   |
| LM77(0x48) >= 70                                      |     Critical      |   System resets                |       -         | 



## 4 Serviceability and DEBUG
### 4.1 Syslogs
Following syslogs are pertaining to thermal policy.

 - When the High Temp Level is crossed, a warning syslog is sent
 - When the temperature goes below the High Temp Level, a clearing warning syslog is sent
 - When the temperature reaches Critical Temp Level, a critical syslog is sent

### Examples:
```
Mar 14 13:01:29.534867 sonic WARNING [***] Alarm for temperature high is detected
Mar 14 13:01:49.534867 sonic WARNING [***] Alarm for temperature high is cleared
Dec 22 20:41:47.809449 sonic CRIT Alarm for temperature critical is detected

```
### 4.2 Debug logs
**_AS4630-54PE_**

 - Debug logs can be found at /usr/local/bin/accton_as4630_54pe_pddf_monitor.log

 - Upon crossing of critical threshold, a message is printed on the console denoting the DUT reset. 
 ```
 Alarm-Critical for temperature critical is detected, reset DUT
 ```

### 4.3 Debug CLIs

Following debug CLIs can be used to get the state of the system.
 - To get the status of fans
 > pddf_fanutil status

 - To get the speed of the fans
 > pddf_fanutil getspeed

 - To get the temperature reading of the sensors
 > pddf_thermalutil gettemp

 - To check the status of the monitoring service
 > systemctl status as4630-54pe-pddf-platform-monitor.service


## 5 Scalability
NA
## 6 Unit Test

**_AS4630-54PE_**
 - Simulate the temperature increase/decrease on the platform and observe that the fan speeds are changing
 - Check if the syslogs are thrown upon crossing the High threshold limit
 - Check if the system is reset upon crossing the Critical threhsold limit

Following commands are used for the simulation,
 1. First stop the monitoring service
 > systemctl stop as4630-54pe-pddf-platform-monitor.service

 2. Run the simulation script and observe the logs
 > /usr/local/bin/accton_as4630_54pe_pddf_monitor.py -t \<simulate-temp-decrease\> \<start value for temp1\> \<start value for temp2\> \<start value for temp3\>

where simulate-temp-decrease: 1 or 0

Polling interval for temperature monitoring is 10 seconds. If simulation is ON, a temperature of 2 degree celcius is added to each sample temperature reading every poll. If **simulate-temp-decrease** is set, temp will increase till High threhsold and then start decreaseing.

Examples:
 > /usr/local/bin/accton_as4630_54pe_pddf_monitor.py -t 0 40 40 36   <<< To simulate the temp increase all the way upto critical threshold
 
 > /usr/local/bin/accton_as4630_54pe_pddf_monitor.py -t 1 40 40 38   <<< To simulate the increase and then decrease of temperature

