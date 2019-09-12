# synology-nagios-plugin
A simple Nagios Plugin which checks the health of a Synology NAS

It supports checking the following items:
* System status
* Power status
* Fan status
* Disks status
* RAID (Volumes) status
* DSM version and update status
* System temperature
* UPS information (maybe)

## Based on
This plugin is a modified version of a plugin by deegan199. It can be found [here.](https://exchange.nagios.org/directory/Plugins/Network-and-Systems-Management/Others/Synology-status/details)
this plugin is an improvement of the plugin of  https://github.com/corben2/check_snmp_synology 

### Changes From Original
This correct this type of errors : 

./check_snmp_synology: line 227: [: : integer expression expected
./check_snmp_synology: line 288: [: : integer expression expected
./check_snmp_synology: line 288: [: : integer expression expected
./check_snmp_synology: line 288: [: : integer expression expected
./check_snmp_synology: line 288: [: : integer expression expected
./check_snmp_synology: line 288: [: : integer expression expected
./check_snmp_synology: line 288: [: : integer expression expected
./check_snmp_synology: line 288: [: : integer expression expected
./check_snmp_synology: line 288: [: : integer expression expected
./check_snmp_synology: line 288: [: : integer expression expected
./check_snmp_synology: line 288: [: : integer expression expected
./check_snmp_synology: line 288: [: : integer expression expected
./check_snmp_synology: line 288: [: : integer expression expected

I divide the request of snmpget to allow more HDD and the original limit ( didn't work with less 10 disk for me )
I also add optimise request of snmpwalk and correct the notification message of the plugins.

If You want divide more snmpget because you have a lot of HDD. 

Edit this : 

    if [ "$checkType" = "disk" ] ; then
        nbDisk=$(echo "$tmpRequestDisk" | wc -l)
        for i in `seq 1 $nbDisk`;
        do
            if [ $i -lt 5 ] ; then
                OID_disk="$OID_disk $OID_diskID.$(($i-1)) $OID_diskModel.$(($i-1)) $OID_diskStatus.$(($i-1)) $OID_diskTemp.$(($i-1)) "
            elif ([ $i -gt 4 ] && [ $i -lt 10 ]) then
                OID_disk2="$OID_disk2 $OID_diskID.$(($i-1)) $OID_diskModel.$(($i-1)) $OID_diskStatus.$(($i-1)) $OID_diskTemp.$(($i-1)) "
            elif ([ $i -gt 9 ] && [ $i -lt 15 ]) then
                OID_disk3="$OID_disk3 $OID_diskID.$(($i-1)) $OID_diskModel.$(($i-1)) $OID_diskStatus.$(($i-1)) $OID_diskTemp.$(($i-1)) "
            elif ([ $i -gt 14 ] && [ $i -lt 20 ]) then
                OID_disk4="$OID_disk4 $OID_diskID.$(($i-1)) $OID_diskModel.$(($i-1)) $OID_diskStatus.$(($i-1)) $OID_diskTemp.$(($i-1)) "
	    else
		OID_disk5="$OID_disk5 $OID_diskID.$(($i-1)) $OID_diskModel.$(($i-1)) $OID_diskStatus.$(($i-1)) $OID_diskTemp.$(($i-1)) "
            fi
        done
    fi
	
	if [ "$OID_disk2" != "" ]; then
        syno2=`$SNMPGET $SNMPArgs $hostname $OID_disk2 2> /dev/null`
		if [ "$OID_disk3" != "" ]; then
			syno3=`$SNMPGET $SNMPArgs $hostname $OID_disk3 2> /dev/null`
			syno=$(echo "$syno";echo "$syno3";)
			if [ "$OID_disk4" != "" ]; then
				syno4=`$SNMPGET $SNMPArgs $hostname $OID_disk4 2> /dev/null`
				syno=$(echo "$syno";echo "$syno4";)
				if [ "$OID_disk5" != "" ]; then
        	        syno5=`$SNMPGET $SNMPArgs $hostname $OID_disk5 2> /dev/null`
	                syno=$(echo "$syno";echo "$syno5";)
			fi	fi
		fi
    fi


This version supports volume usage on DSM 6.2. Also, it checks readings by type,
rather then checking everything at once. This allows for splitting checks into
different services in Nagios. Look below for more information. As a part of this
change, the verbose option was removed and it is now always verbose. Also, the
UPS option was removed and added as a check type instead (but it hasn't been tested with a UPS).


## Requirements
snmpwalk and snmpget need to be installed. Also, the SNMP agent on the NAS has to be activated.

## Usage
```
usage: ./check_snmp_synology [OPTIONS] -u [user] -p [pass] -h [hostname]
options:
            -u [snmp username]   	Username for SNMPv3
            -p [snmp password]   	Password for SNMPv3

            -2 [community name]	  	Use SNMPv2 (no need user/password) & define community name (ex: public)

            -h [hostname or IP](:port)	Hostname or IP. You can also define a different port

            -W [warning temp]		Warning temperature (for disks & synology) (default 50)
            -C [critical temp]		Critical temperature (for disks & synology) (default 60)

            -w [warning %]		Warning storage usage percentage (default 80)
            -c [critical %]		Critical storage usage percentage (default 95)

            -t [check type]	        The type of check to perform, must be one of the following: system version temperature power fan disk raid ups

            -i   			Ignore DSM updates

examples:	./check_snmp_synology -u admin -p 1234 -h nas.intranet -t temperature
	     	./check_snmp_synology -u admin -p 1234 -h nas.intranet -v -t system
		./check_snmp_synology -2 public -h nas.intranet -t power
		./check_snmp_synology -2 public -h nas.intranet:10161 -t ups
```

## Nagios Configuration Examples
### Command Definition
```
define command{
        command_name    synology_check
        command_line    /usr/lib64/nagios/plugins/check_snmp_synology -2 public -h $HOSTADDRESS$ -t $ARG1$
}
```

### Service Definitions
```
define service {
	use	                   generic-service
	hostgroup_name		   synology
	service_description	   System
	check_command		   synology_check!system
	notifications_enabled      1
        check_interval             5
}

define service {
	use	                   generic-service
	hostgroup_name		   synology
	service_description	   Version
	check_command		   synology_check!version
	notifications_enabled      1
        check_interval             5
}

define service {
	use	                   generic-service
	hostgroup_name		   synology
	service_description	   Power
	check_command		   synology_check!power
	notifications_enabled      1
        check_interval             5
}

define service {
	use	                   generic-service
	hostgroup_name		   synology
	service_description	   Fans
	check_command		   synology_check!fan
	notifications_enabled      1
        check_interval             5
}

define service {
	use	                   generic-service
	hostgroup_name		   synology
	service_description	   Disks
	check_command		   synology_check!disk
	notifications_enabled      1
        check_interval             5
}

define service {
	use	                   generic-service
	hostgroup_name		   synology
	service_description	   RAID
	check_command		   synology_check!raid
	notifications_enabled      1
        check_interval             5
}

define service {
        use	                   generic-service
        hostgroup_name		   synology
        service_description	   UPS
        check_command		   synology_check!ups
        notifications_enabled      1
        check_interval             5
}
```
