# How to connect Cisco SCCP based Ip Phones in FreeSWITCH?
https://freeswitch.org/confluence/display/FREESWITCH/mod_skinny

There is no additional dependency needed to build mod_skinny. You just need to edit modules.conf (uncomment endpoints/mod_skinny) and run:

make mod_skinny

Also uncomment <load module="mod_skinny"/> in ~/freeswitch/conf/auto_configs/modules.conf.xml so that FreeSWITCH actually loads the module at start time.

 # Configuration

    192.168.0.2 is the IP of the TFTP server 
    192.168.0.3 is the IP of the FreeSWITCH server (with mod_skinny)
    SEP001120AABBCC is the phone id (based on the MAC address)
    
#     DHCP

First, the phone gets its IP address and other network configs through DHCP.

The important setting is the tftp server option
option 150 ip 192.168.0.2 (Cisco Router,L3)

# TFTP
 

There are plenty of TFTP server implementations. In the root of the tftp directory (usually /srv/tftp), put the following files
XMLDefault.cnf.xml
SEP001120AABBCC.cnf.xml (SEP001120AABBCC is the phone id based on the MAC address)

# FreeSWITCH

The configuration is loaded from autolad_configs as usual. The default is:
conf/autoload_configs/skinny.conf.xml
<configuration name="skinny.conf" description="Skinny Profiles">
  <profiles>
    <X-PRE-PROCESS cmd="include" data="../skinny_profiles/*.xml"/>
  </profiles>
</configuration> 

#  Device Configuration

Devices are set up as users in conf/directory. The critical parameter here is id, which corresponds to the MAC address.
conf/directory/default/skinny-example.xml

# Dialplan configuration

This dialplan snippet is necessary when you need to send a call to a skinny phone. Without this, you will not be able to call your skinny phones!

Add something like this to  (Example with 11xx):
conf/dialplan/default.xml
<extension name="Local_Extension_Skinny">
  <condition field="destination_number" expression="^(11[01][0-9])$">
    <action application="bridge" data="skinny/internal/${destination_number}"/>
  </condition>
</extension> 

# Problem with Cisco Ip Phone
conf/skinny_profiles/internal.xml
<param name="digit-timeout" value="10000"/>

conf/dialplan/skinny-patterns.xml
change this lines
<action application="skinny-wait" data="10" />


# the problem of when you take a handset Cisco phone waits for 10 seconds and then start dial number
Resolution the problem change this values

conf/skinny_profiles/internal.xml
change value 10000 to 2000
 <param name="digit-timeout" value="10000"/> <param name="digit-timeout" value="2000"/>
 
 conf/dialplan/skinny-patterns.xml
 change value 10 to 2
 <action application="skinny-wait" data="10" /> <action application="skinny-wait" data="2" />
 
 after changes enter fs_cli and reload mod_skinny
