#                        prometheus监控-学习笔记

:yum:  更新时间：2025.11.14

## 1. prometheus监控的部署方式

### 1.1 docker方式进行部署

| linux服务器版本        | Ubuntu 22.04 LTS |
| ---------------------- | ---------------- |
| **docker版本**         | **28.5.2**       |
| **docker-compose版本** | **v2.40.2**      |
|                        |                  |
|                        |                  |
|                        |                  |

#### 1.1.1 部署prometheus使用docker-compose的方式

```bash
# 手动创建共享网络(只需要创建一次即可)
docker network create monitoring

# 创建目录
mkdir /prometheus && cd /prometheus

# 创建prometheus.yml配置文件
vim prometheus.yml
global:
  scrape_interval: 15s  # 每15秒收集一次指标
  evaluation_interval: 15s  # 每15秒评估一次告警规则

# 告警规则配置（可选，先留空）
rule_files:
  # - "alert_rules.yml"

# 抓取配置 - 定义要监控的目标
scrape_configs:
  # 第一个任务：监控 Prometheus 自身
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']  # Prometheus 自己的指标端点

  # 第二个任务：监控 Docker 主机（如果你想知道服务器本身的状态）
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['被监控主机的IP地址:9100']
    scrape_interval: 30s

# 编写docker-compose配置文件
cd /docker-compose/prometheus/ && vim docker-compose.yml
services:
  # Prometheus 主服务
  prometheus:
    image: registry.cn-hangzhou.aliyuncs.com/ai-gpt-g/prometheus:v3.7.3
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - /prometheus/prometheus.yaml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=7d'
      - '--web.enable-remote-write-receiver'
    restart: unless-stopped
    networks:
      - monitoring

volumes:
  prometheus_data:

networks:
  monitoring:
    external: true
    name: monitoring

# 启动prometheus服务
docker-compose up -d

```

#### 1.1.2 部署node-exporter使用docker-compose的方式

```shell
# 编写docker-compose配置文件
root@VM-0-14-ubuntu:/docker-compose/node-exporter# cat docker-compose.yml
version: '3.8'

services:
  # Node Exporter
  node-exporter:
    image: registry.cn-hangzhou.aliyuncs.com/ai-gpt-g/node-exporter:v1.10.2
    container_name: node-exporter
    ports:
      - "9100:9100"
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    networks:
      - monitoring


networks:
  monitoring:
    external: true
    name: monitoring

# 启动服务
docker-compose up -d
```

**浏览器访问http://IP:9090**

![](https://gitee.com/yqjgxx/image/raw/master/prometheus/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202025-11-17%20144446.png)

#### 1.1.3 部署grafana使用docker-compose的方式

```shell
# 创建docker-compose配置文件 
root@VM-0-14-ubuntu:/docker-compose/grafana# cat docker-compose.yml
  # Grafana - 新增的服务
services:
  grafana:
    image: registry.cn-hangzhou.aliyuncs.com/ai-gpt-g/grafana:12.2-ubuntu
    container_name: grafana
    ports:
      - "3000:3000"  # Grafana Web 界面端口
    volumes:
      - grafana_data:/var/lib/grafana  # 持久化存储仪表盘、用户等数据
      - ./grafana/provisioning:/etc/grafana/provisioning  # 自动配置数据源和仪表盘
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=2932268642yqj  # 请修改这个密码！
      - GF_USERS_ALLOW_SIGN_UP=false  # 禁止用户注册
    restart: unless-stopped
    networks:
      - monitoring

volumes:
  grafana_data:  # 新增的 Grafana 数据卷

networks:
  monitoring:
    external: true
    name: monitoring

# 启动服务
docker-compose up -d
```

## 2. 配置grafana监控页面

### 2.1 导入promethues监控数据

**浏览器访问http://IP:3000**

![](https://gitee.com/yqjgxx/image/raw/master/prometheus/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202025-11-17%20220206.png)

**导入prometheus数据**

![](https://gitee.com/yqjgxx/image/raw/master/prometheus/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202025-11-17%20220653.png)

![](https://gitee.com/yqjgxx/image/raw/master/prometheus/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202025-11-17%20220912.png)

### 2.2 导入仪表盘

![](https://gitee.com/yqjgxx/image/raw/master/prometheus/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202025-11-17%20221248.png)