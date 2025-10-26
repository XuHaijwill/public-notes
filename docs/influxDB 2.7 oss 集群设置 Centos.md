# influxDB 2.7 oss 集群设置 Centos

> InfluxDB 2.7 OSS 版本**不原生支持集群功能**（集群是商业版 InfluxDB Enterprise 的特性），但可以通过 “多节点数据同步 + 反向代理负载均衡” 的方式模拟集群效果，实现高可用和负载分担。以下是基于 CentOS 系统的具体配置方案：
>
> ### 一、架构设计（3 节点示例）
>
> 采用 “1 主 2 从 + Telegraf 数据同步 + Nginx 负载均衡” 架构，适用于中小规模场景：
>
> - **主节点（node1）**：主要写入节点，数据同步到从节点；
> - **从节点（node2、node3）**：同步主节点数据，分担查询压力；
> - **Telegraf**：实现主节点到从节点的实时数据同步；
> - **Nginx**：统一入口，分发查询请求到从节点，写入请求定向到主节点。
>
> ### 二、环境准备
>
> #### 1. 服务器规划（3 台 CentOS 7/8 服务器）
>
> | 节点  | 角色     | IP 地址       | 需安装软件             |
> | ----- | -------- | ------------- | ---------------------- |
> | node1 | 主节点   | 192.168.1.101 | InfluxDB 2.7、Telegraf |
> | node2 | 从节点   | 192.168.1.102 | InfluxDB 2.7、Telegraf |
> | node3 | 从节点   | 192.168.1.103 | InfluxDB 2.7、Telegraf |
> | nginx | 反向代理 | 192.168.1.100 | Nginx                  |
>
> #### 2. 安装 InfluxDB 2.7 OSS（所有节点）
>
> ```bash
> # 下载并安装 InfluxDB 2.7
> wget https://dl.influxdata.com/influxdb/releases/influxdb2-2.7.5.x86_64.rpm
> sudo yum localinstall -y influxdb2-2.7.5.x86_64.rpm
> 
> # 启动并设置开机自启
> sudo systemctl start influxdb
> sudo systemctl enable influxdb
> 
> # 验证状态（默认端口 8086）
> sudo systemctl status influxdb
> ```
>
> #### 3. 初始化 InfluxDB（所有节点）
>
> 在每个节点执行初始化，创建统一的组织（org）、桶（bucket）和管理员 Token（确保配置一致）：
>
> ```bash
> influx setup \
>   --username admin \
>   --password "YourPass123!" \
>   --org myorg \
>   --bucket mybucket \
>   --retention 72h \  # 数据保留时间
>   --force
> ```
>
> - 记录每个节点的 **Token**（可在 `~/.influxdbv2/configs` 查看）和 IP。
>
> ### 三、配置数据同步（主→从）
>
> 通过 Telegraf 将主节点（node1）的写入数据同步到从节点（node2、node3）。
>
> #### 1. 安装 Telegraf（所有节点）
>
> ```bash
> sudo yum install -y telegraf
> sudo systemctl enable telegraf
> ```
>
> #### 2. 配置主节点（node1）的 Telegraf 同步
>
> 编辑 Telegraf 配置文件 `sudo vim /etc/telegraf/telegraf.conf`，添加从节点输出：
>
> ```toml
> # 输入：监听主节点的写入数据（通过 InfluxDB v2 监听器）
> [[inputs.influxdb_v2_listener]]
>   service_address = ":8186"  # 本地端口，接收主节点写入
>   token = "node1的Token"     # 主节点的认证Token
>   organization = "myorg"
>   bucket = "mybucket"
> 
> # 输出：同步到从节点 node2
> [[outputs.influxdb_v2]]
>   urls = ["http://192.168.1.102:8086"]
>   token = "node2的Token"
>   organization = "myorg"
>   bucket = "mybucket"
>   timeout = "10s"
> 
> # 输出：同步到从节点 node3
> [[outputs.influxdb_v2]]
>   urls = ["http://192.168.1.103:8086"]
>   token = "node3的Token"
>   organization = "myorg"
>   bucket = "mybucket"
>   timeout = "10s"
> ```
>
> #### 3. 启动 Telegraf 并验证同步
>
> ```bash
> # 重启 Telegraf
> sudo systemctl restart telegraf
> 
> # 主节点写入测试数据
> influx write --org myorg --bucket mybucket --url http://192.168.1.101:8086 --token "node1的Token" "cluster_test value=10"
> 
> # 在 node2 验证数据是否同步
> influx query 'from(bucket:"mybucket") |> range(start:-1m)' --org myorg --url http://192.168.1.102:8086 --token "node2的Token"
> ```
>
> ### 四、配置 Nginx 负载均衡（反向代理）
>
> 通过 Nginx 分发查询请求到从节点，写入请求定向到主节点，实现负载分担。
>
> #### 1. 安装 Nginx（192.168.1.100）
>
> ```bash
> sudo yum install -y nginx
> sudo systemctl start nginx
> sudo systemctl enable nginx
> ```
>
> #### 2. 配置 Nginx 规则
>
> 编辑 `sudo vim /etc/nginx/conf.d/influxdb_cluster.conf`：
>
> ```nginx
> upstream influxdb_write {
>   # 写入请求仅转发到主节点 node1
>   server 192.168.1.101:8086;
> }
> 
> upstream influxdb_query {
>   # 查询请求分发到从节点 node2、node3（权重可调整）
>   server 192.168.1.102:8086 weight=1;
>   server 192.168.1.103:8086 weight=1;
> }
> 
> server {
>   listen 8086;
>   server_name _;
> 
>   # 写入请求（POST 方法）转发到主节点
>   location /api/v2/write {
>     proxy_pass http://influxdb_write;
>     proxy_set_header Host $host;
>     proxy_set_header X-Real-IP $remote_addr;
>   }
> 
>   # 查询请求（GET 方法）转发到从节点集群
>   location /api/v2/query {
>     proxy_pass http://influxdb_query;
>     proxy_set_header Host $host;
>     proxy_set_header X-Real-IP $remote_addr;
>   }
> 
>   # 其他请求（如 UI、元数据）默认转发到主节点
>   location / {
>     proxy_pass http://192.168.1.101:8086;
>     proxy_set_header Host $host;
>     proxy_set_header X-Real-IP $remote_addr;
>   }
> }
> ```
>
> #### 3. 验证 Nginx 配置
>
> ```bash
> # 检查配置语法
> sudo nginx -t
> 
> # 重启 Nginx
> sudo systemctl restart nginx
> 
> # 测试通过 Nginx 写入和查询
> # 写入（自动转发到 node1）
> influx write --org myorg --bucket mybucket --url http://192.168.1.100:8086 --token "node1的Token" "nginx_test value=20"
> 
> # 查询（自动分发到 node2 或 node3）
> influx query 'from(bucket:"mybucket") |> range(start:-1m)' --org myorg --url http://192.168.1.100:8086 --token "node1的Token"
> ```
>
> ### 五、故障处理与优化
>
> 1. **主节点故障切换**：
>    - 若 node1 故障，手动修改 Nginx 配置，将写入请求临时转发到 node2（需确保 node2 数据最新）；
>    - 修复 node1 后，通过 `influx export/import` 同步 node2 的新增数据，再切回主节点。
> 2. **数据一致性保障**：
>    - 定时备份主节点数据（参考前文脚本），避免同步失效导致数据丢失；
>    - 监控 Telegraf 同步状态（如通过 `telegraf --test` 检查输出是否正常）。
> 3. **性能优化**：
>    - 从节点可配置更高的查询缓存（`influxd config` 中调整 `cache-max-memory-size`）；
>    - 根据服务器性能调整 Nginx 权重，避免低配置节点负载过高。
>
> ### 六、局限性说明
>
> - 此方案为 “伪集群”，不支持数据分片（所有节点存储全量数据），存储容量受单节点限制；
> - 无自动故障转移，需手动干预主节点故障；
> - 不支持商业版的集群管理、增量同步等高级功能。
>
> 若需真正的分布式集群，建议升级到 InfluxDB Enterprise 或考虑其他时序数据库（如 VictoriaMetrics、Cortex）。