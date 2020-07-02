
## **[IOS XE Programmability Lab](https://github.com/jeremycohoe/cisco-ios-xe-programmability-lab)**

## **Module: gNMI**

## Topics Covered 
Introduction to gNMI

Enabling the API

Tooling 

Use Cases and examples

## Introduction to gNMI

The gNMI Model Driven Programmatic Interface is part of the **Device Configuration and Device Monitoring** ecosystem within Cisco IOS XE, shown below:

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

## Enabling the GNMI Interface

The gNMI API has 2 mode of operating: secure and insecure.  Insecure mode can be used for initial bootstrapping of the secure server - the self signed certificate can be used to install the correct certificates in a day0 workflow. While it is simple to enable the insecure server, this lab will focus on the secure mode as this is what is recommended for use in production envrionments.

#### gNMI Prerequisite: NETCONF and AAA
The **NETCONF** interface being enabled is a prerequisite of the gNMI and other YANG based interfaces. The check the status of NETCONF run the **show netconf-yang status** CLI and ensure it is showing enabled. Start or re-start NETCONF if required by sending the **no netconf-yang** or **netconf-yang** configure commands

```
C9300#show netconf-yang status
netconf-yang: enabled
netconf-yang ssh port: 830
netconf-yang candidate-datastore: disabled

C9300#conf t
C9300(config)#no netconf-yang
C9300(config)#netconf-yang
```

#### DNS Names
In the lab environment's Ubuntu server the /etc/hosts file is used to create the local DNS resolution for the lab machine, including the c9300 which is mapped to 10.1.1.5. This way the DNS name can be used to connect to the device which allows the TLS certificte's CN field (Common Name) to match and be valid. The CN for the TLS certificate is generated with the gen_certs.sh script using the hostname **c9300**

![](imgs/etc_hosts.png)

#### gNMI - Insecure server
To enable the gNMI insecure server for day0 bootstrapping, connect to the C9300 using MobaXterm and send the following CLI commands. It is not necessary to enable insecure mode for this lab as it is not used and this is for your reference only, and instead refer to the secure server section.

```
configure terminal
gnmi-yang
gnmi-yang server
gnmi-yang port 50052
```

Note: The default insecure gNMI port is 50052 and can be changed with the **gnmi-yang port** CLI and it may not appear in the **show run | i gnmi** depending if it has been set.

Use the **show gnmi-yang state detail** CLI to confirm the **gnmi server** has been enabled on port **50052**
```
C9300#show gnmi-yang state detail
Settings
========
  Server: Enabled
  Server port: 50052
  Secure server: Disabled
  Secure server port: 9339
  Secure client authentication: Disabled
  Secure trustpoint:
  Secure client trustpoint:
```

#### gNMI - secure server

The process to enable the secure API is a 4 step process where the SSL certificates are generated using OpenSSL then installed into the IOS XE trustpoint. Next the gNMI API can be enabled using the trustpoint and certificates from the previous steps, and now the API is ready for secure communication using YANGSuite, Python, Go, or any other tooling. It is important to remember that when used in secure mode the **IP address is not used as the certificate is tied to the DNS name of 'c9300'.** Secure connections to the IP address will fail, so ensure the DNS name is used when connecting.

```
Step 1. Create SSL certificates
Step 2. Install certificates into IOS XE trustpoint
Step 3. Enable secure gNMI
Step 4. Connect and vlidate with tooling + cert
```
![](imgs/secure_gnmi_quickstart.png)

The IOS XE 16.12 configuration guide has details for creating and enabling gNMI - Refer to [https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/prog/configuration/1612/b_1612_programmability_cg/grpc_network_management_interface.html#id_89031](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/prog/configuration/1612/b_1612_programmability_cg/grpc_network_management_interface.html#id_89031) if needed.

### Step 1

### Getting Started with MobaXterm

Connect to the Remote Desktop Environment and use MobaXterm to SSH into Ubuntu VM:

![](imgs/mobaxterm.png)


Using the gen_certs.sh script from the Cisco Innovation Edge github at [https://raw.githubusercontent.com/cisco-ie/cisco-gnmi-python/master/scripts/gen_certs.sh](https://raw.githubusercontent.com/cisco-ie/cisco-gnmi-python/master/scripts/gen_certs.sh) we can easily generate the certificates. This script follows the steps that are outlined in the above IOS XE 16.12 Programmability Configuration Guide link.

To generate the certificates lets follow these steps:

```
mkdir -p ~/gnmi_ssl/
cd ~/gnmi_ssl/
# rm -f *	# This will remove any old certs and files!
wget https://raw.githubusercontent.com/cisco-ie/cisco-gnmi-python/master/scripts/gen_certs.sh
bash gen_certs.sh
bash gen_certs.sh c9300 10.1.1.5 Cisco12345
```

![](imgs/gen_certs.png)

To verify the 3 required certificates that are needed to load into IOS XE the following command can be used to cat out the files to the terminal:

**Copy the 3 required certificates to the screen**   (Do not type, instead Copy/Paste for reference if needed) 

```
echo;echo "rootCA.pem:" && cat rootCA.pem ; echo;echo "device.des3.key:" && cat device.des3.key ; echo;echo "device.crt:" &&  cat device.crt
```

You will see the 3 certificate details on the screen, this is used in the next steps when creatinging the Trustpoint on IOS XE.


### Step 2

Now that the certificates are generated they need to be installed into the IOS XE device using the **crypto pki import** commands. Alternatley the [Cisco-IOS-XE-crypto-oper](https://github.com/YangModels/yang/blob/master/vendor/cisco/xe/1721/Cisco-IOS-XE-crypto-oper.yang) YANG file can be used to programmatically install the certificate via the API's

Execute the **ls** command to list the certificate files and use **cat** to read the file out to screen

![](imgs/ls_certs.png)

You will need to copy and paste the following 3 certificates, so it is recomennded to cat out each file now using the following commands:



Next install/load the certificates into the trustpoint. Connect to the switch and follow the porocedure below to load the certificate into the trustpoint:

**configure terminal**


NOTE: It may be required to remove the previous trustpoint with the following CLI

**no crypto pki trustpoint gnmitrustpoint1**

```
C9300(config)#no crypto pki trustpoint gnmitrustpoint1
% Removing an enrolled trustpoint will destroy all certificates
 received from the related Certificate Authority.

Are you sure you want to do this? [yes/no]: yes
% Be sure to ask the CA administrator to revoke your certificates.
```

Create the new trustpoint:

**crypto pki import gnmitrustpoint1 pem terminal password Cisco12345**

Copy and paste the certificates as noted below: **rootCA.pem, device.des3.key, and device.crt**. Workflow examples are at the end of this section that show the completed workflow

```
C9300# configure terminal
C9300(config)# crypto pki import gnmitrustpoint1 pem terminal password Cisco12345 
 
# Send contents of rootCA.pem, followed by newline + 'quit' + newline:
-----BEGIN CERTIFICATE-----
<snip>
-----END CERTIFICATE-----

# Send contents of device.des3.key, followed by newline + 'quit' + newline:
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
<snip>
-----END RSA PRIVATE KEY-----

# Send contents of device.crt, followed by newline + 'quit' + newline:
-----BEGIN CERTIFICATE-----
<snip>
-----END CERTIFICATE-----
```

Now that the certificate is loaded there is one more step to configure the revocation-check on the trustpoint:

```
C9300(config)# crypto pki trustpoint gnmitrustpoint1
C9300(ca-trustpoint)# revocation-check none
C9300(ca-trustpoint)# end
```

![](imgs/install_certs.gif)

Successfull installation of the trustpoint will look similar to the following:

```
C9300(config)#crypto pki import gnmitrustpoint1 pem terminal password Cisco12345
% Enter PEM-formatted CA certificate.
% End with a blank line or "quit" on a line by itself.
-----BEGIN CERTIFICATE-----
MIIC9TCCAd2gAwIBAgIJAO2qa89hrH7BMA0GCSqGSIb3DQEBCwUAMBExDzANBgNV
BAMMBnJvb3RDQTAeFw0yMDA2MjMyMTI0MTBaFw0yMzA2MjMyMTI0MTBaMBExDzAN
BgNVBAMMBnJvb3RDQTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAK05
gXrbe/zYD8EHOviUDap8/Ww59hq6NV47B3UZWOLRyPO7Pj7HElWyu882LnYo4ot+
oaJBcBTB0gcukzrZPrxQH/xmepgs4B0WaTwHjxqycwK7HpIwcHy4PF/OfmMK5ZlT
6ZtVKTyW73G6i9vzrqTi6dY0g0eeNrkclvXM98rqbH2YxRp+PfmoKLBo0+wfQKhB
jshM2uqv08mQylta0CA8410jLpbWjxJiklOUT8/XtlUsny2itEmjeB2mCECh22tY
DqLf++G/vaURiI09T5BL2a/5KW/cvXPgs9o7SRXPiHIlTjG6HoXHNQ1CHhogugYm
lT3Q3+MqENaZMq6ZhIMCAwEAAaNQME4wHQYDVR0OBBYEFDfaecuAWI7BKQfZ+9qw
wHqN71bPMB8GA1UdIwQYMBaAFDfaecuAWI7BKQfZ+9qwwHqN71bPMAwGA1UdEwQF
MAMBAf8wDQYJKoZIhvcNAQELBQADggEBAG8ntz5E6QfWspNvx18lJX+Q3hF4SFCS
V+EOyR/qyPGdxPdN4V/kdW1FNQAkgWh8CqtqDlN4FtKrP+LHwt14EnMbYDrQvEgA
4T1pmKC4+lXBrThqhU0zGTYg7gPimQFclXrhN3nCmSAnTydoFud+h6kVMnTjwAIp
WD/004kLbceOJlTbyEBOQNY5bNZHV33AVh4tEgnwHnEm8xEtQqvAniBcsy4DCfD4
ebwdkzHd5V+xHODvt01E30LdyhAr1WZ2KYcVkdwVLB4KjD73HFY9UvPzDI3ab0cd
q1RiAqukECtALuuVKzWJIVRKmD0ticd80+m14Xv64WsqMFa+vSor8m4=
-----END CERTIFICATE-----

% Enter PEM-formatted encrypted private General Purpose key.
% End with "quit" on a line by itself.
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: DES-EDE3-CBC,D9A4A365DDFBFBF6

+Qnn5cT53jekoY3rzlCjjMJGGw3cNaSMBfR0hMwB83jrxlOleRuwjvRxTdTMXOBJ
604uMalGtGjkYSJBDoelHaDiVJhZsehoWRt7L1/54T/dn85poCYsISasIQf2b7sG
1lxeZDLU6HSQUynsgVManZqPR6wqdrLWNTXp8ghR8r8jRHzmiXWEsLkAI/OU+Pg8
BHJW683WZc0lH5cje9bowRIIhXowKDZ3q77ZRI4hVaUDtEOSInbKYZHd3XsvCCqq
RAgH+Vr1JxqRZe9TMFViveEf4rnWMUDyhI03h7kugHmUdfiYefSaLxcIL4KryrN4
UoUZ9USWqscVIgXzsza17wFRYe63h+rFPRha3ul1/r+V+t1An+Ljnw4TZeHrt19s
IB4E9/TbOGnn1SONHjYD+s3bYybL0q1K4uA8w6lBaH3IxaLBYWNg1Ay6HqyG5TCg
LlaM+RNV4NgtXVd9XiXr8wzyGsvgkOAvE999ZhLUp3R70ytzN+Z5/QeYIm1z14JT
OLx/Vh9XscbmI26aQXMk+7HHpquGv597IzV3SVtIemHwwHVGpNZR+DiXOWLYt8nb
sBjOvcbKHchtMPfkk5jggUE7xo/zV+GVKff7lFsidbhAHfBImcpQoE7Bj8lv4XXE
RPZQiKpTiuUa6P934udlWp4Ic3iYcNI8s9lfYth4Znzh/nXyQ25f6hSabu6jqpHs
OnWi4GpP1Z+1R6e2DkD1td6Czlo9crtZtFYv7QlFX/ETNQKGa1LFNNRwEH1j6CMU
+4o59yv7ZSef+byKGTIM8AHIIJCgqOHO3HuE620hOZfeTKP9rCvmpIDARLNcimj2
JKDV5Cc64yikfC18dmzO74hzXN8bIUfqkagyezewbJZGTVO9vpSiQf37YpC3+wlg
5X00KbDY4//9GdFv7cZNZwzml++5e4kt1BN+aX5qqIy+JEHniCxsenixCCGcJRGu
OBvGYbsIT5ojkarU/+WvQR2yRvUGNJAZ0qoWm4pfNsc+2tZAsfnc2UakcP3EPfKb
BGpaC0SYALWeZQN59nh9bqAZXh46gbU3wuMvk2GiPm7x22yY/WVwM7fpILX3Dl8h
7rSTptsMXV9cFJSiw7IpI2z/pOC+wBnVho2DmnjXJ7wYrZLsI2/qIZ2tnY5bb/oV
oczpdGF2iyLMFkjmRWhsYuhxFZfEc+4u4dchv4h91YiPLNpyXffzjFDtDarPY6qk
8K48apdz1F8+1yxd6HwktF7neuZi4YAVIklOjRzmPPIWyGJ3rRiiiTfi6y5AmGRm
PxGjn2wSuxUvy6GKlPPOqtmNGlAMdC8cBBMkYjnmjRgJpA3BhVW+VHyrhdVqVBfu
AtqWH7O/myCG1StLYrnS+M9Zx44EzpXI0QyQrv20tnw5OpgCDL8hf7UsNCeRe+Oy
YX0yPiD9kIi/kSF4hHi1ewb49syi+uk3bHaaFvnwByadz8xeBv9OT96K5abPOA+J
OX3vecbZ0U0k6/yx4CFfVZIY25gjASz2CboHDWEpOd3ANlNUIls02KqpQUZzuUr3
f9mOpYi+CAZIAL73zKBhLW21wUpNw+GvColL79qMnzLys4oXYb9bPA==
-----END RSA PRIVATE KEY-----

quit
% Enter PEM-formatted General Purpose certificate.
% End with a blank line or "quit" on a line by itself.
-----BEGIN CERTIFICATE-----
MIICvjCCAaagAwIBAgIJAOT1FV1fb9rbMA0GCSqGSIb3DQEBCwUAMBExDzANBgNV
BAMMBnJvb3RDQTAeFw0yMDA2MjMyMTI0MTBaFw0yMzA2MjMyMTI0MTBaMBAxDjAM
BgNVBAMMBWM5MzAwMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAsVFD
GSCLz0CGJ3E6tXb/pkFq4PyXZ01iHfs64wPuNRHqs9tURgQGhgvoUB3xCiRAY9+6
bszmEKN6kT99dlg624OSGmwP0YC8LT3srzBzo/iozg9VTkuYhsBkdsaMdixbHxC1
IA8xvY19tTtAKzQeGxFycXRG3h6+o7GuTF/g74em0fr+ZA7AA2XBjiFTdXyqkWnV
n6Lm7fu3piBLv3zfuvIBr3XLdGyvD47Jh8LYotlAHEHuliLz0YdrCttl1HSPUQI8
ZQverfUhjv2HomtF+mA+bpNZ9aZInKXi9bNXXvYjEAUqdRV1xqQpNhx+dJ+71Uly
0nKnSdQceeUysXDstQIDAQABoxowGDAWBgNVHREEDzANggVjOTMwMIcECgEBBTAN
BgkqhkiG9w0BAQsFAAOCAQEAiwABMx0adfGoObHAA9RxOVIMhIT+za769gU0ARqU
g6pruZK4eqBLNpFXO+/jJ/XaC2WZi2Fh832OM9569Q+cFtMFm4hBkRhXYh5koWuE
eb0TvlnOjDx1UO8wf68bky8WLrkEFz6NxCnaMDhB0HrL9PYGCUEe6f4SmuILnnfA
wNa6oksCx1nYDQTaG54dWBQl9so2Kf6SsBFUyXdaQP24Rb6zCD+n2pU1ebYBTpzE
Fd0NcKAnLaY+OwYhsQ2OoKXz4rceUEt9g7hlevHJptJUY4CP6oK/QC7Q1795qTXO
7H3ZQbfut+YNEnIze5/GPd2TnDkTczuvn+fxs9B8XTbn9w==
-----END CERTIFICATE-----

% PEM files import succeeded.
C9300(config)#
Jun 23 21:25:16.885: %CRYPTO_ENGINE-5-KEY_ADDITION: A key named gnmitrustpoint1 has been generated or imported by pki
```

Finally we can review details of the trustpoint using the **show crypto pki trustpoints** CLI's

```
show crypto pki trustpoints  | i Trustpoint
show crypto pki trustpoints
```

![](imgs/show_tp.png)

### Step 3

Next the gnmi-yang server is configured to use the newly installed trustpoint and associated certificates, using the **gnmi-yang secure-trustpoint gnmitrustpoint1** CLI:

```
gnmi-yang
gnmi-yang secure-trustpoint gnmitrustpoint1
gnmi-yang secure-server
gnmi-yang secure-port 9339
```

Note: The default gNMI secure port is 9339 and can be change with the **gnmi-yang secure-port** CLI and it may not appear in the **show run | i gnmi** depending if it has been set.

Use the **show gnmi-yang state detail** CLI to confirm the **gnmi secure server** has been enabled on port **9339** and that the correct **Trustpoint** has been set.

```
C9300#show gnmi-yang state detail
Settings
========
  Server: Disabled
  Server port: 50052
  Secure server: Enabled
  Secure server port: 9339
  Secure client authentication: Disabled
  Secure trustpoint: gnmitrustpoint1
  Secure client trustpoint:
```

Use the **show gnmi-yang state detail** CLI to confirm the **secure gnmi server** has been enabled on port **9339**

### Step 4

Explore the tooling in the next section that can be used to now connect to the gNMI API securely using the certificates and trustpoint configuration that has been enabled.


## Tooling

### YANGSuite with the gNMI secure server on port 9339

The YANGSuite HTML5 GUI based tooling is used to visually interact with the gNMI API. Refer to the Module 3 - NETCONF + YANG for details of YANGSuite workflows. Access YANGSuite from the web browser in the pod envrionment.


The Device Profile for the C9300 in YANGSuite has already been created, and the **Edit Device** button can be selected to confirm that gNMI is enabled and with which settings. The **Capabilities** button will establish a connection and return the supported YANG data models. 


### Load TLS certificates into YangSuite

From the YANGSuite GUI in the browser navigate to **Setup > Device Profiles** then select the **C9300** and **Edit selected device**

The certificates to load into the YANGSuite tooling are the **rootCA.pem, client.crt, and client.key** - Copy these files using MobaXterm or copy/paste the certificate contents into the Notepad/Sublime editor, and then import them into the **Device Profile** for the **C9300** by selecting the "Choose File" and supplying the correct certificate.

Example of copying and pasting the 3 required files via sublime:
![](imgs/copy_rootCA.png)


When loading the certificate files into YANGSuite ensure the 3 correct certificates are selected: **rootCA.pem client.crt client.key**
![](imgs/yangsuite-load-certs.png)

Enable the "Device supports gNMI" and "Use TLS Certificate". The certificate hostname that was entered with the gen_certs.sh command needs to match the "TLS host override" field, so enter **c9300** into this field.

![](imgs/yangsuite_enable_tls.png)

Once the certificates are loaded select **Save Change** - then select **Check Selected Devices Reachability** - this confirms the gNMI secure server has been enabled correctly.


### YANGSuite workflow for GET OpenConfig Interface Vlan1

Follow the workflow below to build and run the GET RPC for the Vlan1 interface. The completed workflow example in shown below.


![](imgs/yangsuite_get_ocif_vlan1.png)

With YANGSuite the following JSON in generated based off of the YANG modeled data:

```
{
  "prefix": {"origin": "openconfig", "elem": [{"name": "interfaces"}]},
  "path": [
    {
      "origin": "openconfig",
      "elem": [{"name": "interface", "key": {"name": "Vlan1"}}]
    },
    {
      "origin": "openconfig",
      "elem": [
        {"name": "interface", "key": {"name": "Vlan1"}},
        {"name": "name"}
      ]
    }
  ],
  "encoding": "JSON_IETF",
  "action": "get_request"
}
```

The gRPC payload above in then sent to the gNMI server that replies with the following results for the interface configuration and state data:

```
{
  "notification": [
    {
      "timestamp": "1592936497525455890",
      "prefix": {"origin": "openconfig", "elem": [{"name": "interfaces"}]},
      "update": [
        {
          "path": {
            "origin": "openconfig",
            "elem": [
              {"name": "interfaces"},
              {"name": "interface", "key": {"name": "Vlan1"}}
            ]
          },
          "val": {
            "json_ietf_val": "{\"name\":\"Vlan1\",\"config\":{\"name\":\"Vlan1\",\"type\":\"iana-if-type:l3ipvlan\",\"enabled\":true},\"state\":{\"name\":\"Vlan1\",\"type\":\"iana-if-type:l3ipvlan\",\"enabled\":true,\"ifindex\":53,\"admin-status\":\"UP\",\"oper-status\":\"UP\",\"last-change\":\"1591938451589000000\",\"counters\":{\"in-octets\":\"10360\",\"in-unicast-pkts\":\"44\",\"in-broadcast-pkts\":\"0\",\"in-multicast-pkts\":\"0\",\"in-discards\":\"0\",\"in-errors\":\"0\",\"in-unknown-protos\":\"0\",\"in-fcs-errors\":\"0\",\"out-octets\":\"5925\",\"out-unicast-pkts\":\"70\",\"out-broadcast-pkts\":\"0\",\"out-multicast-pkts\":\"0\",\"out-discards\":\"0\",\"out-errors\":\"0\",\"last-clear\":\"1591938280000000000\"},\"openconfig-platform-port:hardware-port\":\"Vlan1\"}}"
          }
        }
      ]
    },
    {
      "timestamp": "1592936497551955750",
      "prefix": {"origin": "openconfig", "elem": [{"name": "interfaces"}]},
      "update": [
        {
          "path": {
            "origin": "openconfig",
            "elem": [
              {"name": "interfaces"},
              {"name": "interface", "key": {"name": "Vlan1"}},
              {"name": "name"}
            ]
          },
          "val": {"json_ietf_val": "\"Vlan1\""}
        }
      ]
    }
  ]
}
```

The payload with the interface configuration and state data is in the **val/jason_ietf_val key-value pair** as hilighted below. The YANG modeled counters and state data, including **enabled state, interface type, infex, admin-status, etc**.

```
"val": {
            "json_ietf_val": "{"name":"Vlan1","config":{"name":"Vlan1","type":"iana-if-type:l3ipvlan","enabled":true},
            "state":{"name":"Vlan1","type":"iana-if-type:l3ipvlan","enabled":true,"ifindex":53,"admin-status":"UP",
            "oper-status":"UP","last-change":"1591938451589000000","counters":{"in-octets":"10360","in-unicast-pkts":"44",
            "in-broadcast-pkts":"0","in-multicast-pkts":"0","in-discards":"0","in-errors":"0","in-unknown-protos":"0",
            "in-fcs-errors":"0","out-octets":"5925","out-unicast-pkts":"70","out-broadcast-pkts":"0","out-multicast-pkts":"0",
            "out-discards":"0","out-errors":"0","last-clear":"1591938280000000000"},"openconfig-platform-port:hardware-port":"Vlan1"}}"
```


The completed workflow will look similar to the following:

![](imgs/yangsuite_get_vlan1.gif)

### gnmi_cli with TLS and the gNMI secure-server on port 9339

The GNMI_CLI tooling is available from [https://github.com/openconfig/gnmi](https://github.com/openconfig/gnmi) and has already been installed in the lab envrionment's Ubuntu VM using the following workflow.

1. Download the go version 1.14 from https://golang.org/dl/
2. Untar the file
3. Move it to /usr/local
4. Set the GOROOT variable
5. Set the PATH variable to include the GOROOT bin
6. "Go get" gnmi_cli from Github

The steps that were required to install Go and the gnmi_cli tool are listed here for reference should you wish to install the tooling in your own lab envrionments.
```
wget https://dl.google.com/go/go1.14.4.linux-amd64.tar.gz
tar xvfz go1.14.4.linux-amd64.tar.gz
sudo mv go /usr/local/
export GOROOT=/usr/local/go
export PATH=$GOPATH/bin:$GOROOT/bin:$PATH
go get -u github.com/openconfig/gnmi/cmd/gnmi_cli
```

The gnmi_cli tool has been installed and is ready for use:

```
auto@automation:~$ gnmi_cli --help
```

![](imgs/gnmi_cli_help.png)

A GET operation to retreive the device hostname can be sent using the following **gnmi_cli** command. First change directory (cd) into **~/gnmi_ssl/certs**  then send the **gnmi_cli -address ....** command with all the options defined:

```
auto@automation:~$ cd ~/gnmi_ssl/certs/

auto@automation:~/gnmi_ssl/certs$ 

gnmi_cli -address 10.1.1.5:9339 -server_name c9300 -with_user_pass -timeout 10s -get \
-ca_crt rootCA.pem -client_crt client.crt -client_key client.key \
-proto "$(cat ~/gnmi_proto/get_hostname.txt)"
```

In this example the payload is defined with the -proto flag and contains the folowing YANG modeled data

![](imgs/yangsuite_explore_oc_system.png)

The **get_hostname.txt** proto file defines which YANG data model and elements to retreive:

```
auto@automation:~$ cat ~/gnmi_proto/get_hostname.txt
path: <
  elem: <
    name: "system"
  >
  elem: <
    name: "config"
  >
  elem: <
    name: "hostname"
  >
>
encoding: JSON_IETF
auto@automation:~$
```

The gNMI secure server replies with the resulting payload:

```
  timestamp: 1592947803699765348
  update: <
    path: <
      elem: <
        name: "system"
      >
      elem: <
        name: "config"
      >
      elem: <
        name: "hostname"
      >
    >
    val: <
      json_ietf_val: "\"C9300\""
    >
  >
>

```

The complete workflow should look similar to the following:

![](imgs/gnmi_cli_get_hostname.gif)

## Conclusion

In this module the gNMI YANG Model Driven Programmatic interface (API) has been configured and enabled in both secure and non-secure modes. The YANGSuite and gNMI_cli tools have been used to interact with the gNMI API interface using the GUI and CLI based tooling to perform basic GET operations.







