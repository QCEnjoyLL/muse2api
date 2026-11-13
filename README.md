# muse2api 镜像构建仓

本仓库**不含任何源码**，只负责定时把上游 [czg86389-hub/muse2api](https://github.com/czg86389-hub/muse2api) 的代码打成 Docker 镜像，推到 GitHub Container Registry。

- 自动构建：**每周一 09:00（北京时间）**
- 手动构建：Actions → `build-image` → **Run workflow**
- 镜像地址：`ghcr.io/qcjoyll/muse2api`

> 注意镜像名全小写。GitHub 用户名是 `QCEnjoyLL`，但 GHCR 不接受大写，请用 `qcjoyll`。

| Tag | 含义 |
|---|---|
| `latest` | 最近一次成功构建 |
| `v1.6.10` | 上游 `version.json` 里的版本号 |
| `sha-b60f474` | 上游 commit，精确定位 |

---

## 一、先手动跑一次

仓库刚建好时还没有镜像，建议先手动出一次：

1. 打开 **Actions** → 左侧 `build-image` → **Run workflow** → 分支选 `main` → 绿色按钮。
2. 约 5~10 分钟跑完（第一次没缓存，之后有缓存会快很多）。
3. 成功后到 **个人头像 → Your profile → Packages** 能看到 `muse2api`。

### 把包设为 public（推荐）

包默认可能是 private，那样服务器拉取要登录。改成公开后就可以匿名 `docker pull`：

**Packages → muse2api → Package settings → Danger Zone → Change visibility → Public**

如果不想公开，就在服务器上登录一次（PAT 需要勾 `read:packages`）：

```bash
echo "<你的PAT>" | docker login ghcr.io -u QCEnjoyLL --password-stdin
```

---

## 二、服务器上跑起来（docker cli）

全程只用 `docker` 命令，不需要装 git、不需要 clone 源码。

```bash
docker pull ghcr.io/qcjoyll/muse2api:latest
```

```bash
mkdir -p /opt/muse2api/data
cd /opt/muse2api

docker run -d \
  --name muse2api \
  --restart unless-stopped \
  -p 18610:18610 \
  --shm-size=2g \
  -e MUSE2API_HOST=0.0.0.0 \
  -e MUSE2API_PORT=18610 \
  -e MUSE2API_KEY=m2a_your_admin_key \
  -e MUSE2API_CONCURRENCY=2 \
  -e MUSE2API_IMAGE_TIMEOUT=240 \
  -e MUSE2API_VIDEO_TIMEOUT=600 \
  -e MUSE2API_CHAT_TIMEOUT=300 \
  -v /opt/muse2api/data:/app/data \
  ghcr.io/qcjoyll/muse2api:latest
```

参数说明：

| 参数 | 作用 |
|---|---|
| `--shm-size=2g` | **不能省**。Chromium 多标签页会吃 `/dev/shm`，默认 64MB 会崩 |
| `-v /opt/muse2api/data:/app/data` | 账号 Cookie、任务记录、生成的图片视频都在这，`docker rm` 不会丢 |
| `-p 18610:18610` | 服务端口 |
| `-e MUSE2API_HOST=0.0.0.0` | 容器内必须监听 0.0.0.0，否则宿主机连不上 |
| `-e MUSE2API_KEY` | 管理面板和 API 的密钥。不传也行，会自动生成并打印在日志里 |
| `-e MUSE2API_CONCURRENCY` | 最大并发生成数，默认 2，机器强可以调高 |

### 拿管理密钥

如果你没传 `MUSE2API_KEY`，从日志里捞：

```bash
docker logs muse2api 2>&1 | grep -o 'm2a_[A-Za-z0-9_-]*' | head -1
```

### 打开管理面板

```
http://<服务器IP>:18610/admin?key=<你的MUSE2API_KEY>
```

API 入口就是 `http://<服务器IP>:18610/v1`，请求头带 `Authorization: Bearer <key>`。

服务起来后账号池是空的，还需要在管理页导入 muse.ai 的登录态（用上游仓库的 Chrome 扩展或油猴脚本，填这个服务地址 + key）。

---

## 三、更新到新镜像

上游更新后，镜像会在下一个周一自动构建。服务器的更新就是 pull + 重建容器（`data/` 不动，账号不丢）：

```bash
docker pull ghcr.io/qcjoyll/muse2api:latest
docker rm -f muse2api
docker run -d \
  --name muse2api \
  --restart unless-stopped \
  -p 18610:18610 \
  --shm-size=2g \
  -e MUSE2API_HOST=0.0.0.0 \
  -e MUSE2API_PORT=18610 \
  -e MUSE2API_KEY=m2a_your_admin_key \
  -e MUSE2API_CONCURRENCY=2 \
  -v /opt/muse2api/data:/app/data \
  ghcr.io/qcjoyll/muse2api:latest
```

> 重建会打断正在跑的生成任务，升级后未完成的任务会被标记为失败，别挑高峰期。

不想每次抄一遍长命令，就存成 `/opt/muse2api/upgrade.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail
IMG=ghcr.io/qcjoyll/muse2api:latest
docker pull "$IMG"
docker rm -f muse2api 2>/dev/null || true
docker run -d --name muse2api --restart unless-stopped \
  -p 18610:18610 --shm-size=2g \
  -e MUSE2API_HOST=0.0.0.0 -e MUSE2API_PORT=18610 \
  -e MUSE2API_KEY="${MUSE2API_KEY:?请先 export MUSE2API_KEY}" \
  -e MUSE2API_CONCURRENCY=2 \
  -v /opt/muse2api/data:/app/data \
  "$IMG"
docker image prune -f
```

```bash
chmod +x /opt/muse2api/upgrade.sh
export MUSE2API_KEY=m2a_your_admin_key
/opt/muse2api/upgrade.sh
```

### 用 compose 也行

把 `image:` 指过来、删掉 `build:`：

```yaml
services:
  muse2api:
    image: ghcr.io/qcjoyll/muse2api:latest
    pull_policy: always
    container_name: muse2api
    restart: always
    ports:
      - "18610:18610"
    shm_size: 2gb
    environment:
      - MUSE2API_HOST=0.0.0.0
      - MUSE2API_PORT=18610
      - MUSE2API_KEY=${MUSE2API_KEY}
      - MUSE2API_CONCURRENCY=2
    volumes:
      - ./data:/app/data
```

```bash
docker compose pull && docker compose up -d
```

### 回滚到某个版本

```bash
docker pull ghcr.io/qcjoyll/muse2api:v1.6.10
# 把上面 docker run 最后的 tag 换成 :v1.6.10 重建即可
```

---

## 四、Windows / 本机临时跑

PowerShell 下卷路径要写绝对路径：

```powershell
docker run -d --name muse2api -p 18610:18610 --shm-size=2g `
  -e MUSE2API_HOST=0.0.0.0 -e MUSE2API_KEY=m2a_your_admin_key `
  -v "${PWD}\data:/app/data" `
  ghcr.io/qcjoyll/muse2api:latest
```

需要 Docker Desktop 且处于 **Linux 容器模式**（镜像是 `python:3.11-slim`，不是 Windows 镜像）。

---

## 五、常见问题

**1. `docker pull` 报 `denied` 或 `unauthorized`**
包还是 private。要么去 Package settings 改成 Public，要么 `docker login ghcr.io -u QCEnjoyLL -p <PAT>`（PAT 勾 `read:packages`）。

**2. 报 `invalid reference format`**
把用户名写成大写了。必须 `ghcr.io/qcjoyll/muse2api`，全小写。

**3. 国内拉 ghcr.io 很慢 / 超时**
给 Docker daemon 配代理，或在能访问的机器上 `docker pull` 后 `docker save` / `docker load` 带过去。长期方案是把镜像同步一份到阿里云 ACR 或 Docker Hub（需要给 workflow 加登录和第二次 push）。

**4. Actions 里每周自动构建没跑**
GitHub 会在仓库连续 60 天没有任何提交时自动停用定时任务，并给你发邮件。点 Actions 页面的 **Enable workflow** 恢复即可；也可以随时手动 Run workflow。这个仓库本来就不常提交，遇到这种情况是正常的。

**5. 容器起来但管理页打不开**
先看 `docker logs muse2api`；再确认 `-p` 端口和云厂商安全组都放行了 18610；如果 `MUSE2API_HOST` 没设成 `0.0.0.0`，宿主机访问不到。

**6. 生成图片/视频报 502（前面套了 Nginx）**
反代要放开长连接和缓冲：

```nginx
proxy_read_timeout 600s;
proxy_send_timeout 600s;
proxy_buffering off;
client_max_body_size 200M;
```

并把 `-e MUSE2API_PUBLIC_BASE=https://你的域名` 传给容器。

**7. 升级后管理页样式错乱**
强制刷新浏览器缓存（Ctrl+F5），上游每次发版都提醒这一点。

---

## 六、这个仓库怎么自动更新

`build-image` 每周一北京时间 09:00 从上游 `main` 拉最新代码构建，不需要你手动同步任何东西。上游改代码 → 下周一镜像自动是新的 → 你在服务器上跑一遍 `upgrade.sh` 即可。

想立刻出镜像：Actions → `build-image` → **Run workflow**。

想改频率：编辑 `.github/workflows/build-image.yml` 里的 `cron`，格式是 `分 时 日 月 周`，注意**用的是 UTC 时间**（北京时间减 8 小时）：

| 想要的效果 | cron |
|---|---|
| 每周一 09:00 北京时间 | `0 1 * * 1` |
| 每天 09:00 北京时间 | `0 1 * * *` |
| 每周一、四 09:00 | `0 1 * * 1,4` |

---

## 许可

上游为 MIT 许可。本仓库只做构建，源码版权归 [czg86389-hub/muse2api](https://github.com/czg86389-hub/muse2api) 所有。
