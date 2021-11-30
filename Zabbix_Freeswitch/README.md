# Monitoring FreeSWITCH.service with Zabbix
1. Install zabbix agent in FreeSWITCH server
apt install zabbix-agent
2. Configuration Zabbix-agent in FreeSWITCH.
change these lines in below 
Server= zabbix-server_IP
ServerActive= zabbix-server_IP
Hostname= FreeSWITCH_Hostname
3. Create freeswitch.conf file /etc/zabbix/zabbix_agentd.conf.d and add these lines
UserParameter=services.systemctl,echo "{\"data\":[$(systemctl list-unit-files --type=service|grep \.service|grep -v "@"|grep freeswitch|sed -E -e "s/\.service\s+/\",\"{#STATUS}\":\"/;s/(\s+)?$/\"},/;s/^/{\"{#NAME}\":\"/;$ s/.$//")]}"

UserParameter=systemctl.status[*],systemctl status $1
#UserParameter=systemctl status freeswitch.service
4. Download the template from https://share.zabbix.com/operating-systems/linux/linux-service-monitoring-using-systemctl and import it to the Zabbix Server
5. Change these severities to the high 
 Service "{#NAME}" is not completely started in last {$SERVICE_STARTUP_WINDOW}
 Service "{#NAME}" is not in healthy state
 Service "{#NAME}" is not running but configured at startup
