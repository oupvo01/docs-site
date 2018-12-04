# Zowe API Mediation Layer Security

- [Introduction and requirements](#introduction-and-requirements)
    - [Transport-level Security](#transport-level-security)
    - [Authentication](#authentication)
    - [Authorization](#authorization)
    - [Types of services](#types-of-services)
    - [Transport Security Requirements](#transport-security-requirements)
    - [Authentication](#authentication-1)
    - [Trust stores and key stores](#trust-stores-and-key-stores)
- [Client Certificates](#client-certificates)
    - [Authentication to the Discovery Service](#authentication-to-the-discovery-service)
- [Certificate Management in Zowe API Mediation Layer](#certificate-management-in-zowe-api-mediation-layer)
- [Running on localhost](#running-on-localhost)
    - [How to start APIML on localhost with full HTTPS](#how-to-start-apiml-on-localhost-with-full-https)
    - [Certificate management script](#certificate-management-script)
    - [Generating own certificates for localhost](#generating-own-certificates-for-localhost)
    - [Generating certificate for a new service on localhost](#generating-certificate-for-a-new-service-on-localhost)
    - [Add a service with an existing certificate to APIML on localhost](#add-a-service-with-an-existing-certificate-to-apiml-on-localhost)
    - [Login to Discovery service on localhost](#login-to-discovery-service-on-localhost)
- [Zowe runtime on z/OS](#zowe-runtime-on-zos)
    - [Certificates for z/OS installation from the Zowe PAX file](#certificates-for-zos-installation-from-the-zowe-pax-file)
    - [Generating certificate for a new service on localhost](#generating-certificate-for-a-new-service-on-localhost-1)
    - [Add a service with an existing certificate to APIML on localhost](#add-a-service-with-an-existing-certificate-to-apiml-on-localhost-1)


## Introduction and requirements

The security of the APIML is performed on several levels and are described in following sections.

### Transport-level Security

Data needs to be secured during transport. This is achieved by using the TLS protocol for all connections to APIML services. While disabling the TLS protocol is permitted (e.g. for debugging purposes), the default mode is to have it on.

### Authentication

Authentication is a way how an entity, whether it be a user (API Client) or an application (API Service), proves its true identity.  

APIML uses two authentication methods:
- user ID and password (and authentication tokens retrieved by using the user ID and password) 
    - These requests originate from a user.
    - The user ID and password are validated by a z/OS security manager and
    a token is issued that is then used to access the API service.
- TLS client certificates
    - These certificates are for service-only requests.

In the future, we would like APIML to support client certificates to access the gateway.

### Authorization

Authorization is a method used to determine access rights of an entity.

In the APIML, the authorization is done by the z/OS security manager ([CA ACF2](https://www.ca.com/us/products/ca-acf2.html), [IBM RACF](https://www.ibm.com/support/knowledgecenter/zosbasics/com.ibm.zos.zsecurity/zsecc_042.htm), [CA Top Secret](https://www.ca.com/us/products/ca-top-secret.html)). The authentication token is used as proof of valid authentication. The authorization checks, however, are always done by the z/OS security manager.


### Types of services

- Zowe Core services:

    - Zowe APIML services:
    
        - Gateway Service (GW)
            - The access point for API clients that need to access API services
            - API Services can be accessed via the gateway by API Clients
            - Gets information about an API Service from the Discovery Service
        
        - Discovery Service (DS)
            - Collects information about API Services and provides it to the Gateway Service and other services
            - API Mediation services are also registered to the Discovery Service
        
        - API Catalog (AC)
            - Displays information about API services in a web UI
            - Gets information about an API Service from the Discovery Service

        - Authentication and Authorization Service (AAS) 
            - Provides authentication and authorization functionality to check access of users to resources on z/OS
            - Security service is not provided as an individual microservice but is included to the Gateway Service
            - For more details, see: [APIML wiki](https://github.com/gizafoundation/api-layer/wiki/Zowe-Authentication-and-Authorization-Service)

    - Non-APIML Zowe Core services (zLUX, Atlas)

        - They are like other regular API Client and Service described below

- API Clients
    - API Clients are external applications, users, or other API services that are accessing API services via the API Gateway
  
- API Services 
    - API Services are applications that want to be accessed via the API Gateway
    - They register themselves to the Discovery Service
    - API Services can always access other services via the API Gateway
    - API Services can sometimes access other services without the API Gateway (if they are installed in such a way that direct access is possible)
    - API Services can also be API Clients (when they access other services)

The following diagram illustrates basic relationships between services:
![Services Diagram](../images/api-mediation/apiml-components.svg)


### Transport Security Requirements

Servers ae required to provide HTTPS ports.

The requirements for the services are the following:

- API Client
    - Is not a server
    - Needs to trust the API Gateway
    - Has a trust store that contains certificate(s) needed to trust the API Gateway

- Gateway Service
    - Provides an HTTPS port
    - Has a key store with a server certificate
        - The certificate needs to be trusted by API Clients
        - This certificate should be trusted by web browsers because the API Gateway can be used to display web UIs
    - Has a trust store that contains certificates needed to trust API Services

- API Catalog
    - Provides an HTTPS port
    - Has a key store with a server certificate
        - The certificate needs to be trusted by the API Gateway
        - This certificate does not need to be trusted by anyone else

- Discovery Service
    - Provides an HTTPS port
    - Has a key store with a server certificate
        - The certificate needs to be trusted by API Clients
    - Has a trust store that contains certificates needed to trust API Services

- API Service
    - Provides an HTTPS port
    - Has a key store with a server and client certificate
        - The server certificate needs to be trusted by GW
        - The client certificate needs to be trusted by DS
        - The client and server certificates can be the same
        - These certificates do not need to be trusted by anyone else
    - Has a trust store that contains certificate(s) needed to trust the API Gateway and Discovery Service
  

### Authentication

- API Gateway

    - API Gateway currently does not handle authentication. Requests are sent to the API services that need to handle authentication

- API Catalog

    - API Catalog is accessed by users and it needs to be protected by a login
    - This is done via Authentication and Authorization Service

- Discovery Service

    - Discovery Service is accessed by API Services
    - This access (reading information and registration) needs to be protected by a client certificate
    - Access can be granted to users (administrators) - optional

- API Services

    - It is up to the service
    - It should be using Authentication and Authorization Service for authentication


###  Trust stores and key stores

A _key store_ is a repository of security certificates consisting of either authorization certificates or public key certificates with corresponding private keys, used in TLS encryption. A _key store_ can be stored in Java specific format (JKS) or use the standard format (PKCS12). The Zowe APIML uses PKCS12 to enable the key stores to be used
by other technologies used in Zowe (Node.js).

The APIML local CA:

- Contains local CA certificate and its private key (needs to be store securely)
- It is used to sign certificates of services
- Its certificate is trusted by API services and clients

The APIML key store:

- server certificate of GW (with PK) - can be signed by local CA or external CA
- server certificate of DS (with PK) - can be signed by local CA
- server certificate of AC (with PK) - can be signed by local CA
- used by APIML services

The APIML trust store:

- contains local CA public certificate
- contains external CA public certificate (optional)
- can contain self-signed certificates of API Services that are not signed by local or external CA
- used by APIML services

Zowe Core services:

- they can use the same key store and trust store as APIML for simpler installation and management
- or they can have individual stores for higher security

API service key store (for each service)

- contains server and client certificate signed by local CA
  
API service trust store (for each service)  

- contains local CA and external CA certificates (optional)


## Client Certificates

A client certificate is a certificate that is used for validation of the HTTPS client.

The client certificate of a Discovery Service client can be the same certificate as the server certificate of the services which the Discovery Service client.

### Authentication to the Discovery Service

The Discovery Service has two types of users that need to authenticate:

1. Administrators and developers who need to login to the homepage of the Discovery Service
   
   - These users need to provide valid user ID and password (currently pre-configured user ID and password, it will be replaced by mainframe security in https://waffle.io/zowe/api-layer/cards/5bd8be8131cd76001dcddd77)

2. Services that need to register to the Discovery Service

    - These are not users that have user ID and password but other services 
    - They authenticate using client certificate. The client certificate is the same TLS certificate that the service uses for HTTPS communication


## Certificate Management in Zowe API Mediation Layer

## Running on localhost

### How to start APIML on localhost with full HTTPS

The https://github.com/zowe/api-layer repository already contains pre-generated certificates that can be used to start APIML with HTTPS on your computer. The certificates are not trusted by your browser so can either ignore security warning or generate your own certificates and add them to the trust store of your browser or system.

The certificates are described in more detail in the https://github.com/zowe/api-layer/blob/https-local-certmgmt-%2372/keystore/README.md.


### Certificate management script

Zowe API Mediation layer provides a script that can used on Windows, Mac, Linux, and z/OS
to generate the certificate and key store for the local CA, API Mediation Layer, and services.

It is stored in [scripts/apiml_cm.sh](https://github.com/zowe/api-layer/blob/master/scripts/apiml_cm.sh).
It is a UNIX shell script that can be executed by Bash or z/OS Shell. For Windows, you need to install Bash, for example by using [cmder](http://cmder.net/).


### Generating own certificates for localhost

Use the following script in the root of the `api-layer` repository:

    scripts/apiml_cm.sh --action setup

This creates the certificates and key store for the API Mediation Layer in your current workspace.


### Generating certificate for a new service on localhost

The instructions are described at:
https://github.com/zowe/api-layer/blob/master/keystore/README.md#generating-certificate-for-a-new-service-on-localhost


### Add a service with an existing certificate to APIML on localhost

This will be documented during work on the following user story: https://waffle.io/zowe/api-layer/cards/5bd8be80283e09001babbf86


### Login to Discovery service on localhost

You need to provide a valid client certificate if you want to access Discovery Service on localhost.

The certificate is stored in the `keystore/localhost/localhost.keystore.p12` key store.

Some utilities including HTTPie requires the certificate to be in PEM format. You can find it in `keystore/localhost/localhost.pem`.

Since the Discovery Service is using HTTPS, your client has also verify the validity of its certificate. This can be done by trusting the local CA certificate which is store at `keystore/local_ca/localca.cer`.

Example how to access Discovery Service from CLI with full certificate validation:

    http --cert=keystore/localhost/localhost.pem --verify=keystore/local_ca/localca.cer -j GET https://localhost:10011/eureka/apps/


## Zowe runtime on z/OS

### Certificates for z/OS installation from the Zowe PAX file

When you install the Zowe runtime on z/OS from the PAX file following the instructions in [Installing the Zowe runtime on z/OS](https://zowe.github.io/docs-site/latest/user-guide/install-zos.html), the certificates for the APIML local CA and APIML service are automatically generated.

They are generated by the certificate management script `apiml_cm.sh` that is installed to `$ZOWE_ROOT_DIR/api-mediation/scripts/apiml_cm.sh`.    

`$ZOWE_ROOT_DIR` is the directory where you installed the Zowe runtime.

The certificates are generated to the directory `$ZOWE_ROOT_DIR/api-mediation/keystore`.

APIML key store and trust store:

  * `$ZOWE_ROOT_DIR/api-mediation/keystore/local/localhost.keystore.p12` 
    - used for the HTTPS servers
    - contains the APIML server certificate signed by the local CA and private key for the server
    
  * `$ZOWE_ROOT_DIR/api-mediation/keystore/local/localhost.truststore.p12` 
    - use to validate trust when communicating with the services that are registered to the APIML
    - contains the root certificate of the local CA (not the server certificate)
    - contains the local CA public certificate
    - can contain additional certificate to trust services that are not signed by local CA

APIML key and trust stores needs be accessible by the user ID that executes the Zowe runtime.

Local CA:

  * `$ZOWE_ROOT_DIR/api-mediation/keystoree/local_ca/localca.cer`
    - public certificate of local CA
  
  * `$ZOWE_ROOT_DIR/api-mediation/keystore/local_ca/localca.keystore.p12`
    - private key of the local CA 

Local CA key store can be accessible only by the user that is installing and managing the Zowe runtime. 

### Import the local CA certificate to your browser

The local CA certificate is not trusted outside of the API Mediation Layer by default.
You need to add it to the trust store of the REST API clients and to your browser.

The public certificate in the [PEM format](https://en.wikipedia.org/wiki/Privacy-Enhanced_Mail) is stored at `$ZOWE_ROOT_DIR/api-mediation/keystore/local_ca/localca.cer` where `$ZOWE_ROOT_DIR`  is the directory that was used for the Zowe runtime during installation.

It is stored in UTF-8 encoding so you need to transfer it as a binary file. Since this is certificate that your browser is going to trust, it is recommended to use a secure connection for transfer.

The recommended method is to use [Zowe CLI](https://github.com/zowe/zowe-cli#zowe-cli--):

    zowe zos-files download uss-file --binary $ZOWE_ROOT_DIR/api-mediation/keystore/local_ca/localca.cer

Alternatively, you can use `sftp`:

    sftp <system>
    get $ZOWE_ROOT_DIR/api-mediation/keystore/local_ca/localca.cer

Verify that the file has been transferred correctly. Open it and you should see something like:

    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----

Then you need to import the certificate to your root certificate store and trust it. 

For **Windows**, you can run the following command:

    certutil -enterprise -f -v -AddStore "Root" localca.cer 
    
You have to open the terminal as administrator. This will install the certificate to the Trusted Root Certification Authorities. 

If you're using **macOS**, you can run the following command: 

    $ sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain localca.cer 

**Firefox** uses its own certificate trust store. You can manually import your root certificate via the Firefox settings, or force Firefox to use the Windows trust store:

Create a new Javascript file firefox-windows-truststore.js at `C:\Program Files (x86)\Mozilla Firefox\defaults\pref` with the following content:

    /* Enable experimental Windows trust store support */
    pref("security.enterprise_roots.enabled", true);
    

### Generating certificate for a new service on z/OS

Follow the same steps as in https://github.com/zowe/api-layer/blob/master/keystore/README.md#generating-certificate-for-a-new-service-on-localhost.

Use the certificate management script that is stored at `$ZOWE_ROOT_DIR/api-mediation/scripts/apiml_cm.sh`.


### Add a service with an existing certificate to APIML on z/OS

This will be documented during work on the following user story: https://github.com/zowe/api-layer/issues/80


### Use an existing certificate for APIML

This will be documented during work on the following user story: 
https://github.com/zowe/api-layer/issues/91