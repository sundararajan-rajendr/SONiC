# Certificate Mangement

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
    - [3.2 DB Changes](#32-DB-Changes)
        - [3.2.1 CONFIG DB](#321-CONFIG-DB)
    - [3.3 User Interface](#33-User-Interface)
        - [3.3.1 Data Models](#331-Data-Models)
        - [3.3.2 CLI](#332-CLI)
        - [3.3.2.1 Configuration Commands](#3321-Configuration-Commands)
        - [3.3.2.2 Show Commands](#3322-Show-Commands)
        - [3.3.2.3 Exec Commands](#3323-Exec-Commands)
        - [3.3.3 REST API Support](#333-REST-API-Support)
        - [3.3.4 gNMI Support](#334-gNMI-Support)
     - [3.4 Upgrade and Downgrade Considerations](#34-Upgrade-and-Downgrade-Considerations)
- [4 Error Handling](#4-Error-Handling)
- [5 Platform](#5-Platform)
- [6 Limitations](#6-Limitations)
- [7 Unit Test](#7-Unit-Test)

# Revision
| Rev |     Date    |       Author       | Change Description                |
|:---:|:-----------:|:------------------:|-----------------------------------|
| 0.1 | <07/20/2021>|   Eric Seifert     | Initial version                   |
| 0.2 | <08/24/2021>|   Eric Seifert     | Add Functionality Section         |

# About this Manual
This document provides comprehensive functional and design information about the certificate management feature implementation in SONiC.

# Definition/Abbreviation

### Abbreviations
| **Term**                 | **Meaning**                         |
|--------------------------|-------------------------------------|
| PKI                      | Public Key Infrastructure           |
| CSR                      | Certificate Signing Request         |
| gNMI                     | gRPC Network Management Interface   |
| gNOI                     | gRPC Network Operations Interface   |
| CA                       | Certificate Authority               |
| PEM                      | Privacy Enhanced Mail               |
| CRL                      | Certificate Revocation List         |

# 1 Feature Overview

X.509 Public Key Certificates are used by REST and gNMI services currently and will be used by other services in the future. Configuring these certificates requires manually generating and placing the certificate and key files on the filesystem. Then you must configure the redis keys and restart the services. There is also the issue of upgrades where the location the certificates are placed is not preserved causing these services to break until the files locations are restored. Moreover the process of generating certificates, especially CA certificates and signing and distributing them is complex and error prone. Finally, when certificates expire or are about to expire, there is no warning or alarm for this event or any other issue with the certificates such as invalid hostnames, weak encryption, revocation etc.

The certificate management feature will introduce new YANG models and CLIs to address the above issues. It will handle certificate generation, signing, distribution and file management along with association of these certificates with a given service via a security profile. It will also ensure that the certificates are available after upgrade/downgrade and handle certificate rotation and alarms to alert on certificate issues. RPCs will be defined in the YANG models and exposed via REST and gNOI RPC interfaces to trigger certificate related actions. The switch will be able to act as a Certificate Authority (CA) for itself and other switches as well to simplify the initial configuration.

## 1.1 Target Deployment Use Cases

The use case for certificates is any service that wants secure communication to the outside. This includes but is not limited to:

  - HTTPS Web Server (REST, Swagger UI etc.)
  - gNMI Telemetry/Configuration
  - Dialout Telemetry
  - Syslog
  - RADIUS
  - SSH

Although only REST and gNMI are to be targeted initially for use with certificate management, the other services can be integrated eventually as well. 

## 1.2 Requirements

### Overview
  - The certificate management feature should allow for host and CA certificates to be installed, generated and removed as well as applied to specific applications. Certificates should also be validated and monitored for issues such as expiration and revocation. The switch should be able to sign Certificates thereby acting as a CA.


### Functionality
  - Establish a directory in the filesystem that all certificates and keys will be installed in to and that will be preserved through upgrades and downgrades.
  - Add ability to install host and CA certificates
  - Add ability to remove host and CA certificates
  - Add ability to configure CRL
  - Add ability to generate self-signed certificates as well as certificate signing requests
  - Add ability to sign CSR's by running in a CA mode
  - Add ability to associate host and CA certs with application
  - Add CLIs to configure and manage certificates
  - Add validation and monitoring of certificates

### Interfaces
 - The configuration of the certificate management YANG model will be available via the CLI, but also the REST and gNMI/gNOI interfaces on the management interface.

### Configurability
  - Install CA certificate from local file (alias location), remote location or copy and past into CLI
  - Generate self-signed certificate 
  - Generate certificate signing request (CSR)
  - Install certificate or private key from local file (alias location), remote location
  - Delete CA certificate
  - Delete certificate
  - Display certificate
  - Display trusted CAs
  - Display raw PEM Format certificate
  - Configure certificate revocation check behavior
  - Refresh CRL
  - Configure CRL download location(s)
  - Configure CRL Override
  - Display CRL
  - Create new security profile
  - Delete security profile
  - Associate a certificate, private key file and CA trust-store with a security-profile
  - Have "CA mode" where switch generates CA certificate and can then sign certificates for other switches
  - Have option to specify CA server to have your CSR signed by it

### User Interfaces  
  - The feature is managed through the SONiC Management Framework, including full support for KLISH, REST and gNMI  

### Serviceability  
  - All session events are logged
  - Alarms are raised for certificate issues
  - RPCs will return error codes and messages
  - All RPC and API calls will be logged
  - YANG models will validate data and return errors as appropriate

### Scaling
  - No known issues with scaling

### Warm Boot/ISSU
  - Warm boot will causes services to restart. On restart services will continue to use their configured certificates.

### Platforms  
  - Certificate management is supported on all SONiC platforms  

### Feature Interactions/Exclusions  
  - HTTP server certificate and CA will be configured with this feature
  - gNMI server certificate and CA will be configured with this feature
  - Other services will be integrated in the future

### Limitations  
  - Services must be restarted in for new certificates to take effect
  - If an invalid certificate is configured, services including REST and gNMI/gNOI may become inaccessible. However the CLI will continue to work.
  - Debian’s pre-installed ca-certificates will only be used by default Debian applications that have not been enabled to use the certificate management infrastructure.
  - In CA mode, the server and clients need to have consistent time set (preferable with NTP) to ensure certificates are valid when generated.

## 1.3 Design Overview
### 1.3.1 Basic Approach

The certificate management functionality will be implemented using new YANG models and accompanying KLISH CLI. The functionality of generating new certificate and key pairs, signing request, downloading and installing will be implemented in a python script that the CLI will call. The script will use the already installed openssl tools on the system for the certificate generation, signing etc. The downloading of the certificates from remote locations will be handled with calls to curl.

To monitor the certificates validity and the correct configuration of the services, new periodically called functions will be added to SONiC sysmonitor.py script. These functions will be responsible for creating alarms and events and downloading CRL lists.

### 1.3.2 YANG Model

The security-profile YANG model will describe the following structure(s) and field(s):

  - security-profile
    - profile-name
    - certificate-filename
    - CA Store (System or app specific)
    - revocation-check
    - peer-name-check
    - key-usage-check

This model will be for the CA server mode:

  - ca-mode
    - ca-mode (true|false)
    - ca-host
    - csr-list
      - name
      - source
      - type

This model will define a CA certificate trust-store:

  - trust-store
    - name
    - ca-list
      - name

This model will store information related to certificate revocation distribution points:

  - cdp-config
    - frequency
    - last-checked
    - cdp-list
      - url


To align with the gNOI [cert.proto](https://github.com/openconfig/gnoi/blob/master/cert/cert.proto), the following RPCs will be defined, but initially only available in gNOI due to limitations with REST RPCs:

**gNOI RPCs**

| **RPC Name**                   | **Description** |
| ------------------------------ | --------------- |
| Rotate                         | Rotate will replace an existing Certificate on the target by creating a new CSR request and placing the new Certificate based on the CSR on the target. If the stream is broken or any steps in the process fail the target must rollback to the original Certificate. |
| Install                        | Install will put a new Certificate on the target by creating a new CSR request and placing the new Certificate based on the CSR on the target.The new Certificate will be associated with a new Certificate Id on the target. If the target has a preexisting Certificate with the given Certificate Id, the operation should fail. If the stream is broken or any steps in the process fail the target must revert any changes in state. |
| GenerateCSR                    | When credentials are generated on the device, generates a key-pair and returns the Certificate Signing Request (CSR). The CSR has the public key, which when signed by the CA, becomes the Certificate  |
| LoadCertificate                | Loads a certificate signed by a Certificate Authority (CA). |
| LoadCertificateAuthorityBundle | Loads a bundle of CA certificates. |
| GetCertificates                | An RPC to get the certificates on the target. |
| RevokeCertificates             | An RPC to revoke specific certificates. If a certificate is not present on the target, the request should silently succeed. Revoking a certificate should render the existing certificate unusable by any endpoints. |
| CanGenerateCSR                 | An RPC to ask a target if it can generate a Certificate. |

In addition, to facilitate local generation of self-signed certificates and easier KLISH implementation these RPCs will also be defined for both REST and gNOI:

**Custom RPCs**

| **RPC Name**                   | **Description** |
| ------------------------------ | --------------- |
| crypto-ca-cert-install | This procedure is used to install an X.509 CA certificate |
| crypto-ca-cert-delete | This procedure is used to delete an X.509 CA certificate |
| crypto-host-cert-install | This procedure is used to install the X.509 host certificate |
| crypto-host-cert-delete | This procedure is used to delete the X.509 host certificate |
| crypto-cdp-refresh | Refresh CRL distribution point (CDP) |
| crypto-cert-generate | This procedure is used to create X.509 CSRs or self-signed certificates |
| crypto-cert-ca-sign | Sign CSR that was sent to us |
| crypto-cert-ca-csr-install | Send CSR to CA |
| crypto-cert-ca-csr-delete | Delete CSR |
| crypto-cert-signed-cert-get | Get/Download the signed certificate after the CA has signed it |

### 1.3.3 KLISH CLI

A new CLI will be added with the following commands:

**CLI Commands**

| **Command** | **Description** |
| ----------- | --------------- |
| crypto ca-cert install | Install CA cert from local (alias location) or remote location |
| crypto ca-cert delete | Delete CA certificate |
| crypto cert generate | Generate signing request or self-signed host certificate |
| crypto cert install | Install host certificate from local (alias location) or remote location |
| crypto cert delete | Delete host certificate |
| show crypto cert | Show installed certificates |
| show crypto ca-cert | Show installed CA certificates |
| show file cert | Show raw certificate file in PEM format |
| crypto security-profile | Create security-profile |
| crypto security-profile certificate | Associate security-profile with certificate |
| crypto ca-server mode | Enable CA server/client or disabled |
| crypto ca-server host | The CA server hostname/ip if in client mode |
| crypto ca-server list-csr | Show list of CSRs sent to us to be signed |
| crypto ca-server show-csr | Show details of CSR |
| crypto ca-server sign-csr | Sign CSR sent to us |
| crypto ca-server delete-csr | Reject and delete CSR sent to us |
| crypto trust-store <name> | Create new or access existing trust-store |
| crypto trust-store <name> add <ca-name> | Add CA cert to trust-store |


The CA server mode will automatically generate a CA certificate to be used to sign other certificates. This CA certificate will be rotated automatically.

**Note:**
Association of security-profile to application happens in the application specific CLI (i.e. telemetry, rest etc.). The CLIs will follow the format below:

`rest security-profile <profile-name>`

### 1.3.4 Validation

When the security-profile model is configured and the RPC's are called, the data and files passed will be validated and return appropriate errors if invalid configuration is applied. The following validations will be run at configuration time:

**Validations**

| **Operation** | **Condition** | **Response** |
| ------------- | ------------- | ------------ |
| Host cert install | Invalid Certificate | Return invalid certificate error |
| Host cert install | Expired Certificate | Return expired certificate error |
| Host cert install | Revoked Certificate | Return revoked certificate error |
| Host cert install | Invalid/No Password | If key file is password protected, return invalid password error |
| CA cert install | Invalid Certificate | Return invalid certificate error |
| CA cert install | Expired Certificate | Return expired certificate error |
| CA cert install | Revoked Certificate | Return revoked certificate error |
| Delete Host Cert | Certificate in use | Return certificate in use error |
| Delete CA Cert | Certificate in use | Return certificate in use error |
| CSR Create | Invalid CSR | Return invalid CSR error |
| Configure CA server | Time on server is different than local | Return time configuration error to prevent invalid certs |

In addition, checking if a certificate has been revoked must be enabled on a per-application basis (rest, gNMI etc.) in the options provided to the server tls settings.

### 1.3.5 Monitoring

The sysmonitor.py script will be enhanced to detect the following conditions, and using the event management framework, raise the appropriate alarm:

**Alarms**

| **Name** | **Type** | **Severity** | **Description** |
| -------- | -------- | ------------ | --------------- |
| Certificate Expiration | EVENT | WARNING | The host certificate is within 60 days of expiring |
| Certificate Expiration | EVENT | WARNING | The host certificate is within 30 days of expiring |
| Certificate Expiration | EVENT | WARNING | The host certificate is within 14 days of expiring |
| Certificate Expiration | EVENT | WARNING | The host certificate is within 7 days of expiring |
| Certificate Expired | ALARM | CRITICAL | The host certificate has expired |
| CA Certificate Expiration | EVENT | WARNING | The CA certificate is within 60 days of expiring |
| CA Certificate Expiration | EVENT | WARNING | The CA certificate is within 30 days of expiring |
| CA Certificate Expiration | EVENT | WARNING | The CA certificate is within 14 days of expiring |
| CA Certificate Expiration | EVENT | WARNING | The CA certificate is within 7 days of expiring |
| CA Certificate Expired | ALARM | CRITICAL | The CA certificate has expired |
| Revoked Certificate | ALARM | CRITICAL | The host certificate has been revoked |
| Revoked CA Certificate | ALARM | CRITICAL | The CA certificate has been revoked |
| CDP connection failed | EVENT | WARNING | Failed to connect and download from CDP server |


Also, the CA server mode will automatically generate a CA certificate to be used to sign other certificates. This CA certificate will be rotated automatically and so sysmonitor.py will periodically check and rotate the CA certificate as needed.

### 1.3.6 Directory Structure

The directory `/etc/sonic/cert` will be used to store certificates and will be mounted on the containers by default. The directory will be preserved during upgrade/downgrade through the use of upgrade hook scripts. All certificate copying, generation and associations will only use this directory path as the target. Under this directory will contain the following sub-directories:

  - ca-certs
  - certs
  - csr
  - crl

### 1.3.7 Application Associations

Applications associations with certificates will be done the same way as they currently are via per-application redis DB keys for certificate location and CA certificate location. This will preserve backwards compatibility and does not require changing the applications. The association will be managed by per-application CLI.

### 1.3.8 Container

No new containers are introduced for this feature. Existing Mgmt container will be updated.

### 1.3.9 SAI Overview

No new or existing SAI services are required.

# 2 Functionality

The Certificate Management Feature will implement new YANG models, security-profile ca-mode, trust-store and cdp-config. The security-profile model will be for associating certificates with applications as well as providing options for using those certificates. The ca-mode model will be for managing the CA certificates, CSRs and configuring a CA server either locally (server mode) or remotely (client mode). The models will also have RPCs defined for all certificate related actions.

A set of functions will be created in sysmonitor.py that will be used to monitor the certificates and configurations and raise appropriate alarms and events.

# 3 Design
## 3.1 Overview
*Big picture view of the actors involved - containers, processes, DBs, kernel etc. What's being added, what's being changed, how will they interact etc. A diagram is strongly encouraged.*

The models will be implemented in sonic-mgmt-common and will be used by the mgmt-framework and telemetry containers. The sysmonitor.py is in sonic-buildimage and will run on the host. The certificates themselves will be stored on the host in /etc/sonic/cert and will be mounted on all of the containers by default.

The YANG models will store their configuration in the configdb and will add new tables, for security-profiles, ca-mode, trust-store and cdp-config.

### 3.1.1 Service and Docker Management

No new containers will be added. The models will be configured via mgmt-framework and gNMI. The monitoring will run on the host as part of sysmonitor.py. No new services will be running either since the monitoring will be periodic and handled by sysmonitor.py.

## 3.2 DB Changes

### 3.2.1 CONFIG DB

The config DB will contain the new model's information.

## 3.3 User Interface

### 3.3.1 Data Models

security-profile:

    +--rw security-profile* [profile-name]
       +--rw profile-name            string
       +--rw certificate-name?       leaf-ref
       +--rw trust-store?            leaf-ref
       +--rw revocation-check?       boolean
       +--rw peer-name-check?        boolean
       +--rw key-usage-check?        boolean


    rpcs:
      +--x crypto-ca-cert-install
        +--w file-path?
        +--w name?
      +--w crypto-ca-cert-delete
        +--w name?
      +--w crypto-host-cert-install
        +--w file-path?
        +--w name?
      +--w crypto-host-cert-delete
        +--w name?
      +--w crypto-cert-generate
        +--w type?
        +--w destination-path?
        +--w parameters?


ca-mode:

    +--rw ca-mode
       +--rw ca-mode                bool
       +--rw ca-host                string
       +--rw csr-list* [name]
           +--rw name               string
           +--rw source             string
           +--rw type               enum


    rpcs:
      +--w crypto-cert-ca-sign
        +--w name?
      +--w crypto-cert-ca-csr-install
        +--w name?
        +--w csr?
      +--w crypto-cert-ca-csr-delete
        +-- name?
      +--w crypto-cert-signed-cert-get
        +-- name?

trust-store:

    +--rw trust-store
       +--rw name                   string
       +--rw ca-list* [name]
           +--rw certificate-name?  leaf-ref

cdp-config:

    +--rw cdp-config
       +--rw frequency              integer
       +--ro last-checked           integer
       +--rw cdp-list* [url]
           +rw url?                 string


    rpcs:
      +--x crypto-cdp-refresh
        +--w url

### 3.3.2 CLI

#### 3.3.2.1 Configuration Commands

#### Configure CRL frequency

    crypto x509 crl frequency <days>

*Parameters*
|**Name**|**Description**|
| ------ | ------------- |
| frequency | How often CDP will be checked for updates in number of days. (Default 7) |

*Note: sysmonitor.py will perform cdp updates based on desired frequency*

#### Refresh CRL

*Force CDP refresh*

    crypto cert refresh crl

#### Configure CRL download location

    [no] revocation crl identifier

*Parameters*
|**Name**|**Description**|
| ------ | ------------- |
| identifier | URL path to crl list |

#### Create new or access existing trust-store

    crypto trust-store <name>

*Parameters*
|**Name**|**Description**|
| ------ | ------------- |
| name | name of trust-store |

#### Add/Remove CA to trust-store

*Sub command of crypto trust-store*

    [no] ca-cert <name>

*Parameters*
|**Name**|**Description**|
| ------ | ------------- |
| ca-cert | name of existing CA cert to add to trust-store |

#### Create new or access existing security-profile

    security-profile <name>

*Parameters*
|**Name**|**Description**|
| ------ | ------------- |
| name | name of security-profile |

#### Associate host certificate to security-profile

*Sub command of security-profile*

    certificate <name>

*Parameters*
|**Name**|**Description**|
| ------ | ------------- |
| name | name of existing host cert to associate with security-profile |

#### Associate trust-store to security-profile

    [no] trust-store <name>

*Parameters*
|**Name**|**Description**|
| ------ | ------------- |
| name | name of existing trust-store to associate with security-profile. If unset, the system trust-store will be used. |

#### 3.3.2.2 Show Commands

#### Show host certificate(s)

    show crypto cert <name|all>

*Parameters*
|**Name**|**Description**|
| ------ | ------------- |
| name | name of certificate or all will show all host certificates |

#### Show CA cert(s)

    show crypto ca-cert <name|all>

*Parameters*
|**Name**|**Description**|
| ------ | ------------- |
| name | name of certificate or all will show all CA certificates |

#### Raw PEM Format Certificate

    show file cert <name>

*Parameters*
|**Name**|**Description**|
| ------ | ------------- |
| name | name of certificate |

### Show CRL distribution points

    show revocation crl

#### 3.3.2.3 Exec Commands

#### Install host certificate

    crypto cert install cert-file <URI> key-file <URI> password <password>

*Parameters*
|**Name**|**Description**|
| ------ | ------------- |
| cert-file | The certificate file location URI as defined by the file mgmt feature. URL or location alias i.e. home:// |
| key-file | The private key file location URI |
| password | Optional password if the private key file is password protected |

#### Delete host certificate

    crypto cert delete <name|all>

*Parameters*
|**Name**|**Description**|
| ------ | ------------- |
| name | The certificate name or all which will delete all host certificates and keys |

#### Install CA Certificate

     crypto ca-cert install <URI>

*Parameters*
|**Name**|**Description**|
| ------ | ------------- |
| cert-file | Optional certificate file location URI as defined by the file mgmt feature. URL or location alias i.e. home://. If no URI is provided, the CLI will prompt to paste the CA certificate |

*Examples*

    sonic(config)# crypto ca-cert install <URI> 
    crypto ca-cert install home://GeoTrust_Universal_CA.crt
    Processing certificate ...
    Installed Root CA certificate
     CommonName = GeoTrust Universal CA
     IssuerName = GeoTrust Universal CA


    sonic(config)# crypto ca-cert install
    Certificate base file name : Dell_interCA1
    Paste certificate below.
    Include the -----BEGIN CERTIFICATE----- and -----END CERTIFICATE----- headers.
    Enter a blank line to abort this command.
    Certificate:
    -----BEGIN CERTIFICATE-----
    MIIFuzCCA6OgAwIBAgICEAAwDQYJKoZIhvcNAQELBQAwdzELMAkGA1UEBhMCVVMx
    EzARBgNVBAgMCkNhbGlmb3JuaWExFDASBgNVBAcMC1NhbnRhIENsYXJhMREwDwYD
    VQQKDAhEZWxsIEVNQzETMBEGA1UECwwKTmV0d29ya2luZzEVMBMGA1UEAwwMRGVs
    bF9yb290Q0ExMB4XDTE4MDcyNTE4NDkyMloXDTI4MDcyMjE4NDkyMlowYjELMAkG
    A1UEBhMCVVMxEzARBgNVBAgMCkNhbGlmb3JuaWExETAPBgNVBAoMCERlbGwgRU1D
    MRMwEQYDVQQLDApOZXR3b3JraW5nMRYwFAYDVQQDDA1EZWxsX2ludGVyQ0ExMIIC
    IjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAuEaThievPvunvcEldv1QhwLe
    mCuVLrBJ5Fx824O55z3jYWPp4elvpOu4Br9Xt7sX0VDufK3RCf7DLOp5v7n6klIi
    DkliC5e4ksJZQy5T4MbU6tXsNXlPwpWCkUPuPj2u46m6N5R5J7MN+VrMG/1tJNYA
    zh09SvqVlMilHGXM8AhKf3nHaE7COrW5IYIcJUX0foT5068oBguN2nLBQRrKwWPe
    1iXv+Oynk4jgoE+TFIGm6JAxerhTTFJE4VxqqpS2DetzuBgh1Zykc6RUFluvsDUN
    Nv/LcgRj0d9IWdPpUeHLKmEg7jElUWgOvpjDIpgp+RMDxC27StLPfQD5TC5GcOOr
    5zyRsMn3SInq599P9PX8Ohfc+IxI5aoDhNcge1Uuc2OFHJehu5aVodOuDHquAjws
    B7abxZdp+oi97IuIs5Dj3KqYFcaRmaTsUnOhF7MxkIh5jynITYOFaPR6Kr+ULvyU
    En/4Nm7YvY7ZynKF37ICknagnFKkr09/VwiIjw+DeaX8Xpbxq8DabaMbYtuNQFN7
    AZ4t/39UCS6FQgumdyh98gpwmDqtTtCeGKs5bUZNW3Uqhq9J70mBPluVyyjF28pi
    CNQlSgXqgIENc440mhPrLVdiHsg9OmkxNAJsN7o4iQrLzaLJRr4mVTBR9L/1up3s
    UJzLNSmRZtkldSF59kcCAwEAAaNmMGQwHQYDVR0OBBYEFKM5y8d2hjsFRDTCb5Bz
    H19kVVx2MB8GA1UdIwQYMBaAFHUiP76Zt/qhWx1oC+leIX2DYqzbMBIGA1UdEwEB
    /wQIMAYBAf8CAQAwDgYDVR0PAQH/BAQDAgGGMA0GCSqGSIb3DQEBCwUAA4ICAQCE
    A5u+roYvaVGTALbo628a3477On5B6kw3NLeDOyzfr5aHa0Z+NNo1UonrQgXWLXJu
    MO3p9+7HG/ShBYUU4aTKWQ+J5aYNqfrA8tc9kOE7yZDvXPOY2wB1zEtiKnmUEEEB
    PXdl8jYCfJRNe9n0KHZ6pTgn5lFRZhagldZCEnWQm7yVRYBaobS+Kjlnwds86/7z
    TNdlBEcrJaQFkfdRwyTa6FSdM3XMxMjNUD7g2wgJXOCRRTsi0U2GHmTGaZQIcARD
    LdUTAgrDUmQUPOX0oS9sdNb0gHxBKpDoaQ30IOe3YIrPuOtE8FnPFDtJp5ajYNwd
    6/uR2NaL76Gnt24P9xR6afixnkghUpO6g8ttv5rC3zhF/P64pjIjI42Kzx8Venxe
    GiPEjcG2x0o8X24+5zv5GSKdGWVtEh9+kKu2sycvNH6p9p1PMWeK/uXZ0ijYknx2
    FpY3nk/dw54Ci14Q/x+sBQQsFiqpHLKKalcLxFgXmamQ/HgahPYamHESljL3d4Dw
    44xIJHVURT2RIRHHBEK8H4uARLL6ouY71gK32IgUVw5D9PtFZPmkKS4Z0m8+VaFO
    86uf2A3B+7/8m7IkySRbi83J1USrEwwuQjQMcag9bTXVia9NOzPz02k29Ev67dSV
    YlJmoZLRJebozS40kikUc4jA2LJMNNaMJPNwihMqkA==
    -----END CERTIFICATE-----
    Processing certificate ...
    Installed CA certificate
     CommonName = Dell_interCA1
     IssuerName = Dell_rootCA1
    sonic(config)#

#### Delete CA certificate

    crypto ca-cert delete <name|all>

*Parameters*
|**Name**|**Description**|
| ------ | ------------- |
| name | The certificate name or all which will delete all CA certificates |

### 3.3.3 REST API Support

**URLs:**

    /sonic-crypto:security-profile
    /sonic-crypto:security-profile/security-profile
    /sonic-crypto:security-profile/security-profile/name
    /sonic-crypto:security-profile/security-profile/certificate-name
    /sonic-crypto:security-profile/security-profile/trust-store
    /sonic-crypto:security-profile/security-profile/peer-name-check
    /sonic-crypto:security-profile/security-profile/key-usage-check
    
    /sonic-crypto:ca-mode
    /sonic-crypto:ca-mode/ca-mode
    /sonic-crypto:ca-mode/ca-host
    /sonic-crypto:ca-mode/csr-list
    /sonic-crypto:ca-mode/csr-list/name
    /sonic-crypto:ca-mode/csr-list/source
    /sonic-crypto:ca-mode/csr-list/type

    /sonic-crypto:trust-store
    /sonic-crypto:trust-store/name
    /sonic-crypto:trust-store/ca-list
    /sonic-crypto:trust-store/ca-list/certificate-name

    /sonic-crypto:cdp-config
    /sonic-crypto:cdp-config/frequency
    /sonic-crypto:cdp-config/last-checked
    /sonic-crypto:cdp-config/cdp-list
    /sonic-crypto:cdp-config/cdp-list/url

### 3.3.4 gNMI Support

The YANG models defined above will be available to read/wrtie from gNMI as well as REST. In addition to the RPCs defined, gNOI defines a set of certificate management RPCs here: [https://github.com/openconfig/gnoi/blob/master/cert/cert.proto](https://github.com/openconfig/gnoi/blob/master/cert/cert.proto) and are described above.

## 3.4 Upgrade and Downgrade Considerations
The certificate directory /etc/sonic/cert must be preserved during upgrades and downgrades. This is acheived using upgrade hook scripts that will copy from the directory to the new partition.

# 4 Error Handling

The primary error handling will happen during configuration/RPC call time where data validation will occur to prevent invalid configuration. Secondly, since certificates can expire or be revoked, periodically the sysmonitor.py will check that the host and CA certificates are still valid and raise alarms if not. Also, events will be generated as certificates approach the invalidation time.

# 5 Platform

All sonic platforms will be supported.

# 6 Limitations

  - Currently the REST and gNMI processes need to be restarted in order to use new certificates and this can take a minute or more. For REST server in particular, this can be an issue since the sonic-cli must be exited and re-started when REST server restarts due to invalidation of the jwt token.
  - Currently no way to ensure that the time/date is the same between multiple switches which is important when generating certificates since they have start/end times. Either the API should require the client send what it thinks is current time or we should require NTP be configured.


# 7 Unit Test

  - Add certificate
  - Add security-profile
  - Add trust-store
  - Configure cdp
  - Apply security-profile to REST & telemetry server and validate it is applied correctly
  - Add CA certificate and client certificate and verify REST & telemetry server can be accessed without insecure mode
  - Use invalid client certificate with CA and verify it is rejected
  - Force CDP refresh
  - Enable CA mode and generate CSR for client and get it signed. Use this certificate for REST/gNMI requests.

