# 更新服务器 DNS

> 由于众所周知的原因，国内的云服务器经常出现访问不了github的情况.通过shell脚本实现定时更新的功能
```shell
#!/usr/bin/bash

# ==================== 配置参数（根据需求修改）====================
# 远程 Hosts 文件的 URL（替换为你的实际远程地址）
REMOTE_HOSTS_URL="https://gitlab.com/ineo6/hosts/-/raw/master/next-hosts"
# 更新间隔（秒），默认 1 小时（3600 秒）
UPDATE_INTERVAL=3600
# 日志文件路径
LOG_FILE="/var/log/github_hosts_updater.log"
# 备份文件存放目录（自动创建）
BACKUP_DIR="/var/backups/hosts"
# 标记：用于识别脚本自动添加的内容（避免覆盖手动配置）
START_MARKER="# === GitHub Hosts Auto-Update Start === #"
END_MARKER="# === GitHub Hosts Auto-Update End === #"
# ==================== 配置结束 ====================

# 初始化：创建备份目录、日志文件
init() {
    # 创建备份目录（若不存在）
    if [ ! -d "$BACKUP_DIR" ]; then
        mkdir -p "$BACKUP_DIR"
        chmod 700 "$BACKUP_DIR"
    fi
    # 创建日志文件（若不存在）
    if [ ! -f "$LOG_FILE" ]; then
        touch "$LOG_FILE"
        chmod 600 "$LOG_FILE"
    fi
    # 检查 curl 是否安装（用于获取远程文件）
    if ! command -v curl &> /dev/null; then
        log "错误：未安装 curl，请先执行 'sudo apt install curl'（Linux）或 'brew install curl'（macOS）"
        exit 1
    fi
}

# 日志函数：输出时间+内容到日志文件和控制台
log() {
    local msg="$1"
    local time=$(date "+%Y-%m-%d %H:%M:%S")
    echo "[$time] $msg" | tee -a "$LOG_FILE"
}

# 备份当前 hosts 文件（带时间戳，避免覆盖）
backup_hosts() {
    local timestamp=$(date "+%Y%m%d%H%M%S")
    local backup_path="$BACKUP_DIR/hosts_$timestamp.bak"
    # 复制 hosts 到备份目录
    if cp /etc/hosts "$backup_path"; then
        log "已备份 hosts 到：$backup_path"
    else
        log "警告：备份 hosts 失败！"
    fi
}

# 从远程 URL 获取最新 Hosts 内容
fetch_remote_hosts() {
    log "正在从远程获取 Hosts：$REMOTE_HOSTS_URL"
    # 使用 curl 获取远程内容（超时 10 秒，失败重试 2 次）
    local remote_content=$(curl -s --connect-timeout 10 --retry 2 "$REMOTE_HOSTS_URL")
    # 检查获取结果（为空则失败）
    if [ -z "$remote_content" ]; then
        log "错误：获取远程 Hosts 失败（URL 无效或网络异常）"
        return 1
    fi
    echo "$remote_content"
    return 0
}

# 更新本地 hosts 文件（保留手动配置，替换脚本添加的部分）
update_hosts() {
    local new_hosts="$1"
    # 1. 先备份当前 hosts
    backup_hosts

    # 2. 读取现有 hosts 内容，提取「非脚本添加」的部分
    # 若存在标记，则保留标记外的内容；若不存在标记，则保留全部内容
    if grep -q "$START_MARKER" /etc/hosts && grep -q "$END_MARKER" /etc/hosts; then
        # 删除标记及其间内容，只保留手动配置部分
        local pre_content=$(sed "/$START_MARKER/,/$END_MARKER/d" /etc/hosts)
    else
        # 无标记，保留全部内容（并提示首次添加）
        local pre_content=$(cat /etc/hosts)
        log "首次更新：未发现标记，将在 hosts 末尾添加自动配置"
    fi

    # 3. 构建新的 hosts 内容（原有内容 + 新 Hosts + 标记）
    local new_content="$pre_content
$START_MARKER
# 更新时间：$(date "+%Y-%m-%d %H:%M:%S")
$new_hosts
$END_MARKER
"

    # 4. 写入新内容到 hosts（需要 root 权限）
    if echo "$new_content" > /etc/hosts; then
        log "hosts 更新成功！"
        # 刷新 DNS 缓存（不同系统命令不同）
        if [ "$(uname -s)" = "Darwin" ]; then
            # macOS 刷新 DNS
            dscacheutil -flushcache && killall -HUP mDNSResponder
            log "已刷新 macOS DNS 缓存"
        else
            # Linux 刷新 DNS（适用于大多数系统）
            systemctl restart systemd-resolved 2>/dev/null || /etc/init.d/nscd restart 2>/dev/null
            log "已刷新 Linux DNS 缓存"
        fi
    else
        log "错误：写入 hosts 失败（请确保以 root 权限运行脚本）"
        return 1
    fi
}

# 主循环：定时执行更新
main() {
    init
    log "=== GitHub Hosts 自动更新脚本启动 ==="
    log "更新间隔：$((UPDATE_INTERVAL/3600)) 小时 | 远程 URL：$REMOTE_HOSTS_URL"

    while true; do
        log "=== 开始新一轮更新 ==="
        # 获取远程 Hosts
        remote_hosts=$(fetch_remote_hosts)
        if [ $? -eq 0 ]; then
            # 更新本地 hosts
            update_hosts "$remote_hosts"
        fi
        # 等待下一次更新
        log "等待 $((UPDATE_INTERVAL/3600)) 小时后再次更新...\n"
        sleep "$UPDATE_INTERVAL"
    done
}

# 启动主程序
main
```

然后
```bash
nohup github_hosts_updater.sh &  #保持后台运行就可以了
```

