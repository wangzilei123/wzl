以下是为 Redis 集群配置 DNS 映射的详细部署步骤，以 **Bind9 DNS 服务器**为例：

---

### **环境准备**
| 组件          | 示例值                     |
|---------------|---------------------------|
| DNS 服务器 IP | 10.0.3.100                |
| 域名          | redis.example.com         |
| Redis 集群 A  | 10.0.1.101, 102, 103      |
| Redis 集群 B  | 10.0.2.101, 102, 103      |

---

### **步骤 1：安装 Bind9 DNS 服务器**
```bash
# Ubuntu/Debian
sudo apt update
sudo apt install bind9 bind9utils -y

# CentOS/RHEL
sudo yum install bind bind-utils -y
```

---

### **步骤 2：配置主配置文件**
编辑 `/etc/bind/named.conf`：
```bash
sudo vim /etc/bind/named.conf
```
添加以下内容：
```conf
options {
    directory "/var/cache/bind";
    allow-query { any; };  # 允许所有客户端查询
    recursion no;          # 关闭递归查询（纯权威DNS）
    dnssec-validation no;  # 简化配置（生产环境应启用）
};

# 添加域配置文件
include "/etc/bind/named.conf.local";
```

---

### **步骤 3：配置域文件**
#### 3.1 创建域声明文件  
编辑 `/etc/bind/named.conf.local`：
```bash
sudo vim /etc/bind/named.conf.local
```
```conf
zone "redis.example.com" {
    type master;
    file "/etc/bind/db.redis.example.com";
};
```

#### 3.2 创建区域数据文件  
新建 `/etc/bind/db.redis.example.com`：
```bash
sudo vim /etc/bind/db.redis.example.com
```
```conf
; TTL 设置为 60 秒（快速切换）
$TTL 60

@       IN      SOA     ns1.redis.example.com. admin.redis.example.com. (
                        2024062601 ; 序列号 (每次修改+1)
                        3600       ; 刷新时间
                        1800       ; 重试时间
                        604800     ; 过期时间
                        86400 )    ; 最小TTL

; 名称服务器记录
        IN      NS      ns1.redis.example.com.

; DNS 服务器 A 记录
ns1     IN      A       10.0.3.100

; Redis 集群 A 记录（指向集群 A）
redis-cluster IN  A   10.0.1.101
redis-cluster IN  A   10.0.1.102
redis-cluster IN  A   10.0.1.103

; 可选：独立节点记录
node1   IN      A       10.0.1.101
node2   IN      A       10.0.1.102
node3   IN      A       10.0.1.103
```

---

### **步骤 4：启动 DNS 服务**
```bash
# 检查配置文件语法
sudo named-checkconf
sudo named-checkzone redis.example.com /etc/bind/db.redis.example.com

# 重启服务
sudo systemctl restart bind9

# 设置开机自启
sudo systemctl enable bind9
```

---

### **步骤 5：配置客户端 DNS 解析**
在客户端机器（应用服务器）配置 DNS 服务器：
```bash
# Ubuntu/Debian
sudo vim /etc/resolv.conf
# 添加：
nameserver 10.0.3.100

# CentOS/RHEL
sudo vim /etc/sysconfig/network-scripts/ifcfg-eth0
# 添加：
DNS1=10.0.3.100
sudo systemctl restart NetworkManager
```

---

### **步骤 6：测试 DNS 解析**
```bash
# 安装测试工具
sudo apt install dnsutils -y  # 或 sudo yum install bind-utils

# 测试解析
dig redis-cluster.redis.example.com +short
```
预期输出：
```
10.0.1.101
10.0.1.102
10.0.1.103
```

---

### **步骤 7：配置 Redis 客户端**
以 Java (Lettuce) 为例：
```java
RedisClusterClient clusterClient = RedisClusterClient.create(
    "redis://redis-cluster.redis.example.com:6379"
);
```

---

### **步骤 8：集群切换操作**
当需要切换到集群 B 时：
1. 修改区域文件：
   ```bash
   sudo vim /etc/bind/db.redis.example.com
   ```
   替换 A 记录：
   ```diff
   - redis-cluster IN  A   10.0.1.101
   - redis-cluster IN  A   10.0.1.102
   - redis-cluster IN  A   10.0.1.103
   + redis-cluster IN  A   10.0.2.101
   + redis-cluster IN  A   10.0.2.102
   + redis-cluster IN  A   10.0.2.103
   ```
2. **增加序列号**（SOA 记录的第二个字段）：
   ```diff
   - 2024062601 ; 序列号
   + 2024062602 ; 序列号+1
   ```
3. 重新加载配置：
   ```bash
   sudo rndc reload redis.example.com
   ```
4. 客户端会在 60 秒（TTL 时间）后自动切换到新集群

---

### **增强方案：通过 HAProxy 代理**
#### 1. 安装 HAProxy
```bash
sudo apt install haproxy -y  # 或 sudo yum install haproxy
```

#### 2. 配置 `/etc/haproxy/haproxy.cfg`
```conf
frontend redis_front
    bind *:6379
    mode tcp
    default_backend redis_cluster

backend redis_cluster
    mode tcp
    balance roundrobin
    option tcp-check
    tcp-check connect
    tcp-check send PING\r\n
    tcp-check expect string +PONG
    server redis1 10.0.1.101:6379 check
    server redis2 10.0.1.102:6379 check
    server redis3 10.0.1.103:6379 check
```

#### 3. 修改 DNS 指向 HAProxy
```conf
; /etc/bind/db.redis.example.com
redis-proxy IN  A   10.0.3.200  ; HAProxy 服务器 IP
```

---

### **验证与监控**
```bash
# 实时监控 DNS 查询
sudo tcpdump -i eth0 port 53

# 检查 DNS 解析延迟
dig redis-cluster.redis.example.com | grep "Query time"

# 测试 Redis 连通性
redis-cli -c -h redis-cluster.redis.example.com -p 6379 PING
```

### **关键注意事项**
1. **TTL 设置**：生产环境建议 TTL=60，平衡切换速度和 DNS 查询负载
2. **客户端兼容性**：
   - 确保客户端支持 DNS 轮询（如 Lettuce、Jedis Cluster）
   - 旧客户端需使用代理方案
3. **高可用 DNS**：
   - 部署至少 2 台 Bind9 服务器
   - 使用 `keepalived` 实现 VIP 漂移
4. **防火墙规则**：
   ```bash
   sudo ufw allow 53/tcp    # DNS TCP
   sudo ufw allow 53/udp    # DNS UDP
   sudo ufw allow 6379/tcp  # Redis
   ```

> 完整方案可在 30 分钟内完成部署，切换集群只需修改 DNS 记录并等待 TTL 过期。****
