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
