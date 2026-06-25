# 基础服务 Jenkins 单容器启动命令

## 1. 使用原则

以下命令用于 Jenkins 分别控制基础服务启动，不使用 Docker Compose，不加入 `docker_agent-network`，全部使用默认 `bridge` 网络，并通过宿主机映射端口对外提供服务。

数据目录继续复用原 `nuwax_deploy/docker` 下的 bind mount，避免数据丢失：

```bash
DEPLOY_DIR=/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker
```

## 2. MySQL

### 2.1 权限修复

```bash
set -e

DEPLOY_DIR=/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker
MYSQL_UID=${MYSQL_UID:-999}
MYSQL_GID=${MYSQL_GID:-999}

docker run --rm \
  -v "${DEPLOY_DIR}/data/mysql:/var/lib/mysql" \
  -v "${DEPLOY_DIR}/logs/mysql:/var/log/mysql" \
  -v "${DEPLOY_DIR}/config/mysql.cnf:/tmp/mysql.cnf:rw" \
  busybox:1.36-uclibc \
  sh -c "mkdir -p /var/lib/mysql /var/log/mysql && chown -R ${MYSQL_UID}:${MYSQL_GID} /var/lib/mysql /var/log/mysql && chmod -R 755 /var/lib/mysql /var/log/mysql && [ ! -f /tmp/mysql.cnf ] || chmod 644 /tmp/mysql.cnf"
```

### 2.2 启动 MySQL

```bash
set -e

DEPLOY_DIR=/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker
MYSQL_CONTAINER=${MYSQL_CONTAINER:-nuwax-mysql}
MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD:-root}
MYSQL_DATABASE=${MYSQL_DATABASE:-agent_platform}
MYSQL_USER=${MYSQL_USER:-agent_platform}
MYSQL_PASSWORD=${MYSQL_PASSWORD:-admin123}

docker rm -f "${MYSQL_CONTAINER}" >/dev/null 2>&1 || true

docker run -d \
  --name "${MYSQL_CONTAINER}" \
  --restart=always \
  -p 13306:3306 \
  -e MYSQL_ROOT_PASSWORD="${MYSQL_ROOT_PASSWORD}" \
  -e MYSQL_DATABASE="${MYSQL_DATABASE}" \
  -e MYSQL_USER="${MYSQL_USER}" \
  -e MYSQL_PASSWORD="${MYSQL_PASSWORD}" \
  -e TZ=Asia/Shanghai \
  -e MYSQL_CHARSET=utf8mb4 \
  -e MYSQL_COLLATION=utf8mb4_unicode_ci \
  -v "${DEPLOY_DIR}/data/mysql:/var/lib/mysql" \
  -v "${DEPLOY_DIR}/config/mysql.cnf:/etc/mysql/conf.d/mysql.cnf:ro" \
  -v "${DEPLOY_DIR}/config/init_mysql.sql:/docker-entrypoint-initdb.d/1_init_mysql.sql:ro" \
  -v "${DEPLOY_DIR}/config/init_mysql_data.sql:/docker-entrypoint-initdb.d/2_init_mysql_data.sql:ro" \
  -v "${DEPLOY_DIR}/logs/mysql:/var/log/mysql" \
  --health-cmd='mysqladmin ping -h localhost -uroot -p"$MYSQL_ROOT_PASSWORD" --silent' \
  --health-interval=10s \
  --health-timeout=5s \
  --health-retries=5 \
  --health-start-period=30s \
  mysql:8.0 \
  --authentication-policy=mysql_native_password \
  --character-set-server=utf8mb4 \
  --collation-server=utf8mb4_unicode_ci \
  --init-connect='SET NAMES utf8mb4 COLLATE utf8mb4_unicode_ci; SET character_set_client=utf8mb4; SET character_set_connection=utf8mb4; SET character_set_results=utf8mb4;' \
  --explicit_defaults_for_timestamp=true \
  --lower_case_table_names=1
```

### 2.3 验证 MySQL

```bash
docker ps --filter name=nuwax-mysql --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

## 3. Redis

### 3.1 启动 Redis

```bash
set -e

DEPLOY_DIR=/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker
REDIS_CONTAINER=${REDIS_CONTAINER:-nuwax-redis}
REDIS_PASSWORD=${REDIS_PASSWORD:-123456}

docker rm -f "${REDIS_CONTAINER}" >/dev/null 2>&1 || true

docker run -d \
  --name "${REDIS_CONTAINER}" \
  --restart=always \
  -p 16379:6379 \
  -e TZ=Asia/Shanghai \
  -v "${DEPLOY_DIR}/data/redis:/data" \
  -v "${DEPLOY_DIR}/config/redis.conf:/usr/local/etc/redis/redis.conf:ro" \
  -v "${DEPLOY_DIR}/logs/redis:/data/logs/redis" \
  --health-cmd="redis-cli -a ${REDIS_PASSWORD} -p 6379 ping | grep PONG" \
  --health-interval=10s \
  --health-timeout=5s \
  --health-retries=5 \
  redis:7.0 \
  redis-server /usr/local/etc/redis/redis.conf --port 6379 --requirepass "${REDIS_PASSWORD}"
```

### 3.2 验证 Redis

```bash
docker ps --filter name=nuwax-redis --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

## 4. Milvus

### 4.1 启动 Milvus

```bash
set -e

DEPLOY_DIR=/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker
MILVUS_CONTAINER=${MILVUS_CONTAINER:-nuwax-milvus}

docker rm -f "${MILVUS_CONTAINER}" >/dev/null 2>&1 || true

docker run -d \
  --name "${MILVUS_CONTAINER}" \
  --restart=always \
  -p 19530:19530 \
  -p 9091:9091 \
  -p 2379:2379 \
  -e TZ=Asia/Shanghai \
  -e ETCD_USE_EMBED=true \
  -e ETCD_DATA_DIR=/var/lib/milvus/etcd \
  -e ETCD_CONFIG_PATH=/milvus/configs/embedEtcd.yaml \
  -e COMMON_STORAGETYPE=local \
  -e TIMEZONE=Asia/Shanghai \
  -e LOG_LEVEL=error \
  -e MILVUS_LOG_LEVEL=error \
  -e LOG_STDOUT=false \
  -e LOG_FILE_ROOTPATH=/var/log/milvus \
  -v "${DEPLOY_DIR}/data/milvus/data:/var/lib/milvus/data" \
  -v "${DEPLOY_DIR}/data/milvus/etcd:/var/lib/milvus/etcd" \
  -v "${DEPLOY_DIR}/config/milvus/embedEtcd.yaml:/milvus/configs/embedEtcd.yaml:ro" \
  -v "${DEPLOY_DIR}/config/milvus/user.yaml:/milvus/configs/user.yaml:ro" \
  -v "${DEPLOY_DIR}/config/milvus/milvus.yaml:/milvus/configs/milvus.yaml:ro" \
  -v "${DEPLOY_DIR}/logs/milvus:/var/log/milvus" \
  --security-opt seccomp:unconfined \
  --log-driver=json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  --health-cmd='curl -f http://localhost:9091/healthz || exit 1' \
  --health-interval=30s \
  --health-timeout=20s \
  --health-retries=3 \
  --health-start-period=90s \
  milvusdb/milvus:v2.5.8 \
  milvus run standalone
```

### 4.2 验证 Milvus

```bash
curl -f http://localhost:9091/healthz
```

## 5. MinIO

### 5.1 启动 MinIO

```bash
set -e

DEPLOY_DIR=/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker
MINIO_CONTAINER=${MINIO_CONTAINER:-nuwax-minio}
MINIO_ROOT_USER=${MINIO_ROOT_USER:-minioadmin}
MINIO_ROOT_PASSWORD=${MINIO_ROOT_PASSWORD:-minioadmin}

docker rm -f "${MINIO_CONTAINER}" >/dev/null 2>&1 || true

docker run -d \
  --name "${MINIO_CONTAINER}" \
  --restart=always \
  -p 9000:9000 \
  -p 9001:9001 \
  -e MINIO_ROOT_USER="${MINIO_ROOT_USER}" \
  -e MINIO_ROOT_PASSWORD="${MINIO_ROOT_PASSWORD}" \
  -e TZ=Asia/Shanghai \
  -v "${DEPLOY_DIR}/data/minio:/data" \
  --health-cmd='curl -f http://localhost:9000/minio/health/live || exit 1' \
  --health-interval=30s \
  --health-timeout=10s \
  --health-retries=3 \
  --health-start-period=30s \
  minio/minio:latest \
  server /data --console-address ":9001"
```

### 5.2 初始化 MinIO Bucket

```bash
set -e

DEPLOY_DIR=/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker
MINIO_INIT_CONTAINER=${MINIO_INIT_CONTAINER:-nuwax-minio-init}
MINIO_ROOT_USER=${MINIO_ROOT_USER:-minioadmin}
MINIO_ROOT_PASSWORD=${MINIO_ROOT_PASSWORD:-minioadmin}
MINIO_BUCKET_NAME=${MINIO_BUCKET_NAME:-quickwit-indexes}
MINIO_SERVER_URL=${MINIO_SERVER_URL:-http://host.docker.internal:9000}

docker rm -f "${MINIO_INIT_CONTAINER}" >/dev/null 2>&1 || true

docker run --rm \
  --name "${MINIO_INIT_CONTAINER}" \
  --entrypoint /bin/sh \
  -v "${DEPLOY_DIR}/script:/scripts" \
  -e MINIO_SERVER_URL="${MINIO_SERVER_URL}" \
  -e MINIO_ROOT_USER="${MINIO_ROOT_USER}" \
  -e MINIO_ROOT_PASSWORD="${MINIO_ROOT_PASSWORD}" \
  -e MINIO_BUCKET_NAME="${MINIO_BUCKET_NAME}" \
  minio/mc:latest \
  -c "chmod +x /scripts/init-minio.sh && /scripts/init-minio.sh"
```

### 5.3 验证 MinIO

```bash
curl -f http://localhost:9000/minio/health/live
```

## 6. Elasticsearch

### 6.1 权限修复

```bash
set -e

DEPLOY_DIR=/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker
ELASTICSEARCH_UID=${ELASTICSEARCH_UID:-1000}
ELASTICSEARCH_GID=${ELASTICSEARCH_GID:-1000}

docker run --rm \
  -v "${DEPLOY_DIR}/data/elasticsearch:/usr/share/elasticsearch/data" \
  -v "${DEPLOY_DIR}/logs/elasticsearch:/usr/share/elasticsearch/logs" \
  busybox:1.36-uclibc \
  sh -c "mkdir -p /usr/share/elasticsearch/data /usr/share/elasticsearch/logs && chown -R ${ELASTICSEARCH_UID}:${ELASTICSEARCH_GID} /usr/share/elasticsearch/data /usr/share/elasticsearch/logs && chmod -R 755 /usr/share/elasticsearch/data /usr/share/elasticsearch/logs"
```

### 6.2 启动 Elasticsearch

```bash
set -e

DEPLOY_DIR=/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker
ELASTICSEARCH_CONTAINER=${ELASTICSEARCH_CONTAINER:-elasticsearch}
ELASTICSEARCH_PASSWORD=${ELASTICSEARCH_PASSWORD:-elastic123}

docker rm -f "${ELASTICSEARCH_CONTAINER}" >/dev/null 2>&1 || true

docker run -d \
  --name "${ELASTICSEARCH_CONTAINER}" \
  --restart=always \
  -p 9200:9200 \
  -p 9300:9300 \
  -e discovery.type=single-node \
  -e xpack.security.enabled=true \
  -e ELASTIC_PASSWORD="${ELASTICSEARCH_PASSWORD}" \
  -e ES_JAVA_OPTS="-Xms1g -Xmx2g" \
  -e bootstrap.memory_lock=false \
  -e cluster.name=docker-cluster \
  -e node.name=es01 \
  -e TZ=Asia/Shanghai \
  -e path.data=/usr/share/elasticsearch/data \
  -e path.logs=/usr/share/elasticsearch/logs \
  -v "${DEPLOY_DIR}/data/elasticsearch:/usr/share/elasticsearch/data" \
  -v "${DEPLOY_DIR}/logs/elasticsearch:/usr/share/elasticsearch/logs" \
  -v "${DEPLOY_DIR}/config/elasticsearch/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro" \
  -v "${DEPLOY_DIR}/config/elasticsearch/plugins/elasticsearch-analysis-ik-9.2.1:/usr/share/elasticsearch/plugins/analysis-ik:rw" \
  --health-cmd="curl -f -u elastic:${ELASTICSEARCH_PASSWORD} http://localhost:9200/_cluster/health || exit 1" \
  --health-interval=30s \
  --health-timeout=10s \
  --health-retries=5 \
  --health-start-period=60s \
  docker.elastic.co/elasticsearch/elasticsearch:9.2.1
```

### 6.3 验证 Elasticsearch

```bash
curl -f -u elastic:elastic123 http://localhost:9200/_cluster/health
```

## 7. 通用验证

## 7. Quickwit

### 7.1 启动 Quickwit

Quickwit 依赖 MinIO 的 `quickwit-indexes` bucket。单容器跨机器模式下，不使用 `http://minio:9000`，改为访问宿主机或远端 MinIO 地址。

```bash
set -e

DEPLOY_DIR=/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker
QUICKWIT_CONTAINER=${QUICKWIT_CONTAINER:-nuwax-quickwit}
QUICKWIT_IMAGE=${QUICKWIT_IMAGE:-quickwit/quickwit:latest}
QUICKWIT_HTTP_PORT=${QUICKWIT_HTTP_PORT:-7280}
QUICKWIT_GRPC_PORT=${QUICKWIT_GRPC_PORT:-7281}
MINIO_ROOT_USER=${MINIO_ROOT_USER:-minioadmin}
MINIO_ROOT_PASSWORD=${MINIO_ROOT_PASSWORD:-minioadmin}
QW_S3_ENDPOINT=${QW_S3_ENDPOINT:-http://host.docker.internal:9000}
QW_METASTORE_URI=${QW_METASTORE_URI:-s3://quickwit-indexes}
QW_DEFAULT_INDEX_ROOT_URI=${QW_DEFAULT_INDEX_ROOT_URI:-s3://quickwit-indexes/}

docker rm -f "${QUICKWIT_CONTAINER}" >/dev/null 2>&1 || true

docker run -d \
  --name "${QUICKWIT_CONTAINER}" \
  --restart=always \
  -p "${QUICKWIT_HTTP_PORT}:7280" \
  -p "${QUICKWIT_GRPC_PORT}:7281" \
  -e TZ=Asia/Shanghai \
  -e QW_S3_ENDPOINT="${QW_S3_ENDPOINT}" \
  -e AWS_ACCESS_KEY_ID="${MINIO_ROOT_USER}" \
  -e AWS_SECRET_ACCESS_KEY="${MINIO_ROOT_PASSWORD}" \
  -e QW_S3_FLAVOR=minio \
  -e AWS_REGION=us-east-1 \
  -e QW_S3_FORCE_PATH_STYLE_ACCESS=true \
  -e QW_METASTORE_URI="${QW_METASTORE_URI}" \
  -e QW_DEFAULT_INDEX_ROOT_URI="${QW_DEFAULT_INDEX_ROOT_URI}" \
  -v "${DEPLOY_DIR}/data/quickwit:/quickwit/qwdata" \
  -v "${DEPLOY_DIR}/logs/quickwit:/quickwit/logs" \
  --health-cmd='curl -f http://localhost:7280/api/v1/version || exit 1' \
  --health-interval=30s \
  --health-timeout=10s \
  --health-retries=3 \
  --health-start-period=30s \
  "${QUICKWIT_IMAGE}" \
  run
```

### 7.2 验证 Quickwit

```bash
curl -f http://localhost:7280/api/v1/version
docker ps --filter name=nuwax-quickwit --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

## 8. 通用验证

查看全部基础服务容器：

```bash
docker ps --filter 'name=nuwax-mysql\|nuwax-redis\|nuwax-milvus\|nuwax-minio\|nuwax-quickwit\|elasticsearch' \
  --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

确认没有加入 `docker_agent-network`：

```bash
docker inspect -f '{{.Name}} {{range $name, $_ := .NetworkSettings.Networks}}{{$name}} {{end}}' \
  nuwax-mysql nuwax-redis nuwax-milvus nuwax-minio nuwax-quickwit elasticsearch
```

期望输出均为 `bridge`。

## 9. 后端跨机器连接参数

后端单容器启动时按外部地址注入依赖，示例：

```bash
MYSQL_HOST=host.docker.internal
MYSQL_PORT=13306
REDIS_HOST=host.docker.internal
REDIS_PORT=16379
MILVUS_URI=http://host.docker.internal:19530
ES_URL=http://host.docker.internal:9200
```

部署到不同机器时，将 `host.docker.internal` 替换为基础服务所在机器的 IP、域名或负载均衡地址。

## 10. 注意事项

- 不执行 `docker compose down -v`。
- 不删除 `DEPLOY_DIR/data/*`、`DEPLOY_DIR/logs/*`、`DEPLOY_DIR/config/*`。
- Jenkins 如果拆成多个 Job，MySQL 和 Elasticsearch 建议先执行权限修复，再执行启动。
- MinIO 建议先启动服务，等待健康检查通过后，再执行 Bucket 初始化。
