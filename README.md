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
# Download script
wget https://raw.githubusercontent.com/yourrepo/network_monitor_prod.sh
chmod +x network_monitor_prod.sh

# Optional: Install systemd service
sudo cp network_monitor_prod.sh /usr/local/bin/
sudo cp network-monitor.service /etc/systemd/system/
sudo systemctl enable network-monitor.service
sudo systemctl start network-monitor.service
```

## Usage

### Basic
```bash
# Monitor eth0, 5-second intervals, default log location
sudo ./network_monitor_prod.sh eth0 5

# Custom log file
sudo ./network_monitor_prod.sh bond0 10 /var/log/custom.log

# Background execution
sudo nohup ./network_monitor_prod.sh eth0 5 /var/log/network.log &
```

### Systemd Service
```bash
# View live output
sudo journalctl -u network-monitor.service -f

# Check status
sudo systemctl status network-monitor.service

# Restart
sudo systemctl restart network-monitor.service
```

## Output Format

### Console
```
[2026-02-03 11:23:45] #0123 | ✓ OK | Drops: 0
[2026-02-03 11:23:50] #0124 | ✖ CRIT | Drops: 523
  ├─ Softirq Dropped: 523
```

### CSV Log
```csv
timestamp,iteration,interface,total_drops,nic_rx,nic_tx,nic_missed,qdisc,softirq,syn_queue,accept_queue,tcp_pruned,tcp_collapsed,udp_rcvbuf,udp_sndbuf,severity
2026-02-03 11:23:50,124,eth0,523,0,0,0,0,523,0,0,0,0,0,0,CRIT
```

## Analysis
```bash
# Run analysis script
./analyze_drops.sh /var/log/network_drops.log

# Output includes:
# - Total drops by category
# - Top 10 worst intervals
# - Drops by hour of day
# - Recent activity summary
```

### Quick Log Queries
```bash
# Show all critical events
awk -F',' '$16=="CRIT"' /var/log/network_drops.log

# Count drops by type
awk -F',' 'NR>1 {print "Softirq:", $9, "SYN:", $10, "Accept:", $11}' /var/log/network_drops.log

# Events in last hour
awk -F',' -v hour=$(date -d '1 hour ago' '+%Y-%m-%d %H') '$1 >= hour' /var/log/network_drops.log
```

## Interpretation

| Metric | Layer | Common Causes |
|--------|-------|---------------|
| `nic_rx`/`nic_missed` | NIC Ring Buffer | Ring buffer too small, increase with `ethtool -G` |
| `softirq` | Kernel Queue | `netdev_max_backlog` too small |
| `syn_queue` | TCP SYN Backlog | `tcp_max_syn_backlog` too small, SYN flood |
| `accept_queue` | TCP Accept Queue | `somaxconn` too small, app not calling `accept()` |
| `tcp_pruned` | Socket RX Buffer | `rmem_max` too small, app slow reading |
| `qdisc` | Traffic Control | `txqueuelen` too small, rate limiting |

## Tuning Examples
```bash
# Increase NIC ring buffer
ethtool -G eth0 rx 4096 tx 4096

# Increase kernel queues
sysctl -w net.core.netdev_max_backlog=5000
sysctl -w net.core.somaxconn=4096
sysctl -w net.ipv4.tcp_max_syn_backlog=8192

# Increase socket buffers
sysctl -w net.core.rmem_max=16777216
sysctl -w net.ipv4.tcp_rmem="4096 87380 16777216"

# Make persistent
echo "net.core.netdev_max_backlog=5000" >> /etc/sysctl.conf
sysctl -p
```

## Bonding Support

Automatically detects and reports bond configuration:
```bash
sudo ./network_monitor_prod.sh bond0 5
# Output includes:
# Bond Mode:   802.3ad (LACP)
# Bond Slaves: eth0 eth1
```

## Performance Impact

- **CPU**: ~0.1% per interface
- **Disk I/O**: ~10KB per hour (CSV logging)
- **Memory**: <5MB resident

## Troubleshooting

### No drops detected despite high load
```bash
# Verify traffic is reaching interface
tcpdump -i eth0 -c 100

# Check if service is listening
ss -tlnp | grep <port>

# Confirm monitoring correct interface
ip link show
```

### Permissions errors
```bash
# Script requires root for ethtool, tc commands
sudo ./network_monitor_prod.sh eth0 5
```

### High drop counts on idle system
```bash
# Counters are cumulative since boot
# Use delta analysis (built into script)
# Or reset interface statistics
ethtool -S eth0 # Note baseline, compare after interval
```

## Log Rotation
```bash
# Create logrotate config
sudo tee /etc/logrotate.d/network-monitor << EOF
/var/log/network_drops.log {
    daily
    rotate 30
    compress
    delaycompress
    notifempty
    copytruncate
}
EOF
```

## Integration

### Grafana/Prometheus
Parse CSV logs into time-series database:
```bash
# Example: telegraf tail plugin
[[inputs.tail]]
  files = ["/var/log/network_drops.log"]
  data_format = "csv"
  csv_header_row_count = 1
  csv_timestamp_column = "timestamp"
  csv_timestamp_format = "2006-01-02 15:04:05"
```

### Alerting
```bash
# Alert on critical drops
tail -f /var/log/network_drops.log | grep CRIT | \
  while read line; do
    echo "$line" | mail -s "Network Drops Alert" admin@example.com
  done
```

## License

MIT

## Author

Jorge - Senior Technical Support Engineer, Skyhigh Security  
https://jarbytes.dev
