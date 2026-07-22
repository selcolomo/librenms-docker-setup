***Configuring Mac & Server***
Configure macOS APFS to monitor Rocky Linux target server

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

- ****Host IP:**** `192.168.99.2`
- **SNMP Version:** `v2c`
- **Community:** `public`
- **Port Association Mode:** `ifIndex`





Deployment of containerized LibreNMS network monitoring on macOS APFS, configured to monitor a Rocky Linux target server (`192.168.99.2`).

- **Host IP:** `192.168.99.2`
- **SNMP Version:** `v2c`
- **Community:** `public`
- **Port Association Mode:** `ifIndex`

Set `--lower-case-table-names=1` in `compose.yml` and `.env` to prevent MariaDB crash loops on case-insensitive filesystems.
