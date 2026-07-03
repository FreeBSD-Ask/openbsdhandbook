# pfctl cheat sheet

## General PFCTL Commands

| Command | Description |
| --- | --- |
| `pfctl -d` | Disable packet-filtering |
| `pfctl -e` | Enable packet-filtering |
| `pfctl -q` | Run quietly |
| `pfctl -v` | Run more verbose than normal |
| `pfctl -v -v` | Run even more verbose |

## Loading PF Rules

| Command | Description |
| --- | --- |
| `pfctl -f /etc/pf.conf` | Load `/etc/pf.conf` |
| `pfctl -n -f /etc/pf.conf` | Test the rules (parse `/etc/pf.conf` but don’t load it) |
| `pfctl -R -f /etc/pf.conf` | Load only the FILTER rules |
| `pfctl -N -f /etc/pf.conf` | Load only the NAT rules |
| `pfctl -O -f /etc/pf.conf` | Load only the OPTION rules |

## Clearing PF Rules & Counters

Flushing rules does not influence or impact any already existing stateful connections

| Command | Description |
| --- | --- |
| `pfctl -F all` | Flush ALL |
| `pfctl -F rules` | Flush only the RULES |
| `pfctl -F queue` | Flush only QUEUE |
| `pfctl -F nat` | Flush only NAT |
| `pfctl -F info` | Flush all statistics that are not part of any rule |
| `pfctl -z` | Clear all counters |

## Output PF Information

| Command | Description |
| --- | --- |
| `pfctl -s rules` | Show filter information |
| `pfctl -sr` | Show filter information (alternative) |
| `pfctl -v -s rules` | Show filter information with hit count |
| `pfctl -vvsr` | Show filter information with rule numbers |
| `pfctl -v -s nat` | Show NAT information and hit count |
| `pfctl -s nat -i xl1` | Show NAT information for interface `xl1` |
| `pfctl -s queue` | Show QUEUE information |
| `pfctl -s label` | Show LABEL information |
| `pfctl -s state` | Show contents of the STATE table |
| `pfctl -s info` | Show statistics for state tables and packet normalization |
| `pfctl -s all` | Show everything |

## Maintaining PF Tables

| Command | Description |
| --- | --- |
| `pfctl -t addvhosts -T show` | Show table `addvhosts` |
| `pfctl -vvsTables` | View global information about all tables |
| `pfctl -t addvhosts -T add 192.168.0.5` | Add entry to table `addvhosts` |
| `pfctl -t addvhosts -T add 192.168.0.0/16` | Add a network to table `addvhosts` |
| `pfctl -t addvhosts -T delete 192.168.0.0/16` | Delete network from table `addvhosts` |
| `pfctl -t addvhosts -T flush` | Remove all entries from table `addvhosts` |
| `pfctl -t addvhosts -T kill` | Delete table `addvhosts` entirely |
| `pfctl -t addvhosts -T replace -f /etc/addvhosts` | Reload table `addvhosts` on the fly |
| `pfctl -t addvhosts -T test 192.168.0.140` | Find IP address `192.168.0.140` in table `addvhosts` |
| `pfctl -T load -f /etc/pf.conf` | Load a new table definition |
| `pfctl -t addvhosts -T show -vi` | Output stats for each IP address in table `addvhosts` |
| `pfctl -t addvhosts -T zero` | Reset all counters for table `addvhosts` |
