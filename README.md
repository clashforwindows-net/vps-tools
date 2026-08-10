# VPS Tools — 服务器诊断与运维工具集

> **不是脚本库，是"工具选型 + 使用手册"。**
> 当服务器出问题时，你需要的不是又一个一键脚本，而是知道**该用哪个工具、怎么读它的输出、下一步查什么**。
> 本仓库整理了 VPS 运维中真正高频的 60+ 个工具，按「排查场景」组织，每个都给出命令、输出解读和判断阈值。

![Linux](https://img.shields.io/badge/Linux-Toolbox-FCC624?logo=linux&logoColor=black)
![Tools](https://img.shields.io/badge/Tools-60%2B-blue)
![License](https://img.shields.io/badge/License-MIT-green)

---

## 目录

- [一、按场景选工具（速查表）](#一按场景选工具速查表)
- [二、工具安装](#二工具安装)
- [三、CPU 与负载诊断](#三cpu-与负载诊断)
- [四、内存诊断](#四内存诊断)
- [五、磁盘与 IO 诊断](#五磁盘与-io-诊断)
- [六、网络诊断工具](#六网络诊断工具)
- [七、流量监控与统计](#七流量监控与统计)
- [八、端口与连接分析](#八端口与连接分析)
- [九、进程管理与追踪](#九进程管理与追踪)
- [十、日志分析工具](#十日志分析工具)
- [十一、批量与远程管理](#十一批量与远程管理)
- [十二、性能火焰图与深度剖析](#十二性能火焰图与深度剖析)
- [十三、USE 方法论：60 秒定位问题](#十三use-方法论60-秒定位问题)
- [十四、真实排查案例](#十四真实排查案例)
- [十五、工具对比与选型建议](#十五工具对比与选型建议)
- [十六、FAQ](#十六faq)
- [十七、相关资源](#十七相关资源)

---

## 一、按场景选工具（速查表）

| 你遇到的现象 | 第一个用的工具 | 第二个 | 第三个 |
|--------------|----------------|--------|--------|
| 网站变慢，不知道哪儿慢 | `uptime` | `vmstat 1` | `pidstat 1` |
| CPU 100% | `top` / `htop` | `pidstat -t 1` | `perf top` |
| 内存被吃光 | `free -h` | `smem -tk` | `dmesg \| grep -i oom` |
| 磁盘 IO 卡顿 | `iostat -x 1` | `iotop -oPa` | `blktrace` |
| 磁盘满了 | `df -h` | `du -x --max-depth=2` | `lsof +L1` |
| 网络丢包 | `mtr` | `ping -f` | `ethtool -S` |
| 带宽跑满 | `iftop` | `nethogs` | `vnstat` |
| 连接数异常 | `ss -s` | `ss -tanp` | `conntrack -L` |
| 端口不通 | `ss -tlnp` | `nc -zv` | `tcpdump` |
| 进程神秘卡死 | `ps -eLo state` | `strace -p` | `cat /proc/PID/stack` |
| 找不到谁在写日志 | `lsof -p PID` | `fuser -v file` | `auditctl` |
| DNS 解析异常 | `dig +trace` | `nslookup` | `resolvectl status` |
| 系统偶发重启 | `journalctl -b -1` | `last -x` | `dmesg -T` |

---

## 二、工具安装

### 一键安装全套（Debian / Ubuntu）

```bash
sudo apt update && sudo apt install -y \
  htop atop glances sysstat dstat \
  iotop iftop nethogs nload bmon vnstat \
  net-tools dnsutils mtr-tiny traceroute tcpdump nmap \
  lsof strace ltrace psmisc procps \
  ncdu tree jq ripgrep fd-find bat \
  smem numactl ethtool conntrack
```

### RHEL 系（Rocky / AlmaLinux / CentOS Stream）

```bash
sudo dnf install -y epel-release
sudo dnf install -y \
  htop atop glances sysstat dstat \
  iotop iftop nethogs nload bmon vnstat \
  bind-utils mtr traceroute tcpdump nmap \
  lsof strace psmisc procps-ng \
  ncdu tree jq ripgrep bat ethtool conntrack-tools
```

### 现代替代品（强烈推荐）

| 传统工具 | 现代替代 | 优势 |
|----------|----------|------|
| `top` | `btop` / `htop` | 交互友好、图形化、鼠标支持 |
| `du` | `ncdu` / `dust` | 交互式浏览，直接定位大目录 |
| `df` | `duf` | 彩色分组，一眼看清挂载点 |
| `netstat` | `ss` | 快 10 倍以上，netstat 已废弃 |
| `ifconfig` | `ip` | net-tools 已停止维护 |
| `grep` | `ripgrep (rg)` | 大日志搜索快 5-10 倍 |
| `cat` | `bat` | 语法高亮、行号 |
| `find` | `fd` | 语法简洁，默认忽略 .git |
| `ps aux \| grep` | `procs` | 结构化输出，带树形 |

```bash
# btop 安装（几乎所有发行版都能装）
sudo snap install btop  # 或 apt/dnf install btop
```

---

## 三、CPU 与负载诊断

### 3.1 读懂 load average

```bash
$ uptime
 22:31:07 up 42 days,  3:12,  2 users,  load average: 4.21, 3.85, 2.10
```

**关键：load 要除以核心数看。**

```bash
echo "cores=$(nproc)  load1=$(awk '{print $1}' /proc/loadavg)"
awk -v c="$(nproc)" '{printf "per-core load: %.2f\n", $1/c}' /proc/loadavg
```

| per-core load | 判断 |
|:-------------:|------|
| < 0.7 | 健康 |
| 0.7 ~ 1.0 | 接近饱和，该关注了 |
| 1.0 ~ 2.0 | 已经排队，用户能感知延迟 |
| > 2.0 | 严重过载 |

⚠️ **Linux 的 load 包含 D 状态（不可中断睡眠）进程**，所以磁盘 IO 慢也会把 load 顶上去，不一定是 CPU 问题。

```bash
# 数一下有多少 D 状态进程
ps -eo state,pid,comm | awk '$1=="D"' | wc -l
```

### 3.2 `vmstat` —— 最值得先看的一条命令

```bash
$ vmstat 1 5
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 5  2      0 182364  95124 892336    0    0    12   428 1820 3210 62 12  8 18  0
```

逐列解读：

| 列 | 含义 | 异常信号 |
|----|------|----------|
| `r` | 运行队列长度 | 持续 > CPU 核数 → CPU 瓶颈 |
| `b` | 阻塞进程数 | 持续 > 0 → IO 瓶颈 |
| `si/so` | swap 换入/换出 | 非 0 → 内存不足，性能雪崩前兆 |
| `bi/bo` | 块设备读/写(KB/s) | 突然飙高 → 定位 IO 源头 |
| `cs` | 上下文切换/秒 | 数万级 → 线程过多或锁竞争 |
| `wa` | IO 等待占比 | > 20% → 磁盘是瓶颈 |
| `st` | 被宿主机偷走的 CPU | > 5% → **VPS 超售严重，换机** |

> 💡 `st`（steal time）是判断一台 VPS 是否被超售的**最直接指标**。如果长期 > 10%，说明宿主机上邻居把 CPU 抢光了，这不是你的锅，换服务商吧。

### 3.3 定位到具体进程/线程

```bash
# 按 CPU 排序的进程
ps -eo pid,ppid,%cpu,%mem,etime,cmd --sort=-%cpu | head -15

# 线程级（-t）视图，找出是哪个线程在烧 CPU
pidstat -t -p <PID> 1 5

# 直接看内核态在干什么
perf top -p <PID>
```

### 3.4 CPU 是 user 高还是 sys 高

```bash
mpstat -P ALL 1 3
```

- **%usr 高**：应用自身计算量大 → 看代码/加缓存/扩容
- **%sys 高**：系统调用频繁 → 大量小 IO、频繁 fork、网络包处理
- **%iowait 高**：等磁盘 → 转去看磁盘章节
- **%soft 高**：软中断，通常是网卡收包 → 检查多队列 / RPS
- **%steal 高**：宿主机超售

---

## 四、内存诊断

### 4.1 `free -h` 的正确读法

```bash
$ free -h
               total        used        free      shared  buff/cache   available
Mem:           3.8Gi       1.9Gi       184Mi        28Mi       1.7Gi       1.6Gi
Swap:          2.0Gi       312Mi       1.7Gi
```

**只看 `available`，不要看 `free`。** `buff/cache` 是可回收的页缓存，Linux 把空闲内存拿去做缓存是好事，不是内存泄漏。

真实可用率：

```bash
free | awk '/^Mem:/ {printf "内存实际使用率: %.1f%%\n", ($2-$7)/$2*100}'
```

### 4.2 谁在吃内存

```bash
# 按 RSS 排序
ps -eo pid,comm,rss,vsz --sort=-rss | head -15 | \
  awk 'NR==1{print;next}{printf "%-8s %-20s %8.1fMB %10.1fMB\n",$1,$2,$3/1024,$4/1024}'

# smem 能算 PSS（按比例分摊共享内存），比 RSS 准确得多
smem -tk -s pss -r | head -15
```

**RSS vs PSS vs USS：**

- `RSS`：进程占用的物理内存，**共享库被重复计算**，多进程服务（如 PHP-FPM）会严重高估
- `PSS`：共享部分按进程数平分，**求和等于系统真实使用量**
- `USS`：只算私有内存，**杀掉这个进程能回收多少**

### 4.3 OOM 排查

```bash
# 有没有被 OOM Killer 干掉过
dmesg -T | grep -i -E 'oom|killed process'
journalctl -k | grep -i oom

# 查看进程的 OOM 评分（越高越容易被杀）
for p in $(pgrep -d' ' -f nginx); do
  echo "$p $(cat /proc/$p/oom_score) $(cat /proc/$p/comm)"
done

# 保护关键进程不被杀（-1000 为完全免疫）
echo -500 > /proc/<PID>/oom_score_adj
```

### 4.4 缓存与 slab

```bash
# 内核 slab 占用（大量 dentry/inode 缓存也会吃光内存）
slabtop -o | head -15

# 手动回收页缓存（生产慎用，会导致短时 IO 飙升）
sync && echo 3 > /proc/sys/vm/drop_caches
```

---

## 五、磁盘与 IO 诊断

### 5.1 `iostat -x` 核心指标

```bash
$ iostat -x 1 3
Device   r/s   w/s   rkB/s   wkB/s  await  r_await  w_await  aqu-sz  %util
vda     2.10 148.30   84.0  9821.4  18.62     1.20    18.86    2.76   96.40
```

| 指标 | 含义 | 阈值 |
|------|------|------|
| `await` | 平均 IO 响应时间(ms) | SSD > 10ms / HDD > 50ms 即异常 |
| `aqu-sz` | 平均队列长度 | > 2 表示排队严重 |
| `%util` | 设备繁忙度 | > 80% 接近饱和（对 NVMe 参考价值低） |
| `r/s w/s` | IOPS | 对比服务商标称值判断是否被限速 |

> ⚠️ 对 NVMe 这种支持高并发的设备，`%util` 100% 不一定代表满载（它能并行处理多个请求）。此时优先看 `await` 和 `aqu-sz`。

### 5.2 找出 IO 大户

```bash
# 实时查看哪个进程在读写磁盘（-o 只显示有 IO 的，-a 累计）
sudo iotop -oPa

# 非交互方式，适合脚本
sudo iotop -boqqn 3 | head -20

# 进程级 IO 统计
pidstat -d 1 5
```

### 5.3 磁盘空间问题

```bash
# 快速定位（交互式，推荐）
ncdu -x /

# 命令行版
du -x -h --max-depth=2 / 2>/dev/null | sort -rh | head -25

# inode 用尽（df 显示有空间但写不进去）
df -i

# 找出小文件最多的目录
for d in /var/*; do echo "$(find "$d" -xdev -type f 2>/dev/null | wc -l) $d"; done | sort -rn | head
```

**"磁盘满了但 du 找不到文件"的经典场景：**

```bash
# 文件被删除但进程还持有句柄，空间不会释放
sudo lsof -nP +L1 | awk 'NR==1 || $5=="REG"'
# 解决：重启持有句柄的进程，或清空文件内容
: > /proc/<PID>/fd/<FD>
```

### 5.4 磁盘性能实测

```bash
# 顺序写
dd if=/dev/zero of=/tmp/test bs=1M count=2048 oflag=direct status=progress

# 随机 4K 读写（更接近数据库真实负载）
fio --name=randrw --ioengine=libaio --direct=1 \
    --rw=randrw --rwmixread=70 --bs=4k --numjobs=4 \
    --size=1G --runtime=60 --group_reporting
```

关注 `iops` 与 `lat(99th percentile)`。廉价 VPS 常见问题是**平均值好看但 P99 抖动巨大**。

---

## 六、网络诊断工具

### 6.1 `mtr` —— 比 ping + traceroute 强得多

```bash
# 报告模式，跑 100 个包
mtr -rwzbc 100 8.8.8.8

# TCP 模式（很多节点屏蔽 ICMP，用 TCP 更真实）
mtr -T -P 443 -rwc 50 example.com
```

输出解读：

```
HOST                        Loss%   Snt   Last   Avg  Best  Wrst StDev
 1. AS?    10.0.0.1          0.0%   100    0.5   0.6   0.4   1.2   0.1
 5. AS4134 219.158.x.x      12.0%   100   45.2  48.1  43.8 210.4  22.3   ← 注意这里
 9. AS15169 8.8.8.8          0.0%   100   52.1  52.4  51.9  55.0   0.6
```

**关键判断规则：**

1. **中间跳丢包但最后一跳不丢 → 忽略**。中间路由器对 ICMP 限速是常态，不代表真丢包。
2. **从某一跳开始一直丢到最后 → 那一跳是真问题点**。
3. **StDev 很大 → 链路抖动**，比丢包更影响体验（视频会议、游戏）。
4. 关注**回程路由**：从对端 mtr 回你的 IP，很多"慢"是回程绕路导致的。

### 6.2 连通性与延迟

```bash
# 高频 ping 测抖动（需要 root）
sudo ping -f -c 500 target

# TCP 端口连通性 + 握手耗时
time nc -zv example.com 443

# 更详细的 HTTP 分段耗时
curl -w '\nDNS: %{time_namelookup}s\nTCP: %{time_connect}s\nTLS: %{time_appconnect}s\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n' \
  -o /dev/null -s https://example.com
```

这条 `curl` 命令能直接告诉你慢在**DNS、TCP 握手、TLS 协商还是服务端处理**。

### 6.3 抓包分析

```bash
# 抓指定端口，不解析域名（-n），显示时间戳
sudo tcpdump -i any -nn port 443 -c 200 -w /tmp/cap.pcap

# 只看 TCP 握手与 RST（排查连接被重置）
sudo tcpdump -i any -nn 'tcp[tcpflags] & (tcp-syn|tcp-rst|tcp-fin) != 0'

# 看 DNS 查询
sudo tcpdump -i any -nn port 53

# 抓包后本地用 Wireshark 分析
scp server:/tmp/cap.pcap .
```

### 6.4 DNS 排查

```bash
# 完整解析链路
dig +trace example.com

# 指定 DNS 服务器对比（判断是否被污染）
dig @8.8.8.8 example.com +short
dig @1.1.1.1 example.com +short
dig @223.5.5.5 example.com +short

# 查看系统实际用的 DNS
resolvectl status         # systemd-resolved
cat /etc/resolv.conf

# 反查 PTR（邮件服务器必须配）
dig -x 1.2.3.4 +short
```

---

## 七、流量监控与统计

### 7.1 实时流量

```bash
# 按连接看（谁在和谁通信，占多少带宽）
sudo iftop -i eth0 -nNP

# 按进程看（哪个程序在跑流量）—— 排查偷跑流量神器
sudo nethogs eth0

# 简洁的总带宽曲线
nload eth0

# 多接口彩色图表
bmon
```

### 7.2 历史流量统计 `vnstat`

```bash
sudo apt install vnstat
sudo systemctl enable --now vnstat

vnstat -d      # 按天
vnstat -m      # 按月（对付流量套餐必备）
vnstat -h      # 按小时
vnstat -t      # Top 10 流量日
vnstat -l      # 实时
vnstat --json  # JSON 输出，方便脚本对接告警
```

**流量超额预警脚本：**

```bash
#!/usr/bin/env bash
LIMIT_GB=1000
used=$(vnstat --json m | jq '.interfaces[0].traffic.month[-1] |
        (.rx + .tx) / 1024 / 1024 / 1024 | floor')
pct=$(( used * 100 / LIMIT_GB ))
(( pct >= 80 )) && echo "⚠️ 本月流量已用 ${used}GB / ${LIMIT_GB}GB (${pct}%)"
```

### 7.3 网卡层面统计

```bash
# 收发包、错误、丢弃
ip -s link show eth0

# 网卡硬件级统计（丢包、CRC 错误）
ethtool -S eth0 | grep -E 'err|drop|discard' | grep -v ': 0$'

# 网卡协商速率（有时被协商到 100M）
ethtool eth0 | grep -E 'Speed|Duplex'
```

---

## 八、端口与连接分析

### 8.1 `ss` 速查（netstat 已过时，别再用了）

```bash
ss -tlnp              # 所有 TCP 监听端口 + 进程
ss -ulnp              # UDP 监听
ss -tanp              # 所有 TCP 连接
ss -s                 # 连接状态汇总
ss -tanp 'sport = :443'          # 只看 443
ss -tanp state established       # 只看已建立
ss -tanp state time-wait | wc -l # TIME_WAIT 数量
ss -tni                          # 显示 RTT、cwnd 等 TCP 内部信息
```

### 8.2 连接数统计

```bash
# 按状态统计
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn

# Top 20 来源 IP（排查 CC 攻击）
ss -tan state established | awk 'NR>1 {print $5}' | \
  cut -d: -f1 | sort | uniq -c | sort -rn | head -20

# 某个 IP 的连接数超过阈值就封
THRESH=100
ss -tan state established | awk 'NR>1{print $5}' | cut -d: -f1 | \
  sort | uniq -c | awk -v t=$THRESH '$1>t {print $2}' | \
  while read -r ip; do echo "候选封禁: $ip"; done
```

### 8.3 TIME_WAIT 过多

```bash
sysctl net.ipv4.tcp_tw_reuse      # 建议 =1
sysctl net.ipv4.ip_local_port_range
```

> ❌ **不要开 `tcp_tw_recycle`**。它在 NAT 环境下会导致大量连接被莫名丢弃，而且新内核已经移除了这个参数。

### 8.4 端口扫描与验证

```bash
# 本机开放了哪些端口给外网（从外部视角）
nmap -Pn -p- your.server.ip

# 快速探测常见端口
nmap -Pn -F your.server.ip

# 检查某端口是否真的通（含 UDP）
nc -zv host 443
nc -zuv host 53
```

---

## 九、进程管理与追踪

### 9.1 进程状态速查

```bash
# 找出所有 D 状态（不可中断，通常卡在 IO）
ps -eo state,pid,ppid,wchan:30,cmd | awk '$1=="D"'

# 看进程卡在内核哪个函数
sudo cat /proc/<PID>/stack

# 僵尸进程
ps -eo state,pid,ppid,cmd | awk '$1=="Z"'

# 进程树
pstree -aps <PID>
```

### 9.2 `strace` / `ltrace`

```bash
# 追踪系统调用（看进程到底卡在哪个 syscall）
sudo strace -f -tt -p <PID>

# 统计各系统调用耗时占比
sudo strace -c -f -p <PID>          # Ctrl+C 后出汇总表

# 只看文件操作
sudo strace -f -e trace=file -p <PID>

# 只看网络
sudo strace -f -e trace=network -p <PID>
```

> ⚠️ `strace` 会让目标进程慢 10-100 倍，**生产环境短暂使用（几秒）即可，不要挂着不管**。

### 9.3 文件句柄

```bash
# 进程打开了哪些文件
sudo lsof -p <PID>

# 谁在占用这个文件/目录（umount 失败时用）
sudo fuser -vm /mnt/data
sudo lsof +D /mnt/data

# 谁占用了这个端口
sudo lsof -i :443

# 句柄数排行（排查 fd 泄漏）
for p in /proc/[0-9]*; do
  echo "$(ls "$p/fd" 2>/dev/null | wc -l) $(cat "$p/comm" 2>/dev/null) ${p##*/}"
done | sort -rn | head -10
```

---

## 十、日志分析工具

### 10.1 journalctl

```bash
journalctl -u nginx -n 100 --no-pager      # 指定服务最近 100 行
journalctl -u nginx -f                     # 实时跟踪
journalctl --since "1 hour ago" -p err     # 最近一小时的错误
journalctl -b -1 -p crit                   # 上次启动的严重日志（排查重启原因）
journalctl -k --since today                # 内核日志
journalctl --disk-usage                    # 日志占了多少空间
journalctl --vacuum-time=7d                # 只保留 7 天
```

### 10.2 Nginx 访问日志快速分析

```bash
LOG=/var/log/nginx/access.log

# Top 20 IP
awk '{print $1}' "$LOG" | sort | uniq -c | sort -rn | head -20

# Top 20 URL
awk '{print $7}' "$LOG" | sort | uniq -c | sort -rn | head -20

# 状态码分布
awk '{print $9}' "$LOG" | sort | uniq -c | sort -rn

# 5xx 错误的请求
awk '$9 ~ /^5/ {print $1, $7, $9}' "$LOG" | sort | uniq -c | sort -rn | head

# 每分钟请求量（找流量尖峰）
awk -F'[:[]' '{print $2":"$3":"$4}' "$LOG" | uniq -c | sort -rn | head

# 慢请求（需在 log_format 中加 $request_time）
awk '$NF > 2 {print $NF, $7}' "$LOG" | sort -rn | head -20
```

### 10.3 大日志用 ripgrep

```bash
# 比 grep 快 5-10 倍
rg -n 'ERROR|FATAL' /var/log/app/*.log

# 统计出现次数
rg -c 'timeout' /var/log/app/app.log

# 带上下文
rg -B2 -A5 'Exception' app.log

# 只搜某段时间
rg '2026-08-10 2[0-3]:' app.log | rg ERROR
```

### 10.4 登录审计

```bash
last -x -n 30                              # 登录历史（含关机重启记录）
lastb -n 30                                # 失败登录
grep 'Failed password' /var/log/auth.log | \
  awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head -20
who -a                                     # 当前在线
w                                          # 在线用户在干什么
```

---

## 十一、批量与远程管理

### 11.1 SSH 配置优化

```
# ~/.ssh/config
Host *
    ServerAliveInterval 30
    ServerAliveCountMax 3
    ControlMaster auto
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 10m
    Compression yes

Host hk1
    HostName 1.2.3.4
    Port 22022
    User ops
    IdentityFile ~/.ssh/id_ed25519
```

`ControlMaster` 复用连接，第二次 ssh 到同一台机器**几乎零延迟**。

### 11.2 并行执行

```bash
# 用 xargs 并行
xargs -a hosts.txt -P 10 -I{} ssh -o BatchMode=yes {} 'uptime; df -h /' 

# 用 GNU parallel（输出不会交错）
parallel -j 10 --tag ssh {} 'uptime' :::: hosts.txt
```

### 11.3 文件分发

```bash
# rsync 增量同步（比 scp 好用太多）
rsync -avz --progress --delete -e 'ssh -p 22022' ./dist/ ops@hk1:/var/www/

# 断点续传大文件
rsync -avzP --partial bigfile.tar.gz ops@hk1:/data/
```

### 11.4 长时间任务不断线

```bash
# tmux（推荐）
tmux new -s deploy
# Ctrl+B D 分离，重连后
tmux attach -t deploy

# 或者 nohup
nohup ./long-task.sh > task.log 2>&1 &
```

---

## 十二、性能火焰图与深度剖析

```bash
# 安装 perf
sudo apt install linux-tools-common linux-tools-$(uname -r)

# 采样 30 秒
sudo perf record -F 99 -a -g -- sleep 30

# 生成报告
sudo perf report --stdio | head -50

# 生成火焰图
git clone https://github.com/brendangregg/FlameGraph
sudo perf script | ./FlameGraph/stackcollapse-perf.pl | \
  ./FlameGraph/flamegraph.pl > flame.svg
```

火焰图看法：**横轴是采样占比（不是时间顺序），越宽的函数越值得优化；纵轴是调用栈深度**。找最宽的那个"平顶"。

---

## 十三、USE 方法论：60 秒定位问题

Brendan Gregg 的 USE 方法：对每个资源检查 **Utilization（使用率）/ Saturation（饱和度）/ Errors（错误）**。

```bash
#!/usr/bin/env bash
# 60 秒快速诊断，把这段存成 quick-diag.sh
echo "===== 1. 系统负载 ====="; uptime
echo "===== 2. 内核错误 ====="; dmesg -T | tail -15
echo "===== 3. 整体资源 ====="; vmstat 1 5
echo "===== 4. 各 CPU 核心 ====="; mpstat -P ALL 1 3
echo "===== 5. 进程 CPU/IO ====="; pidstat 1 3
echo "===== 6. 磁盘 IO ====="; iostat -xz 1 3
echo "===== 7. 内存 ====="; free -h; echo; smem -tk -s pss -r 2>/dev/null | head -8
echo "===== 8. 网络吞吐 ====="; sar -n DEV 1 3 2>/dev/null
echo "===== 9. TCP 状态 ====="; sar -n TCP,ETCP 1 3 2>/dev/null; ss -s
echo "===== 10. 磁盘空间 ====="; df -h; df -i
echo "===== 11. Top 进程 ====="; top -bn1 | head -20
```

| 资源 | Utilization | Saturation | Errors |
|------|-------------|------------|--------|
| CPU | `mpstat` %usr+%sys | `vmstat` r 列 | - |
| 内存 | `free` available | `vmstat` si/so | `dmesg` OOM |
| 磁盘 | `iostat` %util | `iostat` aqu-sz | `dmesg` IO error |
| 网络 | `sar -n DEV` | `ss -s` overflow | `ip -s link` errors |

---

## 十四、真实排查案例

### 案例 1：网站间歇性 502

**现象**：Nginx 每隔十几分钟出现一批 502。

```bash
# 1. 确认是上游挂了还是连接数满了
grep '502' /var/log/nginx/access.log | wc -l
tail -50 /var/log/nginx/error.log
# → "connect() failed (111: Connection refused) while connecting to upstream"

# 2. 看上游进程有没有重启
systemctl status myapp
journalctl -u myapp --since "1 hour ago" | grep -i 'start\|kill'
# → 发现被 OOM Killer 反复杀掉

# 3. 确认 OOM
dmesg -T | grep -i 'killed process'
# → Out of memory: Killed process 2891 (myapp) total-vm:3.9GB
```

**结论**：内存泄漏 + 无 swap。临时方案加 2G swap 顶住，根本方案修代码 + 加内存限制。

### 案例 2：SSH 登录要等 30 秒

```bash
ssh -vvv user@host 2>&1 | grep -i 'debug1' | tail -20
# → 卡在 "pledge: network" 之后
```

**原因**：`sshd_config` 里 `UseDNS yes`，服务器反查客户端 IP 的 PTR 超时。

```bash
sed -i 's/^#\?UseDNS.*/UseDNS no/' /etc/ssh/sshd_config
sed -i 's/^#\?GSSAPIAuthentication.*/GSSAPIAuthentication no/' /etc/ssh/sshd_config
systemctl reload sshd
```

### 案例 3：带宽跑满但不知道谁在跑

```bash
sudo nethogs eth0        # → 发现是某个容器
docker stats --no-stream
sudo iftop -i eth0 -nNP  # → 目标 IP 全是某个境外仓库
```

**结论**：CI 任务在无限重试拉镜像。加上 `--bwlimit` 和重试上限解决。

### 案例 4：磁盘满，但 du 只算出一半

```bash
df -h /            # 98%
du -sh /* | sort -rh | head   # 加起来才 20G，盘是 50G
sudo lsof -nP +L1 | head       # → nginx 持有一个已删除的 25G error.log
```

**解决**：`: > /proc/$(pgrep -f 'nginx: master')/fd/N` 或直接 `systemctl reload nginx`。以后用 logrotate 的 `copytruncate` 或 `postrotate` 发信号。

---

## 十五、工具对比与选型建议

### 监控面板类

| 工具 | 适用场景 | 资源占用 | 推荐度 |
|------|----------|:--------:|:------:|
| `htop` | 日常看进程 | 极低 | ⭐⭐⭐⭐⭐ |
| `btop` | 好看+全面 | 低 | ⭐⭐⭐⭐⭐ |
| `glances` | 一屏看全部（含容器） | 中 | ⭐⭐⭐⭐ |
| `atop` | **可回溯历史**（关键） | 低 | ⭐⭐⭐⭐⭐ |
| `netdata` | 秒级 Web 面板 | 较高 | ⭐⭐⭐⭐ |

> 💡 **`atop` 被严重低估**。它会把历史快照存到 `/var/log/atop/`，事后可以 `atop -r /var/log/atop/atop_20260810 -b 03:00` 回放当时的现场。半夜出问题第二天还能查，这是 `top` 做不到的。

```bash
sudo apt install atop && sudo systemctl enable --now atop
atop -r /var/log/atop/atop_$(date +%Y%m%d) -b 14:00 -e 15:00
```

### 流量工具

| 工具 | 维度 | 用途 |
|------|------|------|
| `iftop` | 连接（IP 对） | 谁在和谁通信 |
| `nethogs` | 进程 | 哪个程序在跑流量 |
| `vnstat` | 接口/时间 | 月流量统计、超额预警 |
| `nload` | 接口 | 简单看总带宽 |
| `sar -n DEV` | 接口/历史 | 事后回溯 |

---

## 十六、FAQ

**Q1：`top` 里 CPU 显示 400%，是不是坏了？**
不是。`top` 默认按单核 100% 计，4 核满载就是 400%。按 `Shift+I` 可切换为 Irix/Solaris 模式。

**Q2：为什么 `netstat` 在新系统上没有了？**
`net-tools` 已停止维护，被 `iproute2` 取代。`netstat` → `ss`，`ifconfig` → `ip addr`，`route` → `ip route`，`arp` → `ip neigh`。

**Q3：`%steal` 多少算超售？**
持续 > 5% 就该警惕，> 10% 基本可以确认宿主机超售严重。这时候优化自己的代码没用，换机器最实际。可以到 [vpsvip.net](https://vpsvip.net) 看看实测数据再选。

**Q4：抓包会影响性能吗？**
`tcpdump` 本身开销不大，但**不加过滤条件抓全部流量**会产生巨大的磁盘写入。务必加端口/主机过滤和 `-c` 包数限制。

**Q5：`strace` 挂上去进程就卡死了？**
`strace` 会显著拖慢进程。生产环境用 `-c` 采样几秒统计就好，或改用开销更低的 `perf` / eBPF 工具（`bpftrace`）。

**Q6：怎么持久化这些指标做趋势分析？**
命令行工具只能看瞬时值。要看趋势请上 Prometheus + Grafana，见 [server-monitoring](https://github.com/clashforwindows-net/server-monitoring)。

**Q7：容器里这些工具都用不了怎么办？**
在宿主机上用 `nsenter` 进入容器的命名空间：
```bash
PID=$(docker inspect -f '{{.State.Pid}}' mycontainer)
sudo nsenter -t $PID -n ss -tanp     # 用宿主机的 ss 看容器网络
```

**Q8：有没有一条命令搞定所有检查？**
把第十三节的 `quick-diag.sh` 存下来，出问题先跑一遍，输出发到群里比截图有用得多。

---

## 十七、相关资源

### 快速开始

```bash
git clone https://github.com/clashforwindows-net/vps-tools.git
cd vps-tools
chmod +x *.sh
./install-toolbox.sh        # 一键安装本文提到的全部工具
./quick-diag.sh             # 60 秒系统体检
```

### 本组织相关仓库

| 仓库 | 说明 |
|------|------|
| [vps-scripts](https://github.com/clashforwindows-net/vps-scripts) | VPS 自动化运维脚本库 |
| [server-monitoring](https://github.com/clashforwindows-net/server-monitoring) | Prometheus + Grafana 监控方案 |
| [vps-bench-20260327](https://github.com/clashforwindows-net/vps-bench-20260327) | VPS 性能测试与评分 |
| [vps-security-20260327](https://github.com/clashforwindows-net/vps-security-20260327) | 安全加固完全手册 |
| [linux-server-20260402](https://github.com/clashforwindows-net/linux-server-20260402) | Linux 运维实战手册 |

### 服务器选购与网络资源

诊断工具再强，也救不了一台被超售的机器。选机时优先看 `%steal`、磁盘 P99 延迟、回程路由质量：

- 🖥️ **VPS 实测评测与推荐**：[https://vpsvip.net](https://vpsvip.net)
- 🚀 **网络加速与代理方案**：[ClashVIP](https://clashvip.net)
- 🧭 **常用工具资源导航**：[nav.clashvip.net](https://nav.clashvip.net)

### 关键词

`Linux运维工具` `服务器诊断` `性能排查` `htop` `iostat` `iftop` `nethogs` `vnstat` `mtr` `tcpdump` `strace` `perf火焰图` `ss命令` `lsof` `日志分析` `USE方法论` `atop历史回溯` `VPS监控`

---

## Contributing

欢迎补充工具与排查案例。提交前请确保：命令在 Debian 12 / Ubuntu 22.04 / Rocky 9 上均可执行，并附上真实输出示例。见 [CONTRIBUTING.md](CONTRIBUTING.md)。

**License**: MIT

---

*最后更新: 2026-08-10 | 维护者: clashforwindows-net | 赞助商: [vpsvip.net](https://vpsvip.net)*
