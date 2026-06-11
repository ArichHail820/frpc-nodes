# frpc 出口节点(GitHub Actions 版)

把 GitHub Actions 的 runner 当 frpc 节点用:每个 runner 一个独立出口 IP,接入你服务器的 frps,组成出口池。配合 `../server` 的 mihomo + hy2,客户端就能随机走这些 GitHub IP 出网。

> 注意:GitHub runner 是数据中心 IP(微软 Azure 段),不是住宅 IP。适合需要"干净/可轮换数据中心 IP"的场景。若目标站点封 Azure 段,请用 `../node` 的住宅节点。

## 工作原理

- `matrix.instance: [1..N]` 开 N 个并行 runner,每个是独立 IP。
- `instance=K` 的节点占用服务器 frps 的 `2000K` 端口,对应 mihomo 里的 `nodeK`。
- 每个 runner 跑 `RUN_MINUTES` 分钟后优雅退出,`relay` 链式触发下一轮 → 换新 runner = 换新 IP。
- `concurrency` 互斥保证同一时刻只有一轮,避免端口冲突。

## 部署步骤

### 1. 先装好服务器

按 `../server/README` 装好 frps + mihomo,**`NODE_COUNT` 要等于这里 matrix 的实例数**(默认 5):

```bash
sudo TOKEN=你的frp令牌 HY2_PASSWORD=你的hy2密码 HY2_PORT=443 NODE_COUNT=5 bash install_server.sh
```

### 2. 新建一个 GitHub 仓库

把本目录的 `.github/workflows/frpc-node.yml` 放进去(路径保持 `.github/workflows/frpc-node.yml`),推上去。

### 3. 配置仓库 Secrets

仓库 → Settings → Secrets and variables → Actions → New repository secret,添加:

| Secret 名 | 值 |
|---|---|
| `FRP_SERVER_ADDR` | 你的 frp 服务器公网 IP |
| `FRP_TOKEN` | 与服务器 `install_server.sh` 里 `TOKEN` 一致 |
| `PAT` | 一个有 `repo` + `workflow` 权限的 Personal Access Token(给 relay 自触发用) |

> `PAT` 创建:GitHub 头像 → Settings → Developer settings → Personal access tokens。经典 token 勾选 `repo` 和 `workflow` 即可。

### 4. 启动

仓库 → Actions → 选 "frpc exit nodes" → Run workflow 手动跑一次,之后会靠 relay 自己接力维持。

## 验证

- 服务器上 `journalctl -u frps -f`,能看到 `node1-socks`...`nodeN-socks` 陆续上线。
- 客户端连 hy2 后多次请求,出口 IP 会在这些 GitHub runner IP 间轮换。

## 调参

在 `frpc-node.yml` 里改:

- **节点数**:改 `matrix.instance` 列表(同时同步服务器 `NODE_COUNT`)。
- **换 IP 频率**:调小 `RUN_MINUTES`(配套 `timeout-minutes` 略大于它)。换得越勤 IP 轮换越快,但中间空窗也更频繁。
- **接力频率**:`schedule` 的 cron 只是兜底,主力是 relay。

## 注意

- GitHub Actions 有用量配额(免费账号每月有限 minutes;公开仓库不限)。N 个实例 × 长时运行会很快吃配额,留意账单。建议放**公开仓库**用免费额度,但公开仓库别把任何敏感信息写进代码(都放 Secrets)。
- 这种长时占用 runner 跑代理属于灰色用法,可能触发 GitHub 的滥用检测。自行评估风险。
