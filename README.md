# HomeAssistant on MacOs
将 Docker 容器内 Home Assistant (HASS) 的 HomeKit 服务（Bridge）通过 mDNS (Bonjour) 协议转发/注册到宿主机所在的网络中

## 创建HomeAssistant workspace
```
mkdir /{your_path}/ha
mkdir /{your_path}/ha/config
cd /{your_path}/ha
```

## 创建Dockerfile

```
vi Dockerfile
```

```
FROM homeassistant/home-assistant:stable

# 在docker container中安装 avahi-daemon
# https://gnanesh.me/avahi-docker-non-root.html
RUN set -ex \
  && apk --no-cache --no-progress add avahi avahi-tools dbus   \
#   Disable default Avahi services
  && rm /etc/avahi/services/* \
  && rm -rf /var/cache/apk/*

# docker-entrypoint.sh文件需要和Dockerfile在同目录下
COPY docker-entrypoint.sh /usr/local/sbin/
RUN chmod +x /usr/local/sbin/docker-entrypoint.sh
ENTRYPOINT ["/usr/local/sbin/docker-entrypoint.sh"]
```

## 构建docker镜像
```
docker build -t homeassistant:avahi .
```

## 创建Docker compose
```
vi docker-compose.yaml
```
```
services:
  homeassistant:
    container_name: homeassistant
    image: homeassistant:avahi
    build: .
    volumes:
      - ./config:/config
      - /etc/localtime:/etc/localtime:ro
    restart: unless-stopped
    privileged: true
    #network_mode: host
    ports:
      - 8123:8123
      - 51827:51827
```

## 创建HA configration
可参考官方文档：[HomeKit Bridge](https://www.home-assistant.io/integrations/homekit#include_domains)
```
vi /{your_path}/ha/config
```
```
default_config:
frontend:
  themes: !include_dir_merge_named themes
homekit:
  - name: HASS Bridge
    port: 51827
    advertise_ip: (your_mac_ip_address)
```

## 创建dns-sd脚本
```
# LaunchAgents对路径及权限有所要求
touch dns_sd.err
touch dns_sd.log
# 复制脚本并修改BASE_DIR
vi /{your_path}/ha/dns-sd.sh
```
```
#!/bin/bash

# 配置区域
DNS_SD_NAME="HASS Bridge"
DOCKER_BIN="/usr/local/bin/docker"
DNS_SD_BIN="/usr/bin/dns-sd"
BASE_DIR="/{your_path}/ha"
LOG_FILE="$BASE_DIR/dns-sd.log"

# 确保工作目录存在
mkdir -p "$BASE_DIR"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" >> "$LOG_FILE"
}

run_dns_sd() {
    log "--- 开始新的注册流程 ---"
    
    # 1. 从 Docker 提取数据
    # 改用 awk 动态查找字段：$5=Type, $6=Domain, $9=Port, $10=TXT
    RAW_DATA=$($DOCKER_BIN exec homeassistant avahi-browse -t -r -p -k _hap._tcp 2>/dev/null | grep -m 1 "$DNS_SD_NAME")
    
    if [ -z "$RAW_DATA" ]; then
        log "错误: 无法从 Docker 获取数据，请检查容器是否运行。"
        return 1
    fi

    # 2. 精确解析字段
    TYPE=$(echo "$RAW_DATA" | cut -d';' -f5)
    DOMAIN=$(echo "$RAW_DATA" | cut -d';' -f6)
    PORT=$(echo "$RAW_DATA" | cut -d';' -f9)
    TXT=$(echo "$RAW_DATA" | cut -d';' -f10)
    
    # 验证端口是否为数字
    if ! [[ "$PORT" =~ ^[0-9]+$ ]]; then
        log "错误: 解析到的端口非法 ($PORT)，请检查 Avahi 输出格式。"
        return 1
    fi

    # 3. 清理残留
    pkill -f "dns-sd -R $DNS_SD_NAME"
    sleep 2

    log "正在注册服务: $DNS_SD_NAME (Port: $PORT)"
    log "TXT 内容: $TXT"

    # 4. 执行注册 (阻塞模式)
    $DNS_SD_BIN -R "$DNS_SD_NAME" "$TYPE" "$DOMAIN" "$PORT" $(echo $TXT | sed 's/\"//g') >> "$LOG_FILE" 2>&1
}

run_dns_sd

```

```
chmod +x /{your_path}/ha/*
```

## 创建LaunchAgents
```
# 复制以下内容并修改路径
vi ~/Library/LaunchAgents/com.homeassistant.dns.sd.plist
```

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.homeassistant.dns.sd</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/zsh</string>
        <string>/{your_path}/ha/dns-sd.sh</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/{your_path}/ha/dns_sd.log</string>
    <key>StandardErrorPath</key>
    <string>/{your_path}/ha/dns_sd.err</string>
</dict>
</plist>
```

```
# 验证 Plist 语法
plutil -lint ~/Library/LaunchAgents/com.homeassistant.dns.sd.plist

# 停止LaunchAgents
launchctl unload ~/Library/LaunchAgents/com.homeassistant.dns.sd.plist

# 启动LaunchAgents
launchctl load ~/Library/LaunchAgents/com.homeassistant.dns.sd.plist
                                             
# 查看LaunchAgents
# 如果左侧数字是 PID，说明运行正常；如果是负数或非零错误码，则有问题，可在配置的dns_sd.err中查看日志解决。
launchctl list | grep homeassistant

# 检查是否有HASS Bridge已被成功添加
dns-sd -B _hap._tcp
```
    
## 启动docker
```
docker compose up -d  

# 打开HA页面开始使用，http://127.0.0.1:8123/
```










