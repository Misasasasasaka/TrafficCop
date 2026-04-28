# TrafficCop - 流量监控与限速工具

[English](README_EN.md) | 中文

TrafficCop 是一组面向 Linux VPS 的 Shell 脚本，用于监控机器总流量、按端口统计流量，并在超出配额前自动限速、阻断或关机。它适合有月度/季度/年度流量额度的服务器，也适合需要为代理、Web、SSH 等端口单独设置流量上限的场景。

## 功能概览

- 机器总流量监控：基于 `vnstat` 统计网卡流量。
- 多种统计模式：`out` 出站、`in` 入站、`total` 入站+出站、`max` 入站/出站取大。
- 自定义周期：支持月度、季度、年度周期和 1-31 日起始日。
- 两种限制方式：`tc` 限速模式、`shutdown` 关机/阻断模式。
- 端口流量限制：为多个端口分别配置流量上限、周期和限制方式。
- 通知推送：支持 Telegram、PushPlus、Server酱。
- 交互式管理器：集中安装、配置、查看日志、更新和停止服务。

## 重要说明

1. `vnstat` 只会统计安装并启动后的流量，无法回溯安装前的历史流量。
2. TC 限速不能阻止 DDoS 或上游计费继续消耗流量，只能降低正常业务出口速度。
3. 端口流量统计基于 iptables 计数器，代理场景下端口总流量为估算值：入站实际测量，出站按入站估算，总量约为入站的 2 倍。
4. 脚本默认安装到 `/root/TrafficCop`，建议使用 root 或 sudo 执行。
5. `shutdown`/阻断模式会中断服务，首次使用建议优先选择 TC 模式。
6. 端口配置采用当前版本 JSON 字段格式；如果从旧版本升级，建议先备份并重新配置端口限制。

## 快速开始

推荐使用交互式管理器，一条命令即可安装和管理所有功能：

```bash
bash <(curl -sL https://raw.githubusercontent.com/Misasasasasaka/TrafficCop/main/trafficcop-manager.sh)
```

管理器菜单包含：

| 选项 | 功能 |
| --- | --- |
| 1 | 安装/管理机器总流量监控 |
| 2 | 安装/管理 Telegram 通知 |
| 3 | 安装/管理 PushPlus 通知 |
| 4 | 安装/管理 Server酱通知 |
| 5 | 安装/管理端口流量限制 |
| 6 | 查看端口流量状态 |
| 7 | 机器限速启用/禁用/恢复 |
| 8 | 查看日志 |
| 9 | 查看当前配置 |
| 10 | 使用预设配置 |
| 11 | 停止所有 TrafficCop 服务 |
| 12 | 更新所有脚本到最新版本 |

## 教程一：配置机器总流量监控

### 1. 启动管理器

```bash
bash <(curl -sL https://raw.githubusercontent.com/Misasasasasaka/TrafficCop/main/trafficcop-manager.sh)
```

选择 `1) 安装/管理流量监控`。

### 2. 选择统计网卡

脚本会自动检测默认网卡。常见名称包括：

- `eth0`
- `ens3`
- `ens4`
- `enp1s0`

如果自动检测正确，直接回车即可；否则输入实际网卡名。

### 3. 选择统计模式

| 模式 | 说明 | 适用场景 |
| --- | --- | --- |
| `out` | 只统计出站 | 云厂商只计算出站流量 |
| `in` | 只统计入站 | 少见，按入站计费时使用 |
| `total` | 入站+出站 | 双向流量都计费 |
| `max` | 入站/出站取较大值 | 部分服务商按较大方向计费 |

### 4. 设置周期和额度

按提示设置：

- 统计周期：`m` 月度、`q` 季度、`y` 年度。
- 周期起始日：1-31，例如账单日是 15 号就填 `15`。
- 流量限制：单位 GB，例如 `200`。
- 容错范围：单位 GB，例如限制 200GB、容错 10GB，则达到 190GB 开始处理。

### 5. 选择限制方式

| 模式 | 行为 | 建议 |
| --- | --- | --- |
| TC 模式 | 超限后给主网卡限速 | 推荐，SSH 通常仍可连接 |
| 关机模式 | 超限后计划关机 | 谨慎使用，适合必须硬停止计费风险的场景 |

TC 模式还需要设置限速值，单位为 `kbit/s`，默认 `20`。

### 6. 确认定时任务

配置完成后，脚本会写入 cron，每分钟自动检查一次：

```bash
crontab -l
```

机器监控日志：

```bash
sudo tail -f -n 50 /root/TrafficCop/traffic_monitor.log
```

## 教程二：配置通知推送

通知脚本会读取机器流量日志，并在状态变化或每日固定时间发送消息。

### Telegram

通过管理器选择 `2) 安装/管理Telegram通知`，按提示填写：

- Bot Token：从 BotFather 创建机器人后获得。
- Chat ID：可通过 `https://api.telegram.org/bot<你的BOT_TOKEN>/getUpdates` 获取。
- Topic ID：群组话题 ID，对应 Telegram Bot API 的 `message_thread_id`。
- 机器名称：用于区分不同服务器。
- 每日报告时间：格式 `HH:MM`，例如 `08:30`。

常用命令：

```bash
sudo tail -f -n 50 /root/TrafficCop/tg_notifier_cron.log
sudo cat /root/TrafficCop/tg_notifier_config.txt
```

### PushPlus

通过管理器选择 `3) 安装/管理PushPlus通知`，按提示填写：

- PushPlus Token
- 机器名称
- 每日报告时间

常用命令：

```bash
sudo tail -f -n 50 /root/TrafficCop/pushplus_notifier_cron.log
sudo cat /root/TrafficCop/pushplus_notifier_config.txt
```

### Server酱

通过管理器选择 `4) 安装/管理Server酱通知`，按提示填写：

- SendKey
- 机器名称
- 每日报告时间

常用命令：

```bash
sudo tail -f -n 50 /root/TrafficCop/serverchan_notifier_cron.log
sudo cat /root/TrafficCop/serverchan_notifier_config.txt
```

## 教程三：配置端口流量限制

端口流量限制适合给特定服务单独设置配额，例如：

- 代理端口
- Web 端口 `80`/`443`
- SSH 端口
- 游戏或下载服务端口

### 1. 安装端口限制功能

通过管理器选择：

```text
5) 安装/管理端口流量限制
```

也可以直接安装：

```bash
sudo mkdir -p /root/TrafficCop
sudo curl -fsSL https://raw.githubusercontent.com/Misasasasasaka/TrafficCop/main/port_traffic_limit.sh -o /root/TrafficCop/port_traffic_limit.sh
sudo curl -fsSL https://raw.githubusercontent.com/Misasasasasaka/TrafficCop/main/view_port_traffic.sh -o /root/TrafficCop/view_port_traffic.sh
sudo chmod +x /root/TrafficCop/port_traffic_limit.sh /root/TrafficCop/view_port_traffic.sh
sudo bash /root/TrafficCop/port_traffic_limit.sh
```

### 2. 添加端口

在端口管理菜单中选择 `1) 添加端口配置`，按提示填写：

| 配置项 | 说明 |
| --- | --- |
| 端口号 | 1-65535，例如 `443` |
| 描述 | 用于展示和通知，例如 `HTTPS` |
| 流量限制 | 单位 GB |
| 容错范围 | 单位 GB |
| 配置方式 | 可同步机器总流量配置，也可自定义 |
| 限制模式 | TC 限速或阻断 |

端口配置会保存到：

```bash
/root/TrafficCop/ports_traffic_config.json
```

### 3. 查看端口流量

```bash
sudo bash /root/TrafficCop/view_port_traffic.sh
```

实时刷新：

```bash
sudo bash /root/TrafficCop/view_port_traffic.sh --realtime
```

输出 JSON：

```bash
sudo bash /root/TrafficCop/view_port_traffic.sh --json
```

导出报告：

```bash
sudo bash /root/TrafficCop/view_port_traffic.sh --export
```

### 4. 删除端口限制

删除指定端口：

```bash
sudo bash /root/TrafficCop/port_traffic_limit.sh --remove 443
```

删除全部端口配置和限制：

```bash
sudo bash /root/TrafficCop/port_traffic_limit.sh --remove
```

## 预设配置

预设配置会下载到 `/root/TrafficCop/traffic_monitor_config.txt`。下载后重新运行机器监控脚本即可应用。

```bash
sudo mkdir -p /root/TrafficCop
```

| 预设 | 命令 |
| --- | --- |
| 阿里云 CDT 20G | `sudo curl -fsSL -o /root/TrafficCop/traffic_monitor_config.txt https://raw.githubusercontent.com/Misasasasasaka/TrafficCop/main/ali-20g` |
| 阿里云 CDT 200G | `sudo curl -fsSL -o /root/TrafficCop/traffic_monitor_config.txt https://raw.githubusercontent.com/Misasasasasaka/TrafficCop/main/ali-200g` |
| 阿里云轻量 1T | `sudo curl -fsSL -o /root/TrafficCop/traffic_monitor_config.txt https://raw.githubusercontent.com/Misasasasasaka/TrafficCop/main/ali-1T` |
| Azure 学生 15G | `sudo curl -fsSL -o /root/TrafficCop/traffic_monitor_config.txt https://raw.githubusercontent.com/Misasasasasaka/TrafficCop/main/az-15g` |
| Azure 学生 115G | `sudo curl -fsSL -o /root/TrafficCop/traffic_monitor_config.txt https://raw.githubusercontent.com/Misasasasasaka/TrafficCop/main/az-115g` |
| GCP 200G | `sudo curl -fsSL -o /root/TrafficCop/traffic_monitor_config.txt https://raw.githubusercontent.com/Misasasasasaka/TrafficCop/main/gcp-200g` |
| GCP 625G | `sudo curl -fsSL -o /root/TrafficCop/traffic_monitor_config.txt https://raw.githubusercontent.com/Misasasasasaka/TrafficCop/main/gcp-625g` |
| Alice 1500G | `sudo curl -fsSL -o /root/TrafficCop/traffic_monitor_config.txt https://raw.githubusercontent.com/Misasasasasaka/TrafficCop/main/alice-1500g` |
| 亚洲云 300G | `sudo curl -fsSL -o /root/TrafficCop/traffic_monitor_config.txt https://raw.githubusercontent.com/Misasasasasaka/TrafficCop/main/asia-300g` |

查看配置：

```bash
sudo cat /root/TrafficCop/traffic_monitor_config.txt
```

应用配置：

如果还没有安装主脚本，请先执行“快速开始”中的管理器命令。已经安装过 TrafficCop 后运行：

```bash
sudo bash /root/TrafficCop/trafficcop.sh
```

## 常用维护命令

查看机器监控日志：

```bash
sudo tail -f -n 50 /root/TrafficCop/traffic_monitor.log
```

查看端口监控日志：

```bash
sudo tail -f -n 50 /root/TrafficCop/port_traffic_monitor.log
```

查看机器配置：

```bash
sudo cat /root/TrafficCop/traffic_monitor_config.txt
```

查看端口配置：

```bash
sudo cat /root/TrafficCop/ports_traffic_config.json | jq
```

停止机器监控进程：

```bash
sudo pkill -f trafficcop.sh
```

取消可能存在的关机计划：

```bash
sudo shutdown -c
```

清除当前网卡 TC 规则：

```bash
sudo tc qdisc del dev $(ip route | awk '/default/ {print $5; exit}') root 2>/dev/null
```

一键解除机器限速：

```bash
sudo curl -sSL https://raw.githubusercontent.com/Misasasasasaka/TrafficCop/main/remove_traffic_limit.sh | sudo bash
```

## 卸载

停止进程、清理 cron、删除工作目录：

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

### 为什么刚安装后流量是 0？

`vnstat` 从安装后才开始统计，需要等待一段时间产生数据。

### 为什么端口流量和服务商后台不完全一致？

端口统计基于 iptables 规则，只能看到匹配端口的数据包。代理类服务会存在转发、重传、协议开销等差异，因此端口总量是估算值。

### SSH 被限速后很慢怎么办？

重新运行机器监控脚本，提高 TC 限速值；或者使用机器限速管理器临时禁用限速。

### 如何修改已有配置？

再次运行对应脚本即可进入交互式菜单：

```bash
sudo bash /root/TrafficCop/trafficcop.sh
sudo bash /root/TrafficCop/port_traffic_limit.sh
```

### 去哪里看错误日志？

优先查看：

```bash
sudo tail -n 100 /root/TrafficCop/traffic_monitor.log
sudo tail -n 100 /root/TrafficCop/port_traffic_monitor.log
```
