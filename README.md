
## **[IOS XE Programmability Lab](https://github.com/jeremycohoe/cisco-ios-xe-programmability-lab)**

## **Module: gNOI cert.proto certificate management API**

## Topics Covered 
Introduction to gNMI

Enabling the API

Tooling 

Use Cases and examples

## Introduction to gNOI

gNOI is the gRPC Network Operations Interface. It has been enabled on IOS XE 17.3 with support for the "certificate protocol buffer", or cert.proto for short. The gNOI cert.proto is the certificate management API - it enables programmatic operations for working with TLS/SSL/crypto CLI and certificiates. Essentially it enables network operators to more effeciently load crypto into the network device via an API.

gNOI is part of the gNMI API interface. The gNMI Model Driven Programmatic Interface is part of the **Device Configuration and Device Monitoring** ecosystem within Cisco IOS XE, shown below:

![](imgs/iosxelifecycle.png)

A review of the IOS XE Programmability and Telemetry interfaces of **gNMI, NETCONF, RESTCONF, and gRPC** is below, specifically the **YANG** data models:

![](imgs/apioverviewyang.png)

The **Google Remote Procedure Call (g) Network Management Interface (NMI)**, or **gNMI** for short, is a specification of RPC's and behaviours for managing the state on network devices. It is built on the open source gRPC framework and uses the Protobuf IDL (protocol buffers interactive data language). 

Details of Protocol Buffers is available at from Google Developers at [https://developers.google.com/protocol-buffers/docs/overview](https://developers.google.com/protocol-buffers/docs/overview) while the specification for gNMI itself is available on Github/Openconfig at [https://github.com/openconfig/gnmi](https://github.com/openconfig/gnmi) and the actual gnmi.proto file is defined at [https://github.com/openconfig/gnmi/blob/master/proto/gnmi/gnmi.proto](https://github.com/openconfig/gnmi/blob/master/proto/gnmi/gnmi.proto) - These resources can be referred if needed however for the purpose of this lab is not necessary to have a deeper understanding of these concepts.

![](imgs/gnmi_intro.png)

Similar to the NETCONF and RESTCONF programmatic interfaces, gNMI can be used for a variety of operations including retrieving operational and runtime details using the GET operations, as well as making configuration changes using the SET operation. The SUBSCRIBE operation supports Model Driven Telemetry, or streaming telemetry, to be enabled from this interface as well.

![](imgs/api_operations.png)

All of the programmatic interfaces (NETCONF, RESTCONF, gNMI, and gRPC) share the same set of YANG data models. An example is to review the interface configurations including interface descriptions which can be completed by using **any** of the programmatic interfaces using the same YANG data model: **Cisco-IOS-XE-interface-oper.YANG**. 

![](imgs/api_comparison.png)

## Enabling the gNMI Interface

The gNMI interface has already been enabled as part of the ZTP / day 0 process. The following CLI's are used to enable gNMI in secure mode with a self-signed certificate:

```
gnxi
gnxi secure-init
gnxi secure-server
gnxi secure-port 9339
```


### Step 1

### Generate certificates

Using the gen_certs.sh script from the Cisco Innovation Edge github at [https://raw.githubusercontent.com/cisco-ie/cisco-gnmi-python/master/scripts/gen_certs.sh](https://raw.githubusercontent.com/cisco-ie/cisco-gnmi-python/master/scripts/gen_certs.sh) we can easily generate certificates. This script follows the steps that are outlined in the above IOS XE 16.12 Programmability Configuration Guide link.

To generate the certificates lets follow these steps:

```
cd ; cd gnmi_ssl
bash gen_certs.sh
bash gen_certs.sh c9300 10.1.1.5 Cisco12345
```

The gen_certs.sh command above will generate the required certificates for loading into IOS XE for use with the secure gNMI API.

### Step 2

### Load Certificates

The gnoi_cert tooling is available from https://github.com/google/gnxi/tree/master/gnoi_cert and has already been installed. If needing to reinstall or install this in your own lab, the commands to install the gnoi_cert tooling would be similar to the following:

```
go get github.com/google/gnxi/gnoi_cert
go install github.com/google/gnxi/gnoi_cert
```

Loading the certificates can be done using the gnoi_cert tooling as shown below. 

The name of the crypto trustpoint is specified in the "cert-id" field which is set to **gnxi-cert**

Copy the command to provision the certificates:

**cd ~/gnmi_ssl/certs/ ; /home/auto/gnoi_cert -target_addr c9300:9339 -op provision -target_name c9300 -alsologtostderr -organization "jcohoe org" -ip_address 10.1.1.5 -time_out=10s -min_key_size=2048 -cert_id gnxi-cert -state BC -country CA -ca ./rootCA.pem -key ./rootCA.key**

You will see a log message like "Install Successfull"

If the gRPC-TLS module was previously completed then this step can be skipped - the certificate was already loaded as part of the previous module. Contiune on to validation as this was not covered in the gRPC-TLS section.



### Step 3

### Review certificate installation


Review details of the trustpoint using the **show crypto pki trustpoints** CLI's

```
show crypto pki trustpoints  | i Trustpoint
show crypto pki trustpoints
```

![](imgs/show_tp.png)


## Step 4

### Validating the gNMI interface state

gnoi_cert was used to install the 'gnxi-cert' for use with the gNMI API interface. Check the status of the gNMI with the following CLI:

**show gnxi state detail**

The ouptut should look similar. Note the **Secure trustpoint** is listed as **gnxi-cert** and that **GNMI** is in **State: Provisioned**

![](./imgs/showgnxistate.png)

This concludes the gNOI cert.proto section as the certifcate has been installed into the device's truststore and is available for use with gNMI.


## Conclusion

In this module the gNMI YANG Model Driven Programmatic interface (API) has been configured and enabled in both secure and non-secure modes. The YANGSuite and gNMI_cli tools have been used to interact with the gNMI API interface using the GUI and CLI based tooling to perform basic GET operations.







