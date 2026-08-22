# Contributing

Contributions and compatibility reports are welcome.

## Compatibility reports

Please include:

- QNAP model
- QTS or QuTS version
- Zabbix version
- Whether SNMPv2c or SNMPv3 is used
- Which items work
- Which items are unsupported or return unexpected values

When useful, include sanitized output from:

```bash
snmpwalk -v2c -c YOUR_COMMUNITY NAS_IP 1.3.6.1.4.1.24681
snmpwalk -v2c -c YOUR_COMMUNITY NAS_IP 1.3.6.1.2.1.25.2.3.1
snmpwalk -v2c -c YOUR_COMMUNITY NAS_IP 1.3.6.1.2.1.25.3.3.1.2
```

## Sanitize diagnostic data

Before posting output publicly, remove or replace:

- SNMP community strings
- IP addresses
- Hostnames
- Serial numbers
- Share names
- Usernames
- Any other environment-specific or sensitive information

## Pull requests

Keep changes focused and describe:

1. the QNAP model/firmware tested,
2. the Zabbix version tested,
3. the OIDs changed or added, and
4. how the behavior was validated.
