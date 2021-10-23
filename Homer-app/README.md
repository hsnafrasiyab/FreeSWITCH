apt-get install sqlite
apt-get install fonts-urw-base35
apt-get install unzip
# Installation of Prometheus
cd /usr/src/
wget https://github.com/prometheus/prometheus/releases/download/v2.22.0/prometheus-2.22.0.linux-amd64.tar.gz
tar -xzvf prometheus-2.22.0.linux-amd64.tar.gz
mv prometheus-2.22.0.linux-amd64/ prometheus/
mkdir /home/prometheus
mv /usr/src/prometheus /home/prometheus/prometheus

cat << 'EOF' > /etc/systemd/system/prometheus.service

[Unit]
Description=Prometheus Server
Documentation=https://prometheus.io/docs/introduction/overview/
After=network-online.target

[Service]
User=root
Restart=on-failure

ExecStart=/home/prometheus/prometheus/prometheus --config.file=/home/prometheus/prometheus/prometheus.yml --storage.tsdb.path=/home/prometheus/prometheus/data

[Install]
WantedBy=multi-user.target

EOF

systemctl daemon-reload
systemctl enable --now prometheus.service

mkdir -p /etc/prometheus
ln -s /home/prometheus/prometheus/prometheus.yml /etc/prometheus/prometheus.yml
echo "  - job_name: 'heplify-server'" >> /etc/prometheus/prometheus.yml
echo "    scrape_interval: 5s" >> /etc/prometheus/prometheus.yml
echo "    static_configs:" >> /etc/prometheus/prometheus.yml
echo "    - targets: [':::9096']" >> /etc/prometheus/prometheus.yml
service prometheus restart

# Installatio Grafana Serversudo apt-get install -y adduser libfontconfig1
wget https://dl.grafana.com/enterprise/release/grafana-enterprise_7.2.2_amd64.deb
sudo dpkg -i grafana-enterprise_7.2.2_amd64.deb

systemctl daemon-reload
systemctl enable --now grafana-server.service

cat << EOF | sqlite3 /var/lib/grafana/grafana.db || echo "Failed to add data source."
INSERT INTO data_source VALUES (2,1,0,'prometheus','Prometheus','proxy','http://localhost:9090',NULL,NULL,NULL,0,NULL,NULL,1,'{"httpMethod":"GET","keepCookies":[]}','2017-01-15 20:00:00','2017-01-15 20:00:00',0,'{}',NULL,1);
EOF

Install preconfigured Grafana Dashboards from https://github.com/sipcapture/homer-docker/tree/master/heplify-server/hom7-hep-prom-graf/grafana/provisioning/dashboards

Put all files to /etc/grafana/provisioning/dashboards/ and
service grafana-server restart

# Installation PostgreSQL

wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" |sudo tee  /etc/apt/sources.list.d/pgdg.list

sudo apt update
sudo apt -y install postgresql-12 postgresql-client-12

 
su postgres

 psql -U postgres -d postgres -c "alter user postgres with password 'postgres';"

sed -i "s|ident\+|password|g" /etc/postgresql/12/main/pg_hba.conf

systemctl restart postgresql

# Installation of Heplify Server
apt-get install libluajit-5.1-dev

heplify-server installation

curl -s https://packagecloud.io/install/repositories/qxip/sipcapture/script.deb.sh | sudo bash

sudo apt-get install homer-app=1.4.18

ln -s /etc/heplify-server.toml /etc/heplify-server/heplify-server.tom

Execute whole text till EOF at the end:

cat << 'EOF' > /etc/heplify-server.toml
HEPAddr = "0.0.0.0:9060"
HEPTCPAddr = ""
HEPTLSAddr = "0.0.0.0:9060"
ESAddr = ""
ESDiscovery = false
LokiURL = ""
LokiBulk = 200
LokiTimer = 4
LokiBuffer = 100000
LokiHEPFilter = [1,5,100]
ForceHEPPayload = []
PromAddr = "0.0.0.0:9096"
PromTargetIP = ""
PromTargetName = ""
DBShema = "homer7"
DBDriver = "postgres"
DBAddr = "127.0.0.1:5432"
DBUser = "postgres"
DBPass = "postgres"
DBDataTable = "homer_data"
DBConfTable = "homer_config"
DBBulk = 200
DBTimer = 4
DBBuffer = 400000
DBWorker = 8
DBRotate = true
DBPartLog = "2h"
DBPartSip = "1h"
DBPartQos = "6h"
DBDropDays = 14
DBDropDaysCall = 0
DBDropDaysRegister = 0
DBDropDaysDefault = 0
DBDropOnStart = false
Dedup = false
DiscardMethod = ["OPTIONS","NOTIFY"]
AlegIDs = []
CustomHeader = []
SIPHeader = []
LogDbg = "hep,sql"
LogLvl = "warning"
LogStd = false
LogSys = false
Config = "./heplify-server.toml"
ConfigHTTPAddr = ""
EOF

sed -i '/LokiURL               = ""/c\LokiURL               = "http://localhost:3100/loki/api/v1/push"' heplify-server.toml
sed -i '/HEPWSAddr             = "0.0.0.0:3000"/c\HEPWSAddr             = "0.0.0.0:3002"' heplify-server.toml
sed -i '/DBPass                = ""/c\DBPass                = "postgres"' heplify-server.toml

mkdir -p /var/log/homer

vim /etc/systemd/system/heplify-server.service

[Unit]
Description=HEP Server & Switch in Go
After=network.target

[Service]
WorkingDirectory=/var/log/homer
Environment="HEPLIFY_CONFIG=-config=/etc/heplify-server.toml"
ExecStart=/usr/local/bin/heplify-server $HEPLIFY_CONFIG
ExecStop=/bin/kill ${MAINPID}
Restart=on-failure
RestartSec=10s
Type=simple

[Install]
WantedBy=multi-user.target

systemctl daemon-reload
systemctl enable --now heplify-server

# Installation of  Homer-app

apt install homer-app

systemctl enable --now homer-app

homer-app -create-config-db -database-root-user=postgres -database-host="127.0.0.1" -database-root-password=postgres -database-homer-user=homer_user
homer-app -create-data-db -database-root-user=postgres -database-host="127.0.0.1" -database-root-password=postgres -database-homer-user=homer_user
homer-app -create-table-db-config
homer-app -populate-table-db-config
homer-app -upgrade-table-db-config
service homer-app restart

# Installation of Loki & Promtail

Loki
curl -s https://api.github.com/repos/grafana/loki/releases/latest | grep browser_download_url |  cut -d '"' -f 4 | grep loki-linux-amd64.zip | wget -i -

unzip loki-linux-amd64.zip
sudo mv loki-linux-amd64 /usr/local/bin/loki

Add the following configuration to the file:

loki --version
sudo mkdir -p /data/loki
sudo vim /etc/loki-local-config.yaml

auth_enabled: false

server:
  http_listen_port: 3100

ingester:
  lifecycler:
    address: 127.0.0.1
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
    final_sleep: 0s
  chunk_idle_period: 5m
  chunk_retain_period: 30s
  max_transfer_retries: 0

schema_config:
  configs:
    - from: 2018-04-15
      store: boltdb
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 168h

storage_config:
  boltdb:
    directory: /data/loki/index

  filesystem:
    directory: /data/loki/chunks

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h

chunk_store_config:
  max_look_back_period: 0s

table_manager:
  retention_deletes_enabled: false
  retention_period: 0s

sudo tee /etc/systemd/system/loki.service<<EOF
[Unit]
Description=Loki service
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/loki -config.file /etc/loki-local-config.yaml

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start loki.service

curl -s https://api.github.com/repos/grafana/loki/releases/latest | grep browser_download_url |  cut -d '"' -f 4 | grep promtail-linux-amd64.zip | wget -i -

unzip promtail-linux-amd64.zip
sudo mv promtail-linux-amd64 /usr/local/bin/promtail

promtail --version

sudo vim /etc/promtail-local-config.yaml

Add the following content to the file:
server:
  http_listen_port: 9070
  grpc_listen_port: 0

positions:
  filename: /data/loki/positions.yaml

clients:
  - url: http://localhost:3100/loki/api/v1/push

scrape_configs:
- job_name: system
  static_configs:
  - targets:
      - localhost
    labels:
      job: varlogs
      __path__: /var/log/*log

sudo tee /etc/systemd/system/promtail.service<<EOF
[Unit]
Description=Promtail service
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/promtail -config.file /etc/promtail-local-config.yaml

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start promtail.service
                                                  
# Installation of Heplify server agent
                                                  
apt update && apt upgrade -y && apt install sudo -y && apt install libpcap-dev -y
 wget https://dl.google.com/go/go1.13.1.linux-amd64.tar.gz && tar -xvf $ go1.13.1.linux-amd64.tar.gz
mv go /usr/local && export GOROOT=/usr/local/go && export GOPATH=$HOME/Projects/Proj1
 export PATH=$GOPATH/bin:$GOROOT/bin:$PATH

 git clone https://github.com/sipcapture/heplify
 pushd heplify/ && make && popd && mv heplify/ /opt/heplify && cd /opt/heplify && chmod -R 777 *


cat <<EOF > /etc/systemd/system/heplify.service
[Unit]
Description=Captures packets from wire and sends them to Homer
After=network.target

[Service]
WorkingDirectory=/opt/heplify
ExecStart=/opt/heplify/heplify -i ens192 -hs 10.47.47.71:9060 -m SIPRTCP
ExecStop=/bin/kill ${MAINPID}
Restart=on-failure
RestartSec=10s
Type=simple

[Install]
WantedBy=multi-user.target
EOF

 systemctl daemon-reload
 systemctl enable heplify.service
systemctl start heplify.service


    Add the following lines to /etc/freeswitch/autoload_configs/sofia.conf.xml : You should Change IP to your Homer7 Server IP

<param name="capture-server" value="udp:10.47.47.71:9060"/>


    Open /etc/freeswitch/sip_profiles/internal.xml and change sip-capture param to "yes" :

<param name="sip-capture" value="yes"/>

Default Credentials for Homer7

    Username : admin

    Password : sipcapture





