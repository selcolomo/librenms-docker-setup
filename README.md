## Configuring Mac & Server
Configure macOS APFS to monitor Rocky Linux target server

- **Host IP:** `192.168.99.2`
- **SNMP Version:** `v2c`
- **Community:** `public`
- **Port Association Mode:** `ifIndex`

#### Physical Link & IP Configuration
- Determine Ethernet interface
  ```
  ip link show
  ```
- Assign static IP address to specfic interface & test connection from server to Mac
  ```
  sudo ip addr 192.168.99.2/24 dev eno4
  ping -c 3 192.168.99.1
  ```
  - Successful when 0% packet loss

#### Rocky Linux Target Server Configuration
 - **Install & Configure SNMP Daemon (background service that runs on network devices or servers to monitor the host's health, collect performance metrics, and processes requests from manager applications)**
- Install `net-snmp` packages
  ```
  sudo dnf install -y net-snmp net-snmp-utils
  ```
- Back up original configuration and set up minimal SNMPv2c listener (software program that is designed to passively wait and listen for unsolicted alert messages)
  ```
  sudo mv /etc/snmp/snmpd.conf /etc/snmp/snmpd.conf.bak
  sudo bash -c 'cat <<EOF > /etc/snmp/snmpd.conf
  agentAddress udp:161
  rocommunity public 192.168.99.0/24
  EOF'
  ```
- Enable and start `nsmpd` service
  ```
  sudo systemctl enable --now snmpd
  ```
- **Configure firewall rules**
- Allow incoming SNMP traffic through `firewalld`
  ```
  sudo firewall-cmd --permanent --add-port=161/udp
  sudo firewall-cmd --reload
  ```

## Monitoring Platforms

### LibreNMS
#### MacOS Docker Compose Setup (LibreNMS Host)
- **Directory & Environment Preparation**
- Create project directory and navigate into it
  ```
  mdkir -p ~/librenms && cd ~/librenms
  ```
- Create `.env` file for LibreNMS container environment variables (ignored by Git to prevent exposure of passwords and sensitive information)
  ```
  cat <<EOF> .env
  TZ=America/Los_Angeles
  PUID=1000
  PGID=1000
  MYSQL_ROOT_PASSWORD=root_db_pass
  MYSQL_PASSWORD=librenms_db_pass
  MYSQL_USER=librenms
  MYSQL_DATABASE=librenms
  EOF
  ```
- **MacOS APFS Database Workaround:**
  - Create `compose.yml` to enforece `--lower-case-table-names=1` on the `db` service. This prevents MariaDB crash loops caused by macOS's default case-insensitive filesystem (APFS).
- Launch container stack in detatched mode
  ```
  docker compose up -d
  ```

#### Verification & Device Onboarding
- **Test SNMP Polling from Mac**
- Query target Rocky Linux server from host/container to verify SNMP response
  ```
  docker exec -it librenms snmpwalk -v2c -c public 192.168.99.2
  ```
  - Successful when MIB system details (`sysDescr`, `sysUptime`) return without timing out

#### Web UI Onboarding
 - Navigate to `http://localhost:8000` in browser to complete setup wizard
 - Add target device `192.168.99.2` and trigger inital discovery and polling pass manually
   ```
   docker exec -it librenms ffping 192.168.99.2
   docker exec -it librenms ./discovery.php -h 192.168.99.2
   docker exec -it librenms ./poller.php -h 192.168.99.2
   ```

### Prometheus + Grafana
#### Download & Transfer Node Exporter from Mac
- Download Node Exporter Binary on Mac
  ```
  cd ~/Downloads
  curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.1.linux-amd64.tar.gz
  ```
- Transfer Archive to Rocky Linux via SCP
  ```
  scp node_exporter-1.8.1.linux-amd64.tar.gz root@192.168.99.2:/tmp/
  ```

#### Offline Node Exporter Installation on Server
- Verify the transferred tarball exists in `/tmp`
  ```
  ls -l /tmp/node_exporter-1.8.1.linux-amd64.tar.gz
  ```
- Extract the compressed file and move the executable into a system directory so it can be run as a service:
  ```
  cd /tmp
  tar xvfz node_exporter-1.8.1.linux-amd64.tar.gz
  sudo mv node_exporter-1.8.1.linux-amd64/node_exporter /usr/local/bin/
  sudo rm -rf node_exporter-1.8.1.linux-amd64*
  ```
- Create a `systemd` service which will allow Node Exporter to run quietly in the background without needing an open terminal window
  ```
  sudo bash -c 'cat <<EOF > /etc/systemd/system/node_exporter.service
  [Unit]
  Description=Node Exporter
  After=network.target
  
  [Service]
  User=nobody
  ExecStart=/usr/local/bin/node_exporter
  
  [Install]
  WantedBy=multi-user.target
  EOF'
  ```
  
#### Enable and Start Node Exporter
- Tell `systemd` to register the new service file and start Node Exporter:
  ```
  sudo systemctl daemon-reload
  sudo systemctl enable --now node_exporter
  sudo systemctl status node_exporter
  ```
  
#### Configure Firewalld for TCP Port 9100
- Open TCP port `9100` so Prometheus running on the Mac can pull metrics across the Ethernet cable
  ```
  sudo firewall-cmd --permanent --add-port=9100/tcp
  sudo firewall-cmd --reload
  ```
#### Test Local Metrics
- Verify that Node Exporter is listening and producing system metrics locally on TCP port 9100.
  ```
  nc -zv 127.0.0.1 9100
  ```
- Expected Outpout:
  ```
  Ncat: Connected to 127.0.0.1:9100.
  ```
  
#### Verify Prometheus is Scraping Rocky Linux
- Open browser and go to: `http://localhost:9090/targets`
- Look at `rocky-linux-server` target and verify that it is `UP` and Prometheus is successfully pulling metrics from `192.168.99.2:9100`
  
#### Configure Grafana Dashboard
- Open `http://localhost:3000` in browser
  - Username: `admin`
  - Password: `admin`
- **Add Prometheus as a Data Source**
- In Prometheus server URL enter: `http://prometheus:9090`
- **Import the Official Node Exporter Dashboard**
- In the *Find and import dashboards* fields enter, `1860` and import

### Zabbix
#### Local Environment Preparation 
- Create local directory
  ```
  cd ~/monitoring-lab
  mkdir zabbix
  ```
- Create an `.env` file to manage environment variables:
  ```
  cat << 'EOF' > .env
  POSTGRES_USER=zabbix
  POSTGRES_PASSWORD=zabbix_password
  POSTGRES_DB=zabbix
  PHP_TZ=America/Los_Angeles
  EOF
  ```
- Create `compose.yml` file for Zabbix inside directory
- **Download `amd64` Images on Mac**
  ```
  regctl image export --platform linux/amd64 postgres:15-alpine postgres.tar
  regctl image export --platform linux/amd64 zabbix/zabbix-server-pgsql:alpine-7.0-latest zabbix-server.tar
  regctl image export --platform linux/amd64 zabbix/zabbix-web-nginx-pgsql:alpine-7.0-latest zabbix-web.tar
  regctl image export --platform linux/amd64 zabbix/zabbix-agent:alpine-7.0-latest zabbix-agent.tar
  
  tar -cvf zabbix-images.tar *.tar
  ```
  
#### Transfer Images onto Rocky Linux server
- Transfer files to the server using `scp`
  ```
  scp zabbix-images.tar root@192.168.99.2:~/
  ```
- On the Rocky Linux server, unpack and load the images into Podman:
  ```
  tar -xvf zabbix-images.tar
  podman load -i postgres.tar
  podman load -i zabbix-server.tar
  podman load -i zabbix-web.tar
  podman load -i zabbix-agent.tar
  ```
  
#### Deploy stack using Podman Compose
- Create working folder and live `.env` file on the server:
  ```
  mkdir -p ~/zabbix
  cd ~/zabbix
  
  cat << 'EOF' > .env
  POSTGRES_USER=zabbix
  POSTGRES_PASSWORD=zabbix_password
  POSTGRES_DB=zabbix
  PHP_TZ=America/Los_Angeles
  EOF
  ```
- Copy over `docker-compose.yml` to `~/zabbix/docker-compose.yml` and launch with podman-compose:
  ```
  podman-compose up -d
  ```
- Check running containers
  ```
  podman ps
  ```

#### Web UI Initial Setup
- Open browser and navigate to `http://192.168.99.2:8080`
  - Username: `Admin`
  - Password: `zabbix`


## TAP Configuration
- 
