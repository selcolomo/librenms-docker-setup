
Deployment of containerized LibreNMS network monitoring on macOS APFS, configured to monitor a Rocky Linux target server (`192.168.99.2`).

- **Host IP:** `192.168.99.2`
- **SNMP Version:** `v2c`
- **Community:** `public`
- **Port Association Mode:** `ifIndex`

Set `--lower-case-table-names=1` in `compose.yml` and `.env` to prevent MariaDB crash loops on case-insensitive filesystems.
