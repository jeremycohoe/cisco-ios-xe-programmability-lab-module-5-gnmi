
## **[IOS XE Programmability Lab](https://github.com/jeremycohoe/cisco-ios-xe-programmability-lab)**

## **Module: gNOI OS.proto operating system API**

## Version: 17.6

## Topics Covered 
Introduction to gNMI

Enabling the API

Tooling 

Use Cases and examples


![](imgs/iosxelifecycle.png)

## Introduction to gNOI

gNOI is the gRPC Network Operations Interface. gNOI defines a set of gRPC-based microservices for executing operational commands on network devices. OS Install, Activate, and Verification are defined and addressed here:
https://github.com/openconfig/gnoi/blob/master/os/os.proto

The OS service provides an interface for OS installation on a Target. The Client progresses through 3 RPCs:
1) Installation - provide the Target with the OS package.
2) Activation - activate an installed OS package.
3) Verification – verify the installed and activated version


## Enabling the gNMI Interface

The gNMI interface has needs to be enabled . The following CLI's are used to enable gNMI in secure mode with a self-signed certificate:

```
gnxi
gnxi secure-init
gnxi secure-server
gnxi secure-port 9339
```

## Verify GNMI and GNOI with show CLI

Use the **show gnxi state detail** CLI to check the status of both the GNMI interface as well as the GNOI 

```
C9300#show gnxi state detail
Settings
========
  Secure server: Enabled
  Secure server port: 9339

...

  OS Image service
  ----------------
    Admin state: Enabled
    Oper status: Up
    Supported: Supported


```

## Provision gNMI

Loading the TLS certificates that are required for use with the GNMI API, this can be done using the gnoi_cert tooling as shown below. 

Copy the command to provision the certificates:

**cd ~/gnmi_ssl/certs/ ; /home/auto/gnoi_cert -target_addr c9300:9339 -op provision -target_name c9300 -alsologtostderr -organization "jcohoe org" -ip_address 10.1.1.5 -time_out=10s -min_key_size=2048 -cert_id gnxi-cert -state BC -country CA -ca ./rootCA.pem -key ./rootCA.key**

You will see a log message like "Install Successfull"


## Validating gNMI provisioned

gnoi_cert was used to install the 'gnxi-cert' for use with the gNMI API interface. Check the status of the gNMI with the following CLI:

**show gnxi state detail**

The ouptut should look similar. Note the **Secure trustpoint** is listed as **gnxi-cert** and that **GNMI** is in **State: Provisioned**

![](./imgs/showgnxistate.png)

This concludes the gNOI cert.proto section as the certifcate has been installed into the device's truststore and is available for use with gNMI.


## The gnoi_os tooling

The gnoi_os tooling is available from https://github.com/google/gnxi/tree/master/gnoi_os and has already been installed into the Linux VM. If needing to reinstall or install this in your own lab, the commands to install the gnoi_os tooling would be similar to the following:

```
go get github.com/google/gnxi/gnoi_os
go install github.com/google/gnxi/gnoi_os
```

Use the command below to **verify** the current running IOS XE version by using the **gnoi_os** verify operation:

**cd ~/gnmi_ssl/certs ; gnoi_os -insecure -target_addr 10.1.1.5:9339 -op verify -target_name c9300 -alsologtostderr -cert ./client.crt -ca ./rootCA.pem   -key ./rootCA.key**

The output should be similar to the following:

```
Running OS version: 17.06.01.0.1005.1623134733
```

Using the verify operation shows the full version of the running software. To determine the full version of IOS XE software that is to be installed next, the linux head command is used to look at the first 1024 bytes in the IOS XE.bin file and to filter for the CW_FULL variable. The IOS XE.bin file that is used will be stored in the **IMG** envrionment variable

**IMG=/tftpboot/cat9k_iosxe.BLD_V176_THROTTLE_LATEST_20210616_014014.SSA.bin**

**head -c 1024 $IMG | strings | grep -i CW_FULL**

The output should look similar to the below:
```
7CW_FULL_VERSION=$17.06.01.0.1228.1623826172..Bengaluru$
```

In this example the version number is: 17.06.01.0.1228.1623826172 

Set this version number in the VER envrionment variable for use, similar to the IMG variable used earlier:

**VER=17.06.01.0.1228.1623826172**

Now that the exact version is known it can be used as part of the Install and Activate operations below.

## Install operation
Run the **Install** operation:

**gnoi_os -insecure -target_addr 10.1.1.5:9339 -op install -print_progress -target_name c9300 -alsologtostderr -cert ./client.crt -ca ./rootCA.pem   -key ./rootCA.key -version $VER -time_out 999s -os $IMG**

On the switch console you should see log messages similar to below:

```
Started install add flash:gNOI_iosxe_17.06.01.0.1228.1623826172.bin

Completed install add PACKAGE flash:gNOI_iosxe_17.06.01.0.1228.1623826172.bin
```


## Activate operation
Run the **activate** operation once the install operation completes

**gnoi_os -insecure -target_addr 10.1.1.5:9339 -op activate -target_name c9300 -alsologtostderr -cert ./client.crt -ca ./rootCA.pem   -key ./rootCA.key -version $VER -time_out 999s -os $IMG**





## Conclusion

In this module the gNMI YANG Model Driven Programmatic interface (API) has been configured and enabled in both secure and non-secure modes. The YANGSuite and gNMI_cli tools have been used to interact with the gNMI API interface using the GUI and CLI based tooling to perform basic GET operations.







