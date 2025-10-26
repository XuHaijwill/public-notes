# 如何配置InfluxDB OSS版本的高可用架构

> InfluxDB OSS 版本本身不提供原生高可用（HA）机制，但可以通过 **“数据冗余 + 故障转移 + 负载均衡”** 的组合方案实现高可用架构，核心目标是确保数据不丢失、服务持续可用。以下是可落地的详细配置方案：
>
> ### 一、架构设计（双节点高可用）
>
> 推荐采用 **“主从热备 + 数据双向同步 + 反向代理 + 定时备份”** 架构，适用于中小规模场景，兼顾可靠性和易维护性：
>
> - **主节点**：处理主要写入 / 查询请求，作为默认活跃节点；
> - **从节点**：实时同步主节点数据，主节点故障时自动接管服务；
> - **Telegraf**：负责主从节点数据实时双向同步（避免单节点写入瓶颈）；
> - **Nginx/HAProxy**：统一访问入口，实现故障自动切换和查询负载均衡；
> - **定时备份**：定期全量备份数据，作为数据一致性的兜底保障。
>
> ### 二、分步配置流程
>
> #### 1. 环境准备
>
> - 部署 2 台服务器（硬件配置一致），均安装
>
>    
>
>   InfluxDB OSS 2.x
>
>   （版本需相同）；
>
>   ```bash
>   # 以 Ubuntu 为例安装 InfluxDB 2.x
>   wget https://dl.influxdata.com/influxdb/releases/influxdb2_2.7.5-1_amd64.deb
>   sudo dpkg -i influxdb2_2.7.5-1_amd64.deb
>   sudo systemctl start influxdb && sudo systemctl enable influxdb
>   ```
>
>   
>
> - 安装
>
>    
>
>   Telegraf
>
>   （数据同步工具）和
>
>   Nginx
>
>   （反向代理）：
>
>   ```bash
>   sudo apt install telegraf nginx -y
>   ```
>
>   
>
> - 确保节点间网络互通（开放 8086 端口，InfluxDB 通信端口）。
>
> #### 2. 初始化 InfluxDB 主从节点
>
> 在主、从节点分别完成初始化（创建组织、桶、管理员 Token）：
>
> ```bash
> # 主节点初始化（从节点操作相同，仅 IP 不同）
> influx setup \
>   --username admin \
>   --password "YourStrongPassword" \
>   --org myorg \
>   --bucket mybucket \
>   --retention 72h \
>   --force
> ```
>
> - 记录主从节点的 **Token**（在 `~/.influxdbv2/configs` 中查看）和 **IP 地址**。
>
> #### 3. 配置数据实时同步（核心步骤）
>
> 通过 Telegraf 实现主从节点数据双向同步（写入任一节点时，自动同步到另一节点），避免数据孤岛。
>
> ##### （1）在主节点配置 Telegraf 同步到从节点
>
> 编辑 Telegraf 配置文件 `sudo vim /etc/telegraf/telegraf.conf`，添加输出到从节点的配置：
>
> ```toml
> # 采集主节点本地数据（作为输入）
> [[inputs.influxdb_v2_listener]]
>   service_address = ":8186"  # 监听本地 8186 端口，接收主节点写入的数据
>   token = "主节点的Token"    # 验证写入权限
>   organization = "myorg"
>   bucket = "mybucket"
> 
> # 输出到从节点（同步数据）
> [[outputs.influxdb_v2]]
>   urls = ["http://从节点IP:8086"]
>   token = "从节点的Token"
>   organization = "myorg"
>   bucket = "mybucket"
>   timeout = "10s"
> ```
>
> ##### （2）在从节点配置 Telegraf 同步到主节点
>
> 同样编辑从节点的 `telegraf.conf`，实现反向同步：
>
> ```toml
> # 采集从节点本地数据
> [[inputs.influxdb_v2_listener]]
>   service_address = ":8186"
>   token = "从节点的Token"
>   organization = "myorg"
>   bucket = "mybucket"
> 
> # 输出到主节点（同步数据）
> [[outputs.influxdb_v2]]
>   urls = ["http://主节点IP:8086"]
>   token = "主节点的Token"
>   organization = "myorg"
>   bucket = "mybucket"
>   timeout = "10s"
> ```
>
> ##### （3）验证数据同步
>
> 重启主从节点的 Telegraf 服务，测试数据同步：
>
> ```bash
> # 重启 Telegraf
> sudo systemctl restart telegraf && sudo systemctl enable telegraf
> 
> # 向主节点写入测试数据
> influx write --org myorg --bucket mybucket --url http://主节点IP:8086 --token 主节点Token "ha_test value=100"
> 
> # 在从节点查询，确认数据同步成功
> influx query 'from(bucket:"mybucket") |> range(start:-1m)' --org myorg --url http://从节点IP:8086 --token 从节点Token
> ```
>
> #### 4. 配置 Nginx 反向代理（故障切换）
>
> 通过 Nginx 实现请求自动转发，主节点故障时切换到从节点，客户端无需感知节点变化。
>
> 编辑 Nginx 配置文件 `sudo vim /etc/nginx/nginx.conf`，添加以下内容：
>
> ```nginx
> http {
>   # 定义 InfluxDB 节点组
>   upstream influxdb_ha {
>     server 主节点IP:8086 weight=8 max_fails=2 fail_timeout=5s;  # 主节点权重更高，允许2次失败后标记为不可用
>     server 从节点IP:8086 weight=2 backup;  # backup：仅主节点故障时启用
>   }
> 
>   server {
>     listen 8086;  # 与 InfluxDB 默认端口一致，客户端直接连接此端口
>     server_name _;
> 
>     location / {
>       proxy_pass http://influxdb_ha;
>       proxy_set_header Host $host;
>       proxy_set_header X-Real-IP $remote_addr;
>       proxy_connect_timeout 3s;  # 连接超时时间
>       proxy_read_timeout 10s;    # 读取超时时间
>     }
>   }
> }
> ```
>
> 重启 Nginx 并验证：
>
> ```bash
> sudo nginx -t  # 检查配置是否有误
> sudo systemctl restart nginx && sudo systemctl enable nginx
> 
> # 测试通过 Nginx 访问（客户端只需连接 Nginx 服务器的 8086 端口）
> influx write --org myorg --bucket mybucket --url http://Nginx服务器IP:8086 --token 主节点Token "nginx_test value=200"
> ```
>
> #### 5. 配置定时备份（数据兜底）
>
> 定期备份主节点数据，防止同步工具故障导致数据丢失：
>
> ##### （1）创建备份脚本 `influx_backup.sh`
>
> ```bash
> #!/bin/bash
> BACKUP_DIR="/data/influxdb_backup"
> TIMESTAMP=$(date +%Y%m%d_%H%M%S)
> ORG="myorg"
> BUCKET="mybucket"
> TOKEN="主节点Token"
> URL="http://主节点IP:8086"
> 
> # 创建备份目录
> mkdir -p $BACKUP_DIR/$TIMESTAMP
> 
> # 执行备份（InfluxDB 2.x 命令）
> influx backup --org $ORG --bucket $BUCKET --token $TOKEN --url $URL $BACKUP_DIR/$TIMESTAMP
> 
> # 保留最近14天的备份
> find $BACKUP_DIR -type d -mtime +14 -delete
> ```
>
> ##### （2）添加定时任务（每天凌晨 2 点执行）
>
> ```bash
> chmod +x influx_backup.sh
> crontab -e
> # 添加以下内容
> 0 2 * * * /path/to/influx_backup.sh
> ```
>
> ### 三、高可用验证与故障恢复
>
> 1. **故障切换测试**：
>
>    - 手动停止主节点的 InfluxDB 服务：`sudo systemctl stop influxdb`；
>    - 通过 Nginx 写入 / 查询数据，确认请求自动切换到从节点；
>    - 恢复主节点服务后，数据会通过 Telegraf 自动同步回主节点。
>
> 2. **数据恢复**：
>
>    - 若从节点数据丢失，可通过备份恢复：
>
>      ```bash
>      # 从备份恢复到从节点
>      influx restore --org myorg --bucket mybucket --token 从节点Token --url http://从节点IP:8086 /data/influxdb_backup/备份目录
>      ```
>
> ### 四、注意事项
>
> 1. **数据一致性**：Telegraf 同步存在毫秒级延迟，适合大部分场景，写入密集场景需评估延迟影响；
> 2. **扩展限制**：此方案为 “双节点热备”，无法实现数据分片，存储容量受单节点限制（需大规模存储建议升级商业版或更换时序库如 TimescaleDB）；
> 3. **监控告警**：需额外监控节点状态（如 `influx ping`）、同步延迟、Nginx 转发状态，推荐使用 Prometheus + Grafana 监控。
>
> 通过以上配置，可实现 InfluxDB OSS 版本的高可用架构，满足中小规模场景的可靠性需求。如果需要更复杂的多节点架构（如 3 节点集群），可在此基础上扩展节点并调整 Nginx 权重配置。