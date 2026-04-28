# TrafficCop - Traffic Monitoring and Limiting Toolkit

English | [中文](README.md)

TrafficCop is a set of Linux shell scripts for VPS traffic monitoring, machine-level traffic limiting, per-port traffic limiting, and traffic notifications. It is designed for servers with monthly, quarterly, or yearly transfer quotas.

## Features

- Machine traffic monitoring based on `vnstat`.
- Traffic modes: `out`, `in`, `total`, and `max`.
- Billing periods: monthly, quarterly, yearly, with a custom start day from 1 to 31.
- Limit actions: TC rate limiting or shutdown/blocking mode.
- Multi-port traffic limits with independent quotas.
- Notifications via Telegram, PushPlus, and ServerChan.
- Interactive manager for installation, configuration, logs, updates, and cleanup.

## Important Notes

1. `vnstat` only records traffic after it is installed and running.
2. TC mode cannot prevent DDoS traffic from consuming your provider quota.
3. Per-port traffic is based on iptables counters. For proxy services, inbound traffic is measured and outbound/total traffic is estimated.
4. Scripts are installed under `/root/TrafficCop` by default and should be run as root or with sudo.
5. Shutdown/blocking mode can interrupt services. TC mode is recommended for first-time setup.
6. Current port configuration uses the latest JSON field format. If upgrading from an old version, back up and reconfigure port limits.

## Quick Start

Use the interactive manager:

```bash
bash <(curl -sL https://raw.githubusercontent.com/Misasasasasaka/TrafficCop/main/trafficcop-manager.sh)
```

Manager menu:

| Option | Action |
| --- | --- |
| 1 | Install/manage machine traffic monitoring |
| 2 | Install/manage Telegram notifications |
| 3 | Install/manage PushPlus notifications |
| 4 | Install/manage ServerChan notifications |
| 5 | Install/manage port traffic limits |
| 6 | View port traffic status |
| 7 | Enable/disable/restore machine traffic limiting |
| 8 | View logs |
| 9 | View current configuration |
| 10 | Apply preset configuration |
| 11 | Stop all TrafficCop services |
| 12 | Update all scripts |

## Tutorial: Machine Traffic Monitoring

1. Start the manager and choose `1) Install/manage traffic monitoring`.
2. Confirm the detected network interface, or enter the correct one such as `eth0`, `ens3`, or `ens4`.
3. Choose a traffic mode:

| Mode | Meaning |
| --- | --- |
| `out` | Count outbound traffic only |
| `in` | Count inbound traffic only |
| `total` | Count inbound + outbound |
| `max` | Use the larger direction |

4. Choose a period: `m` monthly, `q` quarterly, or `y` yearly.
5. Enter the period start day, traffic limit in GB, and tolerance in GB.
6. Choose TC mode for rate limiting or shutdown mode for strict cutoff.

Log file:

```bash
sudo tail -f -n 50 /root/TrafficCop/traffic_monitor.log
```

## Tutorial: Notifications

Install notifications from the manager:

- `2) Telegram notifications`
- `3) PushPlus notifications`
- `4) ServerChan notifications`

Telegram requires:

- Bot Token
- Chat ID
- Topic ID / `message_thread_id`
- Machine name
- Daily report time, for example `08:30`

Useful logs:

```bash
sudo tail -f -n 50 /root/TrafficCop/tg_notifier_cron.log
sudo tail -f -n 50 /root/TrafficCop/pushplus_notifier_cron.log
sudo tail -f -n 50 /root/TrafficCop/serverchan_notifier_cron.log
```

## Tutorial: Per-Port Traffic Limits

Install from the manager with option `5) Install/manage port traffic limits`, or install directly:

```bash
sudo mkdir -p /root/TrafficCop
sudo curl -fsSL https://raw.githubusercontent.com/Misasasasasaka/TrafficCop/main/port_traffic_limit.sh -o /root/TrafficCop/port_traffic_limit.sh
sudo curl -fsSL https://raw.githubusercontent.com/Misasasasasaka/TrafficCop/main/view_port_traffic.sh -o /root/TrafficCop/view_port_traffic.sh
sudo chmod +x /root/TrafficCop/port_traffic_limit.sh /root/TrafficCop/view_port_traffic.sh
sudo bash /root/TrafficCop/port_traffic_limit.sh
```

Add a port from the menu and provide:

| Field | Meaning |
| --- | --- |
| Port | 1-65535, for example `443` |
| Description | Display name for reports |
| Traffic limit | Quota in GB |
| Tolerance | Buffer in GB before limiting |
| Config mode | Sync machine config or customize |
| Limit mode | TC rate limit or blocking mode |

View port traffic:

```bash
sudo bash /root/TrafficCop/view_port_traffic.sh
sudo bash /root/TrafficCop/view_port_traffic.sh --realtime
sudo bash /root/TrafficCop/view_port_traffic.sh --json
sudo bash /root/TrafficCop/view_port_traffic.sh --export
```

Remove limits:

```bash
sudo bash /root/TrafficCop/port_traffic_limit.sh --remove 443
sudo bash /root/TrafficCop/port_traffic_limit.sh --remove
```

## Preset Configurations

Preset files are written to `/root/TrafficCop/traffic_monitor_config.txt`.

```bash
sudo mkdir -p /root/TrafficCop
sudo curl -fsSL -o /root/TrafficCop/traffic_monitor_config.txt https://raw.githubusercontent.com/Misasasasasaka/TrafficCop/main/ali-200g
sudo cat /root/TrafficCop/traffic_monitor_config.txt
```

Available presets:

- `ali-20g`
- `ali-200g`
- `ali-1T`
- `az-15g`
- `az-115g`
- `gcp-200g`
- `gcp-625g`
- `alice-1500g`
- `asia-300g`

Replace the preset filename in the URL as needed.

If the main script is not installed yet, run the Quick Start manager command first. After TrafficCop is installed, apply the downloaded config with:

```bash
sudo bash /root/TrafficCop/trafficcop.sh
```

## Maintenance Commands

View logs:

```bash
sudo tail -f -n 50 /root/TrafficCop/traffic_monitor.log
sudo tail -f -n 50 /root/TrafficCop/port_traffic_monitor.log
```

View configs:

```bash
sudo cat /root/TrafficCop/traffic_monitor_config.txt
sudo cat /root/TrafficCop/ports_traffic_config.json | jq
```

Stop monitor processes:

```bash
sudo pkill -f trafficcop.sh
```

Cancel a scheduled shutdown:

```bash
sudo shutdown -c
```

Clear current TC rules:

```bash
sudo tc qdisc del dev $(ip route | awk '/default/ {print $5; exit}') root 2>/dev/null
```

Remove machine rate limit:

```bash
sudo curl -sSL https://raw.githubusercontent.com/Misasasasasaka/TrafficCop/main/remove_traffic_limit.sh | sudo bash
```

## Uninstall

```bash
sudo pkill -f trafficcop.sh 2>/dev/null
sudo pkill -f tg_notifier.sh 2>/dev/null
sudo pkill -f pushplus_notifier.sh 2>/dev/null
sudo pkill -f serverchan_notifier.sh 2>/dev/null
crontab -l 2>/dev/null | grep -v 'TrafficCop\|trafficcop.sh\|tg_notifier.sh\|pushplus_notifier.sh\|serverchan_notifier.sh\|port_traffic_limit.sh' | crontab -
sudo tc qdisc del dev $(ip route | awk '/default/ {print $5; exit}') root 2>/dev/null
sudo shutdown -c 2>/dev/null
sudo rm -rf /root/TrafficCop
```

## FAQ

### Why is traffic still zero after installation?

`vnstat` needs time to collect data after installation.

### Why does per-port traffic differ from provider billing?

Per-port counters only see packets matching iptables rules. Proxy forwarding, retransmission, and protocol overhead can cause differences.

### SSH is too slow after rate limiting. What should I do?

Run the monitor script again and increase the TC speed limit, or temporarily disable machine limiting from the manager.

### How do I change existing configuration?

```bash
sudo bash /root/TrafficCop/trafficcop.sh
sudo bash /root/TrafficCop/port_traffic_limit.sh
```
