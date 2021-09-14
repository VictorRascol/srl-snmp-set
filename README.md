# How to use Perl and gNMIc to extend the SRLinux net-snmp module with SET commands to enable/disable interfaces

Nokia SRLinux comes with an SNMP module based on https://github.com/net-snmp/net-snmp. However, the default configuration only supports read access, and sometimes that is not enough.

For example, some people have a use case for enabling and disabling interfaces via SNMP. net-snmp has support for various extensions to do this, in this example we will be using a custom Perl module.

# Prerequisites
Enable SNMP in SRLinux:
````
enter candidate
/system snmp
community private
network-instance mgmt admin-state enable source-address [172.20.20.2]
commit stay
````

The Perl module and shell script requires the following packages to be installed:
````
curl -sL https://github.com/karimra/gnmic/raw/master/install.sh | sudo bash
sudo yum install -y epel-release
sudo yum install -y net-snmp-perl jq
````

* Copy the file snmp_write_handler.pl to /usr/share/snmp/snmp_write_handler.pl
* Copy the file gnmic-set-ifstatus.sh to /usr/local/bin/, use chmod 755 to make it executable
* Temporarily edit /etc/snmp/snmpd.conf_mgmt to include:
````
    access custom_grp "" any noauth exact sys2view rwview none      # Edit this line, note 'rwview'
    view rwview included interfaces.ifTable.ifEntry.ifAdminStatus
    perl do "/usr/share/snmp/snmp_write_handler.pl";
````
  Note that the above configuration will get overwritten upon reboot or change of SNMP configuration.
  To make these settings persistent, note that one can use 'includeDir' or 'includeFile' directives

* Restart the SNMP daemon by sending it a HUP signal, to reload the config

# Usage
The following examples assume interface lo0 has index 1073758206:

## Enable
```
snmpset -v 2c -c private 172.20.20.2 .1.3.6.1.2.1.2.2.1.7.1073758206 i 1
```
You may be able to use 'ifAdminStatus' instead of the OID '.1.3.6.1.2.1.2.2.1.7' when the corresponding SNMP MIBs are installed

## Disable
```
snmpset -v 2c -c private 172.20.20.2 .1.3.6.1.2.1.2.2.1.7.1073758206 i 2
````

## Limitations
When done manually, the configuration is not persisted across reboots

# Agent
In order to overcome the limitations of manual configuration and to make things manageable through a single configuration file, a sample Python agent is included.
This agent performs the necessary SNMP configuration as described above, when enabled:
```
A:leaf1# /system snmp                                                                                                                                                                                              
--{ + running }--[ system snmp ]--                                                                                                                                                                                 
A:leaf1# info                                                                                                                                                                                                      
    community $aes$73MtC3Nkc1zg=$hxMfoH62G7g5IgBMzT46vg==
    network-instance mgmt {
        admin-state enable
        enable-set-interface true  !!! Agent enabled
        source-address [
            172.20.20.2
        ]
    }
```

