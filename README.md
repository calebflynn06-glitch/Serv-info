#!/bin/bash

# Server Information Gathering Script
# Usage: ./server_info.sh [output_file]

OUTPUT_FILE="${1:-server_info_$(hostname)_$(date +%Y%m%d_%H%M%S).txt}"

echo "=================================================="
echo "Server Information Gathering Script"
echo "Starting at: $(date)"
echo "=================================================="
echo ""

# Function to print section headers
print_header() {
    echo ""
    echo "=========================================="
    echo "$1"
    echo "=========================================="
}

# Redirect all output to both console and file
exec > >(tee -a "$OUTPUT_FILE") 2>&1

print_header "BASIC SYSTEM INFORMATION"
echo "Hostname: $(hostname)"
echo "FQDN: $(hostname -f 2>/dev/null || echo 'N/A')"
echo "Uptime: $(uptime)"
echo "Current Date/Time: $(date)"
echo "Timezone: $(timedatectl 2>/dev/null | grep "Time zone" || date +%Z)"

print_header "OPERATING SYSTEM"
if [ -f /etc/os-release ]; then
    cat /etc/os-release
else
    uname -a
fi
echo ""
echo "Kernel Version: $(uname -r)"
echo "Architecture: $(uname -m)"

print_header "CPU INFORMATION"
lscpu 2>/dev/null || cat /proc/cpuinfo | grep -E "model name|processor|cpu cores"
echo ""
echo "CPU Load Average: $(uptime | awk -F'load average:' '{print $2}')"

print_header "MEMORY INFORMATION"
free -h
echo ""
cat /proc/meminfo | grep -E "MemTotal|MemFree|MemAvailable|SwapTotal|SwapFree"

print_header "DISK INFORMATION"
echo "Disk Usage:"
df -h
echo ""
echo "Disk Partitions:"
lsblk 2>/dev/null || fdisk -l
echo ""
echo "Mounted Filesystems:"
mount | column -t

print_header "NETWORK INFORMATION"
echo "Network Interfaces:"
ip addr show 2>/dev/null || ifconfig -a
echo ""
echo "Routing Table:"
ip route 2>/dev/null || route -n
echo ""
echo "DNS Configuration:"
cat /etc/resolv.conf
echo ""
echo "Listening Ports:"
ss -tulpn 2>/dev/null || netstat -tulpn

print_header "RUNNING PROCESSES"
echo "Top 10 CPU Consuming Processes:"
ps aux --sort=-%cpu | head -11
echo ""
echo "Top 10 Memory Consuming Processes:"
ps aux --sort=-%mem | head -11

print_header "LOGGED IN USERS"
who
echo ""
echo "Last Logins:"
last -n 10

print_header "INSTALLED SERVICES"
if command -v systemctl &> /dev/null; then
    systemctl list-units --type=service --state=running
else
    service --status-all 2>/dev/null
fi

print_header "CRON JOBS"
echo "Root Crontab:"
crontab -l 2>/dev/null || echo "No crontab for root"
echo ""
echo "System Cron Jobs:"
ls -la /etc/cron.* 2>/dev/null

print_header "FIREWALL STATUS"
if command -v ufw &> /dev/null; then
    ufw status verbose
elif command -v firewall-cmd &> /dev/null; then
    firewall-cmd --list-all
else
    iptables -L -n -v 2>/dev/null || echo "Firewall information not available"
fi

print_header "SECURITY UPDATES"
if command -v apt &> /dev/null; then
    apt list --upgradable 2>/dev/null | grep -i security || echo "No security updates pending"
elif command -v yum &> /dev/null; then
    yum list updates --security 2>/dev/null || echo "No security updates pending"
fi

print_header "DOCKER INFORMATION (if installed)"
if command -v docker &> /dev/null; then
    docker --version
    echo ""
    docker ps -a
    echo ""
    docker images
else
    echo "Docker not installed"
fi

print_header "INSTALLED PACKAGES (sample)"
if command -v dpkg &> /dev/null; then
    echo "Total packages: $(dpkg -l | grep ^ii | wc -l)"
    echo "Recently installed (last 20):"
    grep " install " /var/log/dpkg.log* 2>/dev/null | tail -20
elif command -v rpm &> /dev/null; then
    echo "Total packages: $(rpm -qa | wc -l)"
    echo "Recently installed (last 20):"
    rpm -qa --last | head -20
fi

print_header "SYSTEM LOGS (last 20 lines of syslog)"
tail -20 /var/log/syslog 2>/dev/null || tail -20 /var/log/messages 2>/dev/null || echo "System logs not accessible"

echo ""
echo "=================================================="
echo "Information gathering completed at: $(date)"
echo "Report saved to: $OUTPUT_FILE"
echo "=================================================="
