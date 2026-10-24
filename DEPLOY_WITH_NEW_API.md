# 与 new-api 同机部署 AliMPay（Docker Compose）

以下命令按服务器的 root shell 编写；普通账户执行系统级操作时需使用 sudo。示例以 Ubuntu/Debian、宿主机 Nginx、已有 new-api 占用 `127.0.0.1:3000`、AliMPay 目录 `/data/AliMPay` 和独立域名 `pay.tokenaigate.com` 为前提。实际目录或域名不同就替换示例值。AliMPay 在容器内部仍监听 3000，只把宿主机 `127.0.0.1:3001` 映射到容器 3000；不要修改 new-api 的端口。

## 1. 准备域名与代码

给 `pay.tokenaigate.com` 配置指向服务器的 DNS 记录，确认服务器已安装 Docker Compose 和 Nginx，并检查 3001 尚未被占用：

```bash
docker compose version
nginx -v
ss -lntp | grep -E ':3000|:3001' || true
```

代码使用自己的 AliMPay 仓库；已有部署目录时直接进入，保留原有 `data/` 和 `.env`，不要重新克隆覆盖：

```bash
# 仅首次部署且目标目录不存在时执行 clone
git clone https://github.com/lgwtonylgw/AliMPay.git /data/AliMPay
cd /data/AliMPay
```

如果 3001 已被其他服务占用，可选一个空闲端口并在后续 `.env`、健康检查和 Nginx 配置中统一替换。

## 2. 配置 Compose 与数据目录

确认 `compose.yaml` 的端口配置是：

```yaml
ports:
  - "127.0.0.1:${ALIMPAY_HOST_PORT:-3001}:3000"
```

若服务器上的旧版仍为 `127.0.0.1:3000:3000`，先改成上面这一行。创建项目自己的 `.env`：

```dotenv
PUBLIC_BASE_URL=https://pay.tokenaigate.com
ALIMPAY_HOST_PORT=3001
```

`PUBLIC_BASE_URL` 要写外部访问的 HTTPS 根地址，不写 `127.0.0.1:3001`。全新部署默认不设置 `APP_MASTER_KEY`，程序首次启动会在持久化目录生成 `data/.master-key`。已经运行过的实例必须保留原有 `data/` 和这把密钥；如果此前在 `.env` 中设置过 `APP_MASTER_KEY`，也必须保留原值，不能换成新密钥。

`.env` 中的 `PORT`、`DATA_DIR` 是直接运行 Bun 时的示例；Compose 明确指定容器内 `PORT=3000`、`DATA_DIR=/data`，并通过 `./data:/data` 把宿主机的 `./data` 挂进去。容器日志中的 `/data/.master-key` 就是宿主机 AliMPay 项目下的 `data/.master-key`。非空 `APP_MASTER_KEY` 会跳过该文件；如果刚修改了 `.env`，必须重新创建容器才能使用新环境变量。

不要把宿主机项目路径 `/data/AliMPay` 填入 Compose 的 `DATA_DIR`；它会被当作**容器内路径**，而目前只挂载了容器内的 `/data`。也不要把容器内 `PORT` 改为 3001：宿主机 3001 已由端口映射指定，容器及镜像健康检查仍使用 3000。

如果先前启动失败且容器仍在反复重启，先运行 `docker compose stop alimpay`，再执行下面的数据目录权限检查。

```bash
cd /data/AliMPay
chmod 600 .env
mkdir -p data
docker compose build --pull alimpay
ALI_UID=$(docker compose run --rm --no-deps --entrypoint id alimpay -u)
ALI_GID=$(docker compose run --rm --no-deps --entrypoint id alimpay -g)
chown -R "${ALI_UID}:${ALI_GID}" ./data
chmod -R u+rwX ./data
if [ -f ./data/.master-key ]; then chmod 600 ./data/.master-key; fi
docker compose run --rm --no-deps --entrypoint sh alimpay -c 'id; ls -ldn /data; tmp=$(mktemp /data/.write-probe.XXXXXX) && rm "$tmp"'
```

最后一行必须成功，才启动服务。它以容器中的应用用户测试 `/data` 可写，能提前发现 `EACCES` 和 `SQLITE_READONLY_DIRECTORY`。若失败，先核对当前工作目录、`ls -ldn ./data`、`findmnt -T ./data -o TARGET,SOURCE,FSTYPE,OPTIONS`，不要删除数据库、WAL/SHM 文件或 `.master-key`。SQLite WAL 模式需要在数据库目录创建日志文件，因此数据目录应位于支持该模式的本地可写文件系统。

## 3. 启动并检查

```bash
cd /data/AliMPay
docker compose up -d --no-build
docker compose ps
docker compose logs --tail=50 alimpay
curl -fsS http://127.0.0.1:3001/healthz
```

健康检查成功应返回含 `"ok":true` 的 JSON。端口只绑定本机回环地址，对外访问交给 Nginx。AliMPay 只运行一个应用实例。

## 4. Nginx 与 HTTPS

以下适用于 **Nginx 直接运行在宿主机** 的情况。在已有的 Nginx 站点目录中为 `pay.tokenaigate.com` 新建独立站点，不改 new-api 原有站点：

```nginx
server {
    listen 80;
    server_name pay.tokenaigate.com;
    client_max_body_size 6m;

    location / {
        proxy_pass http://127.0.0.1:3001;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_read_timeout 30s;
    }
}
```

如果系统使用 `/etc/nginx/sites-available/` 和 `sites-enabled/`，把配置保存为 `/etc/nginx/sites-available/pay.tokenaigate.com`，再启用并验证：

```bash
ln -s /etc/nginx/sites-available/pay.tokenaigate.com /etc/nginx/sites-enabled/pay.tokenaigate.com
nginx -t
systemctl reload nginx
```

已有同名站点或符号链接时编辑原文件，不重复创建。确认域名指向本机且 HTTP 可访问后，用服务器现有的证书方案启用 HTTPS；如果已经安装 Certbot 的 Nginx 插件，可运行：

```bash
certbot --nginx -d pay.tokenaigate.com
nginx -t
curl -fsS https://pay.tokenaigate.com/healthz
```

Certbot 会为该站点配置证书与 HTTPS。这里由 Nginx 重设 `X-Forwarded-For`，避免把客户端伪造的转发头传给 AliMPay 的登录限流逻辑；如果前面还有 Cloudflare 等代理，需要先在 Nginx 中单独配置受信任的真实 IP 来源。如果 Nginx 本身也在容器中，不能在 Nginx 容器内用 `127.0.0.1:3001` 指代宿主机，需改用共同 Docker 网络中的服务地址。

Cloudflare 中 `pay.tokenaigate.com` 的 A 记录可保持橙云代理。确认 **SSL/TLS → 概述**为 **Full (strict)**，并确认源站 Nginx 的证书覆盖 `pay.tokenaigate.com`。主站的 `/api/user/epay/notify` 是 AliMPay 到 new-api 的服务端回调；如果设置了“缓存所有内容”、人机验证或严格的 WAF 规则，需要确保该路径不被缓存或挑战。用真实小额订单检查回调，不要仅凭收银台跳转判断充值成功。Cloudflare 的 DNS 配置无法代替 Nginx 的证书与反向代理配置。

## 5. 首次设置及 new-api 对接

1. 打开 `https://pay.tokenaigate.com`，完成固定管理员 `admin` 的首次密码设置，确认公开根地址与 `PUBLIC_BASE_URL` 一致。
2. 在 AliMPay“密钥中心”生成支付宝应用密钥，将**应用公钥**上传支付宝开放平台；在“收款配置”填写支付宝应用 ID、**支付宝公钥**，取得账务明细查询权限并执行连接测试。
3. 上传支付宝经营码。AliMPay 后台取得易支付 V1 商户 ID 和 V1 密钥。
4. 在 new-api“系统设置 → 计费与支付 → 支付网关”的“支付方式”启用“使用 AliMPay（仅支付宝）”，支付类型只使用 `alipay`。
5. 在 new-api 的“Epay”页签填写 `https://pay.tokenaigate.com`、AliMPay 的 V1 商户 ID 和 V1 密钥。回调站点地址填写 new-api 自己的 HTTPS 根地址（本例为 `https://tokenaigate.com`），不是 AliMPay 地址。保存设置。
6. 确认公网可访问 new-api 的 `/api/user/epay/notify`，再做一笔小额真实订单，核对实际到账、AliMPay 订单、签名回调和 new-api 钱包余额。测试成功后再开放充值。

AliMPay 默认拒绝向私网回调，生产环境不要把 `ALLOW_PRIVATE_CALLBACKS` 改成 `true`。经营码模式会给实付金额增加 0.01–0.99 元用于订单匹配；new-api 按原始下单金额入账。

“常规 → 付款方式”只保留 `alipay` 供 AliMPay 使用。另一个独立部署的 USDT 网关走 new-api 的“epusdt”页签；它配置完整后会自动出现在钱包，不要在 AliMPay 的 `PayMethods` 中手动添加 `usdt.tron`。

## 6. 备份与更新

`data/` 含 SQLite 主库、可能尚未合并的 `-wal`/`-shm`、经营码上传文件和 `.master-key`。离线备份可先停止容器，然后把整个 `data/` 连同 `.env` 一起归档；备份包含密钥，应限制访问：

```bash
cd /data/AliMPay
docker compose stop alimpay
umask 077
ALI_BACKUP="/root/alimpay-backup-$(date +%F-%H%M%S).tgz"
tar -czf "$ALI_BACKUP" data .env
chmod 600 "$ALI_BACKUP"
docker compose up -d
```

更新前先备份，再拉取自己的仓库代码并重建。更新后核对容器日志和本机、HTTPS 两级健康检查：

```bash
cd /data/AliMPay
git pull --ff-only
docker compose build --pull alimpay
docker compose up -d --no-build
docker compose logs --tail=50 alimpay
curl -fsS http://127.0.0.1:3001/healthz
curl -fsS https://pay.tokenaigate.com/healthz
```

如果服务器上的 `data/` 或 `.env` 已有真实配置，只修权限和代码，绝不清空后重新初始化。
