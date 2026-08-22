# QNAP TS-869 Pro Zabbix 7.4 SNMP Template

Community Zabbix template for monitoring a **QNAP TS-869 Pro** running legacy **QTS 4.3.x** over SNMP.

This template was built from live SNMP output from a TS-869 Pro rather than from assumptions about newer QNAP models. It is intended to provide useful monitoring for older QNAP systems whose SNMP layout differs from current QTS/QuTS hero devices.

## Tested configuration

| Component | Tested value |
|---|---|
| NAS | QNAP TS-869 Pro |
| Firmware | QTS 4.3.4 |
| Zabbix | 7.4.13 |
| SNMP | v2c |
| SNMP polling port | UDP 161 |
| QNAP enterprise tree | `1.3.6.1.4.1.24681` |
| CPU / memory MIB | HOST-RESOURCES-MIB |

The template may work with other older QNAP NAS models that expose the same QNAP enterprise OIDs and HOST-RESOURCES-MIB layout, but only the TS-869 Pro/QTS 4.3.4 combination above has been validated.

## Features

The template includes monitoring for:

- SNMP availability
- System description
- All **8 physical drive bays**
- Disk model
- Disk capacity
- Disk health
- Disk temperature
- Both system fan speeds
- DataVol1 total capacity
- DataVol1 free capacity
- DataVol1 utilization percentage
- DataVol1 status
- Four logical CPU processor loads
- Average CPU utilization
- Physical memory total
- Raw physical memory used
- Memory buffers
- Cached memory
- Calculated effective memory used
- Calculated available memory
- Calculated memory utilization

It also includes graphs for:

- CPU utilization
- Memory utilization
- Disk temperatures
- Fan speeds
- Volume utilization

## Alerts / triggers

Default triggers include:

| Trigger | Default condition |
|---|---|
| SNMP unavailable | SNMP unavailable for 5 minutes |
| Disk health problem | Disk health is neither `GOOD` nor `--` |
| High disk temperature | `>= 50 °C` |
| Fan stopped | Fan RPM remains `0` for 5 minutes |
| Volume not ready | DataVol1 status is not `Ready` |
| High volume utilization | `>= 90%` |
| High CPU utilization | Average CPU `>= 90%` for 10 minutes |
| High memory utilization | Effective RAM utilization `>= 90%` for 10 minutes |

The thresholds can be changed with template or host macros.

## Why HDD8 is included even when empty

The TS-869 Pro reports all eight bays over SNMP. An empty bay returns placeholder values such as:

- Temperature: `0`
- Capacity: `0`
- Health: `--`

The template intentionally keeps HDD8 configured so that a newly installed drive begins reporting automatically.

The disk-health trigger excludes `--`, and the temperature trigger ignores a value of `0`, so an empty bay should not generate false alerts.

## Template macros

| Macro | Default | Description |
|---|---:|---|
| `{$SNMP_COMMUNITY}` | `CHANGE_ME` | SNMPv2c community configured on the NAS |
| `{$QNAP.DISK.TEMP.WARN}` | `50` | Disk temperature warning threshold in °C |
| `{$QNAP.VOLUME.USED.WARN}` | `90` | Volume utilization warning threshold in % |
| `{$QNAP.CPU.UTIL.WARN}` | `90` | Average CPU warning threshold in % |
| `{$QNAP.MEM.UTIL.WARN}` | `90` | Effective memory utilization warning threshold in % |

For security, do not leave `{$SNMP_COMMUNITY}` set to `CHANGE_ME`. Set it to the value used by your NAS, preferably at the host level.

## QNAP SNMP configuration

On QTS 4.3.x:

1. Open **Control Panel**.
2. Locate **SNMP**.
3. Enable the SNMP service.
4. Use UDP port **161**.
5. Enable SNMP v1/v2 if using this template as provided.
6. Configure a non-default community string.
7. Apply the settings.

SNMP traps are optional and are **not required** for this template. This template uses SNMP polling.

If traps are enabled on the NAS, they normally target an SNMP trap receiver on UDP port 162. Configuring Zabbix to receive traps is a separate task from using this polling template.

## Verify SNMP before importing

From the Zabbix server or Zabbix server container, verify that the NAS responds:

```bash
snmpwalk -v2c -c YOUR_COMMUNITY NAS_IP 1.3.6.1.2.1.1
```

A successful response should include a system description similar to:

```text
SNMPv2-MIB::sysDescr.0 = STRING: Linux TS-869 4.3.4
```

You can also verify the QNAP enterprise tree:

```bash
snmpwalk -v2c -c YOUR_COMMUNITY NAS_IP 1.3.6.1.4.1.24681
```

## Docker note

If Zabbix Server runs in Docker, test SNMP from the **Zabbix server container**, not only from the Docker host.

For example:

```bash
docker exec -it <zabbix-server-container> sh
```

Some Zabbix container images do not include `snmpwalk`. On Alpine-based images it can be installed temporarily as root:

```bash
docker exec -u 0 -it <zabbix-server-container> sh
apk add --no-cache net-snmp-tools
```

This package installation is normally lost when the container is recreated unless it is built into a custom image.

## Importing the template

In Zabbix 7.4:

1. Go to **Data collection → Templates**.
2. Click **Import**.
3. Select:

   `templates/qnap-ts869-pro-zabbix-7.4.yaml`

4. Import the template.
5. Create or edit the QNAP host.
6. Add an **SNMP interface**.
7. Set the NAS IP address.
8. Set port to `161`.
9. Link the template **QNAP TS-869 Pro by SNMP**.
10. Set `{$SNMP_COMMUNITY}` to the community configured on the NAS.

## Suggested host configuration

Example:

```text
Host name: QNAP TS-869 Pro
Interface type: SNMP
IP: <NAS_IP>
Port: 161
SNMP version: SNMPv2
Community: {$SNMP_COMMUNITY}
```

The community macro may be defined on the template, but overriding it on the host is recommended.

## Important OIDs used

### QNAP disk table

| Metric | OID pattern |
|---|---|
| Disk label | `1.3.6.1.4.1.24681.1.3.11.1.2.X` |
| Disk temperature | `1.3.6.1.4.1.24681.1.3.11.1.3.X` |
| Disk model | `1.3.6.1.4.1.24681.1.3.11.1.5.X` |
| Disk capacity | `1.3.6.1.4.1.24681.1.3.11.1.6.X` |
| Disk health | `1.3.6.1.4.1.24681.1.3.11.1.7.X` |

`X` is the drive bay number, `1` through `8`.

The numeric `.1.3` QNAP branch is used where possible because it returns values such as temperature and capacity in formats that are easier for Zabbix to graph and trigger on.

### Fan table

| Metric | OID pattern |
|---|---|
| Fan name | `1.3.6.1.4.1.24681.1.3.15.1.2.X` |
| Fan RPM | `1.3.6.1.4.1.24681.1.3.15.1.3.X` |

The TS-869 Pro exposes two system fans.

### Volume table

| Metric | OID |
|---|---|
| Volume name | `1.3.6.1.4.1.24681.1.3.17.1.2.1` |
| Filesystem | `1.3.6.1.4.1.24681.1.3.17.1.3.1` |
| Total space | `1.3.6.1.4.1.24681.1.3.17.1.4.1` |
| Free space | `1.3.6.1.4.1.24681.1.3.17.1.5.1` |
| Status | `1.3.6.1.4.1.24681.1.3.17.1.6.1` |

The tested NAS exposes one volume named `DataVol1`.

### CPU

CPU utilization is collected from HOST-RESOURCES-MIB:

```text
1.3.6.1.2.1.25.3.3.1.2.<processor-index>
```

The tested TS-869 Pro exposed four processor indexes:

```text
768
769
770
771
```

The template collects all four and calculates the average.

The QNAP enterprise tree also exposes a formatted CPU value:

```text
1.3.6.1.4.1.24681.1.2.1.0
```

which returns a string such as:

```text
"0.00 %"
```

The template uses the standard numeric HOST-RESOURCES-MIB processor values instead.

### Memory

The tested QTS release does **not** expose the common UCD-SNMP memory tree at:

```text
1.3.6.1.4.1.2021.4
```

Instead, physical memory is available through HOST-RESOURCES-MIB.

The physical-memory entry is index `1`:

```text
hrStorageDescr.1 = "Physical memory"
hrStorageAllocationUnits.1 = 1024 Bytes
```

The template uses:

| Metric | OID |
|---|---|
| Physical memory total | `1.3.6.1.2.1.25.2.3.1.5.1` |
| Physical memory used | `1.3.6.1.2.1.25.2.3.1.6.1` |
| Memory buffers | `1.3.6.1.2.1.25.2.3.1.6.6` |
| Cached memory | `1.3.6.1.2.1.25.2.3.1.6.7` |

The raw HOST-RESOURCES-MIB memory values are expressed in allocation units of 1024 bytes, so the template multiplies them by 1024.

### Effective memory utilization

Linux aggressively uses otherwise-idle RAM for cache and buffers. Treating all cached RAM as unavailable produces misleadingly high memory utilization.

The template therefore calculates:

```text
effective used =
    raw physical used
    - buffers
    - cached memory
```

and:

```text
memory utilization % =
    effective used / physical total * 100
```

This is intended to provide a more useful operational view of memory pressure on QTS 4.3.x.

## Graphs

The included graphs are:

### QNAP: CPU utilization

Shows all four logical processor loads plus calculated average utilization.

### QNAP: Memory utilization

Shows calculated effective physical memory utilization.

### QNAP: Disk temperatures

Shows temperature for HDD1 through HDD8 on one graph.

An empty drive bay may report `0 °C`.

### QNAP: Fan speeds

Shows RPM for both system fans.

### QNAP: Volume utilization

Shows calculated percentage utilization for DataVol1.

## Current template design

This template intentionally defines HDD1-HDD8 explicitly rather than using Low-Level Discovery.

Why:

- The TS-869 Pro has a fixed eight-bay chassis.
- QTS exposes all eight indexes even when a bay is empty.
- Explicit items are easy to understand and troubleshoot.
- A replacement or newly installed disk immediately begins populating the existing item set.

LLD could be added in a future version if broader model compatibility becomes a goal.

## Compatibility notes

This template is designed around the OIDs observed on a legacy TS-869 Pro.

QNAP has changed SNMP behavior across product and firmware generations. A template that works on newer QTS or QuTS hero systems may not work on legacy QTS, and vice versa.

Before reporting incompatibility on another QNAP model, compare these walks:

```bash
snmpwalk -v2c -c YOUR_COMMUNITY NAS_IP 1.3.6.1.4.1.24681
snmpwalk -v2c -c YOUR_COMMUNITY NAS_IP 1.3.6.1.2.1.25.2.3.1
snmpwalk -v2c -c YOUR_COMMUNITY NAS_IP 1.3.6.1.2.1.25.3.3.1.2
```

If the relevant tables are structurally similar, support for that model may be straightforward.

## Troubleshooting

### SNMP item is unsupported

Verify the exact OID from the Zabbix server:

```bash
snmpget -v2c -c YOUR_COMMUNITY NAS_IP <OID>
```

If the OID returns `No Such Object`, your QTS/model may expose a different SNMP layout.

### Zabbix reports timeout

Check:

- NAS IP address
- SNMP service enabled on the QNAP
- Community string
- UDP 161 firewall rules
- Docker/container routing
- VLAN/firewall ACLs between Zabbix and the NAS

### SNMP works on the Docker host but not in Zabbix

Test from the actual Zabbix Server container. Docker bridge/overlay networking can behave differently from the host network.

### HDD8 shows zero or placeholder values

This is expected when bay 8 is empty.

### Memory utilization looks different from the QTS GUI

QTS and Zabbix may account for Linux filesystem cache differently. This template subtracts QTS-reported buffers and cache from raw used RAM to estimate effective memory use.

### Graphs from an older template still appear

Zabbix graphs are linked to specific item definitions. Importing or linking this template does not automatically rewrite graphs inherited from another template. Unlink/remove the old template when appropriate, or use the graphs included with this template.

## SNMP traps

This template does not currently consume SNMP traps.

The QNAP can be configured to send traps to a Zabbix server or another trap receiver on UDP 162. Trap processing requires additional receiver configuration on the Zabbix side and, when Zabbix runs in Docker, appropriate UDP 162 exposure/routing.

Polling and traps are independent; the template works without traps.

## Security considerations

- Avoid the default SNMP community `public`.
- Use a unique community string.
- Restrict UDP 161 access with network ACLs/firewalls where practical.
- Do not commit your real SNMP community into a public repository.
- Consider SNMPv3 for environments that require authentication and encryption. This template was tested with SNMPv2c.

## Repository layout

```text
qnap-ts869-zabbix/
├── README.md
├── LICENSE
├── CHANGELOG.md
├── CONTRIBUTING.md
├── .gitignore
├── templates/
│   └── qnap-ts869-pro-zabbix-7.4.yaml
└── screenshots/
    └── README.md
```

## Contributing

Reports from other QNAP models are welcome.

Useful information for compatibility reports includes:

- QNAP model
- QTS/QuTS version
- Zabbix version
- Sanitized output from `1.3.6.1.4.1.24681`
- Sanitized HOST-RESOURCES-MIB storage output
- Sanitized HOST-RESOURCES-MIB processor output
- Which items are unsupported or return unexpected values

Please remove hostnames, serial numbers, IP addresses, community strings, share names, and any other sensitive information before posting SNMP walks publicly.

## License

MIT. See [LICENSE](LICENSE).

## Disclaimer

This is a community-maintained template and is not an official QNAP or Zabbix project.

QNAP and Zabbix product names and trademarks belong to their respective owners.
