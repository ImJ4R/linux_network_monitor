```markdown
# Network Stack Monitor

Real-time network bottleneck detection and historical drop analysis for Linux production environments.

## Overview

Continuously monitors packet drops across all network stack layers with per-interval delta tracking. Supports bonded interfaces and provides CSV logging for historical analysis.

**Monitored Components:**
- NIC ring buffers (RX/TX)
- Traffic control (qdisc)
- Softirq/NAPI queues
- TCP SYN backlog
- TCP accept queue
- Socket buffers (TCP/UDP)

## Requirements

- Linux kernel 2.6+
- Root/sudo access
- Dependencies: `ethtool`, `tc`, `netstat` (or `ss`)

## Installation

```bash
# Download scripts
wget https://raw.githubusercontent.com/yourrepo/network_monitor_prod.sh
wget https://raw.githubusercontent.com/yourrepo/analyze_drops.sh
chmod +x network_monitor_prod.sh analyze_drops.sh

# Optional: Install systemd service
sudo cp network_monitor_prod.sh /usr/local/bin/
sudo cp network-monitor.service /etc/systemd/system/
sudo systemctl enable network-monitor.service
sudo systemctl start network-monitor.service
```

## Usage

### Syntax

```bash
./network_monitor_prod.sh [INTERFACE] [INTERVAL] [LOGFILE]
```

### Parameters

| Parameter | Description | Default | Example |
|-----------|-------------|---------|---------|
| `INTERFACE` | Network interface to monitor | `eth0` | `eth0`, `bond0`, `ens33` |
| `INTERVAL` | Sampling interval in seconds | `5` | `5`, `10`, `30` |
| `LOGFILE` | Path to CSV log file | `/var/log/network_drops.log` | `/tmp/test.log` |

### Examples

```bash
# Monitor eth0, 5-second intervals, default log
sudo ./network_monitor_prod.sh eth0 5

# Monitor bond0, 10-second intervals, custom log
sudo ./network_monitor_prod.sh bond0 10 /var/log/custom.log

# Monitor with default settings (eth0, 5s interval)
sudo ./network_monitor_prod.sh

# Background execution
sudo nohup ./network_monitor_prod.sh eth0 5 /var/log/network.log &

# Stop with Ctrl+C or:
sudo pkill -f network_monitor_prod.sh
```

## Configuration

### Alert Threshold

Edit the script to modify the alert threshold (default: 100 drops per interval):

```bash
# In network_monitor_prod.sh, line 13:
ALERT_THRESHOLD=100  # Change to desired value

# Triggers:
# - Green (OK):   0 drops
# - Yellow (WARN): 1-99 drops
# - Red (CRIT):   100+ drops
```

### Monitored Metrics

The script tracks these counters per interval:

| Metric | Source | Description |
|--------|--------|-------------|
| `nic_rx` | `ethtool -S` | NIC RX dropped packets |
| `nic_tx` | `ethtool -S` | NIC TX dropped packets |
| `nic_missed` | `ethtool -S` | NIC RX missed (ring full) |
| `qdisc` | `tc -s qdisc` | Traffic control drops |
| `softirq` | `/proc/net/softnet_stat` | Softirq queue drops (all CPUs) |
| `syn_queue` | `netstat -s` | SYN backlog drops |
| `accept_queue` | `netstat -s` | Accept queue overflows |
| `tcp_pruned` | `netstat -s` | TCP RX buffer pruned |
| `tcp_collapsed` | `netstat -s` | TCP RX buffer collapsed |
| `udp_rcvbuf` | `netstat -su` | UDP RX buffer errors |
| `udp_sndbuf` | `netstat -su` | UDP TX buffer errors |

## Output Format

### Console Output

```
[2026-02-03 11:23:45] #0123 | ✓ OK | Drops: 0
[2026-02-03 11:23:50] #0124 | ⚠ WARN | Drops: 45
  ├─ Softirq Dropped: 45
[2026-02-03 11:23:55] #0125 | ✖ CRIT | Drops: 523
  ├─ Softirq Dropped: 500
  ├─ Accept Queue Overflow: 23
```

**Severity Levels:**
- `✓ OK` (Green): 0 drops
- `⚠ WARN` (Yellow): 1 to `ALERT_THRESHOLD-1` drops
- `✖ CRIT` (Red): ≥ `ALERT_THRESHOLD` drops

### CSV Log Format

```csv
timestamp,iteration,interface,total_drops,nic_rx,nic_tx,nic_missed,qdisc,softirq,syn_queue,accept_queue,tcp_pruned,tcp_collapsed,udp_rcvbuf,udp_sndbuf,severity
2026-02-03 11:23:50,124,eth0,523,0,0,0,0,500,0,23,0,0,0,0,CRIT
2026-02-03 11:23:55,125,eth0,0,0,0,0,0,0,0,0,0,0,0,0,OK
```

## Analysis

### Using analyze_drops.sh

```bash
# Syntax
./analyze_drops.sh [LOGFILE]

# Examples
./analyze_drops.sh /var/log/network_drops.log
./analyze_drops.sh /tmp/test.log

# Sample output:
# - Total intervals monitored
# - Intervals with drops (percentage)
# - Total drops by category
# - Top 10 worst intervals
# - Drops by hour of day
# - Last 20 intervals
```

### Manual Log Queries

```bash
# Show all critical events
awk -F',' '$16=="CRIT"' /var/log/network_drops.log

# Sum drops by category
awk -F',' 'NR>1 {
  nic_rx+=$5; softirq+=$9; syn+=$10; accept+=$11
} END {
  print "NIC RX:", nic_rx
  print "Softirq:", softirq
  print "SYN Queue:", syn
  print "Accept Queue:", accept
}' /var/log/network_drops.log

# Events in last hour
awk -F',' -v hour=$(date -d '1 hour ago' '+%Y-%m-%d %H') '$1 >= hour' /var/log/network_drops.log

# Drops per minute (last hour)
tail -720 /var/log/network_drops.log | awk -F',' '{sum+=$4} END {print sum}'

# Find peak drop time
awk -F',' 'NR>1 {print $1, $4}' /var/log/network_drops.log | sort -k2 -rn | head -1
```

## Systemd Service

### Service File

Create `/etc/systemd/system/network-monitor.service`:

```ini
[Unit]
Description=Network Stack Monitor
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/network_monitor_prod.sh eth0 5 /var/log/network_drops.log
Restart=always
RestartSec=10
User=root

[Install]
WantedBy=multi-user.target
```

### Service Management

```bash
# Enable at boot
sudo systemctl enable network-monitor.service

# Start/stop/restart
sudo systemctl start network-monitor.service
sudo systemctl stop network-monitor.service
sudo systemctl restart network-monitor.service

# View status
sudo systemctl status network-monitor.service

# View logs (real-time)
sudo journalctl -u network-monitor.service -f

# View logs (last 100 lines)
sudo journalctl -u network-monitor.service -n 100
```

### Multiple Interfaces

Monitor multiple interfaces with separate service instances:

```bash
# Create service template
sudo cp /etc/systemd/system/network-monitor.service \
        /etc/systemd/system/network-monitor@.service

# Edit to use instance name:
# ExecStart=/usr/local/bin/network_monitor_prod.sh %i 5 /var/log/network_%i.log

# Start per interface
sudo systemctl start network-monitor@eth0.service
sudo systemctl start network-monitor@eth1.service
sudo systemctl start network-monitor@bond0.service
```

## Interpretation

### Drop Sources & Solutions

| Metric | Layer | Common Causes | Solution |
|--------|-------|---------------|----------|
| `nic_rx`/`nic_missed` | NIC Ring Buffer | Ring buffer too small | `ethtool -G eth0 rx 4096` |
| `softirq` | Kernel Queue | `netdev_max_backlog` too small | `sysctl -w net.core.netdev_max_backlog=5000` |
| `syn_queue` | TCP SYN Backlog | SYN flood, backlog too small | `sysctl -w net.ipv4.tcp_max_syn_backlog=8192` |
| `accept_queue` | TCP Accept Queue | App slow calling `accept()` | `sysctl -w net.core.somaxconn=4096` or fix app |
| `tcp_pruned` | Socket RX Buffer | App slow reading, buffer too small | `sysctl -w net.core.rmem_max=16777216` or fix app |
| `tcp_collapsed` | Socket RX Buffer | Memory pressure | Increase buffers, reduce memory usage |
| `qdisc` | Traffic Control | TX queue too small, rate limiting | `ip link set eth0 txqueuelen 10000` |
| `udp_rcvbuf` | UDP Socket | App slow reading, buffer too small | Increase `rmem_max`, optimize app |
| `udp_sndbuf` | UDP Socket | App sending too fast | Increase `wmem_max`, rate limit app |

### Tuning Commands

```bash
# NIC ring buffers
ethtool -G eth0 rx 4096 tx 4096

# Kernel queues
sysctl -w net.core.netdev_max_backlog=5000
sysctl -w net.core.netdev_budget=600
sysctl -w net.core.netdev_budget_usecs=8000

# TCP queues
sysctl -w net.core.somaxconn=4096
sysctl -w net.ipv4.tcp_max_syn_backlog=8192

# Socket buffers
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216
sysctl -w net.ipv4.tcp_rmem="4096 87380 16777216"
sysctl -w net.ipv4.tcp_wmem="4096 65536 16777216"

# TX queue length
ip link set eth0 txqueuelen 10000

# Make persistent
cat >> /etc/sysctl.conf << EOF
net.core.netdev_max_backlog=5000
net.core.somaxconn=4096
net.ipv4.tcp_max_syn_backlog=8192
net.core.rmem_max=16777216
net.core.wmem_max=16777216
EOF
sysctl -p
```

## Bonding Support

Automatically detects bonded interfaces:

```bash
sudo ./network_monitor_prod.sh bond0 5

# Console output includes:
# Interface:       bond0
# Bond Mode:       802.3ad (LACP)
# Bond Slaves:     eth0 eth1
```

**Note:** Script monitors the bond master only. Individual slave drops are visible via `ethtool -S eth0` on each slave.

## Performance Impact

| Resource | Impact |
|----------|--------|
| CPU | ~0.1% per interface |
| Memory | <5MB resident |
| Disk I/O | ~10KB/hour (5s interval) |
| Network | None (passive monitoring) |

## Troubleshooting

### No drops detected despite high load

```bash
# Verify traffic reaching interface
tcpdump -i eth0 -c 100

# Check application is listening
ss -tlnp | grep <port>

# Confirm monitoring correct interface
ip link show
ip -s link show eth0  # Show interface statistics
```

### Permission denied errors

```bash
# Script requires root for: ethtool, tc, netstat
sudo ./network_monitor_prod.sh eth0 5

# Verify permissions
ls -l network_monitor_prod.sh  # Should be executable
```

### High drop counts on idle system

```bash
# Counters are cumulative since boot/interface up
# Script uses delta analysis (only interval changes matter)

# To reset counters (resets all stats):
ip link set eth0 down && ip link set eth0 up

# Or just note baseline:
ethtool -S eth0 | grep dropped
```

### Interface not found

```bash
# List available interfaces
ip link show

# Check interface naming
dmesg | grep eth
ls /sys/class/net/
```

## Log Rotation

Prevent log file growth with logrotate:

```bash
# Create config
sudo tee /etc/logrotate.d/network-monitor << EOF
/var/log/network_drops.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
}
EOF

# Test rotation
sudo logrotate -f /etc/logrotate.d/network-monitor
```

## Integration Examples

### Grafana/Prometheus

```bash
# Use telegraf tail plugin
[[inputs.tail]]
  files = ["/var/log/network_drops.log"]
  data_format = "csv"
  csv_header_row_count = 1
  csv_timestamp_column = "timestamp"
  csv_timestamp_format = "2006-01-02 15:04:05"
  csv_tag_columns = ["interface", "severity"]
```

### Email Alerting

```bash
# Alert on critical drops
tail -f /var/log/network_drops.log | grep CRIT | \
  while read line; do
    echo "$line" | mail -s "Network Drops: CRITICAL" admin@example.com
  done &
```

### Slack/Webhook

```bash
# Alert to webhook
tail -f /var/log/network_drops.log | grep CRIT | \
  while read line; do
    curl -X POST -H 'Content-type: application/json' \
      --data "{\"text\":\"Network drops: $line\"}" \
      https://hooks.slack.com/services/YOUR/WEBHOOK/URL
  done &
```

## FAQ

**Q: How much historical data should I keep?**  
A: At 5-second intervals, 30 days = ~520MB uncompressed, ~50MB compressed.

**Q: Can I monitor VLANs?**  
A: Yes. Use interface name like `eth0.100` for VLAN 100.

**Q: Does this work on virtual interfaces?**  
A: Yes (tun, tap, veth), but some metrics may not be available.

**Q: What about Docker/container interfaces?**  
A: Yes, monitor `docker0`, `veth*` interfaces normally.

**Q: Can I change the CSV format?**  
A: Yes, edit line 186 in `network_monitor_prod.sh` (the `echo` to `$LOGFILE`).

**Q: Will this work on old kernels (2.6.x)?**  
A: Most features yes, but `/proc/net/softnet_stat` format may differ on very old kernels.

## License

MIT

## Author

Jorge - Senior Technical Support Engineer, Skyhigh Security  
https://jarbytes.dev

## Contributing

Issues and pull requests welcome at: https://github.com/yourrepo/network-monitor
```
