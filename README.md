***Configuring Mac & Server***
Configure macOS APFS to monitor Rocky Linux target server

- **Host IP:** `192.168.99.2`
- **SNMP Version:** `v2c`
- **Community:** `public`
- **Port Association Mode:** `ifIndex`

**Physical Link & IP Configuration**
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

 **Rocky Linux Target Server Configuration**
 - Install & Configure SNMP Daemon (background service that runs on network devices or servers to monitor the host's health, collect performance metrics, and processes requests from manager applications)
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
- Configure firewall rules
   - Allow incoming SNMP traffic through `firewalld`
     ```
     sudo firewall-cmd --permanent --add-port=161/udp
     sudo firewall-cmd --reload
     ```

**MacOS Docker Compose Setup (LibreNMS Host)**
- Directory & Environment Preparation
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
  - Create `compose.yml` defining LibreNMS, MariaDB, and Redis
  - Launch container stack in detatched mode
    ```
    docker compose up -d
    ```

**Verification & Device Onboarding**
- Test SNMP Polling from Mac
    - Query target Rocky Linux server from host/container to verify SNMP response
      ```
      docker exec -it librenms snmpwalk -v2c -c public 192.168.99.2
      ```
  - Successful when MIB system details (`sysDescr`, `sysUptime`) return without timing out

 **Web UI Onboarding**
 - Navigate to `http://localhost:8000` in browser to complete setup wizard
 - Add target device
 - Trigger inital discovery and polling pass manually
   ```
   docker exec -it librenms ffping 192.168.99.2
   docker exec -it librenms ./discovery.php -h 192.168.99.2
   docker exec -it librenms ./poller.php -h 192.168.99.2
   ```
