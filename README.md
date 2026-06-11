# frpc 出口节点(GitHub Actions 版 · 动态端口滚动池)

把 GitHub Actions 的 runner 当 frpc 节点:每个 runner 一个独立出口 IP,接入服务器的 frps,组成出口池。配合 `../server` 的 mihomo + hy2,客户端随机走这些 IP 出网,并且 IP 会持续轮换。

> GitHub runner 是 Azure 数据中心 IP,不是住宅 IP。目标站点若封 Azure 段,请改用 `../node` 的住宅节点。

## 这版怎么消除"换批空窗"

旧版每个节点端口写死(node1→20001),两批一重叠就撞端口,只能靠互斥锁串行 → 换批必有空窗。新版改成:

- **动态抢空端口**:每个 runner 启动时在端口池里随机挑一个,先用 TCP 预检(连得上=被占用,换一个),再用 `frpc status` 确认抢到了才留下。两批可以安全重叠,不再撞端口。
- **服务端开大端口池**:frps 开 `POOL_SIZE`(默认 60)个端口,mihomo 把这 60 个全列进 load-balance 组,健康检查间隔短(默认 30s)→ **新端口自动纳管、死端口自动剔除**,无需手工维护节点列表。
- **错峰退出 + 滚动接力**:19 个节点不再同时死,而是按 `instance` 依次多活几秒。relay 提前触发下一轮,下一轮的 job 排队等本轮节点错峰腾名额时逐个补位 → 滚动替换,池子基本不空。

### 绕不开的硬限制

GitHub 免费账号**同一时刻最多 20 个并发 job**(整账号共享)。所以无法让"完整一批新节点"和"完整一批旧节点"同时在线(那是 40 个 job)。本设计用"错峰退出 + 逐个补位"在 20 名额内尽量贴合,空窗压到最小,但不是数学意义的零。要更稳就把 `RUN_SECONDS` 调大(换批没那么频繁,空窗占比更低)。

## 关键参数(在 `frpc-node.yml` 的 `env` 里)

| 参数 | 默认 | 说明 |
|---|---|---|
| `POOL_SIZE` | 60 | 端口池大小,**必须和服务器 install_server.sh 的 POOL_SIZE 一致** |
| `RUN_SECONDS` | 300 | 单个 runner 基础寿命(5 分钟)。调小=换 IP 更勤但空窗更频繁 |
| `STAGGER_SECONDS` | 8 | 错峰步长,节点退出分散开的间隔 |
| `BASE_PORT` | 20000 | 端口池起点,与服务器一致 |
| matrix `instance` | 1..19 | 节点数,19 + 1 relay = 20 占满并发上限 |

## 部署步骤

### 1. 服务器(开大端口池)

```bash
cd ../server
sudo TOKEN=你的强token HY2_PASSWORD=你的hy2密码 HY2_PORT=443 \
     POOL_SIZE=60 HEALTH_INTERVAL=30 STRATEGY=round-robin \
     bash install_server.sh
```

`POOL_SIZE` 要 ≥ 峰值在线节点数,给新旧重叠留余量(19 节点用 60 很宽裕)。改完确认 mihomo 起来:`systemctl status mihomo`。

### 2. 新建 GitHub 仓库并推送

把本目录(含 `.github/workflows/frpc-node.yml`)推到一个新仓库:

```cmd
cd /d d:\desktop\zip_main_local\frp-hy2-pool\node-github-action
git init
git add .
git commit -m "frpc dynamic-port rolling pool"
git branch -M main
git remote add origin https://github.com/你的用户名/frpc-nodes.git
git push -u origin main
```

### 3. 配置仓库 Secrets

仓库 → Settings → Secrets and variables → Actions:

| Secret | 值 |
|---|---|
| `FRP_SERVER_ADDR` | frp 服务器公网 IP |
| `FRP_TOKEN` | 与服务器 `TOKEN` 一致 |
| `PAT` | 有 `repo` + `workflow` 权限的 Personal Access Token |

### 4. 启动

Actions → "frpc exit nodes" → Run workflow 手动跑一次,之后靠 relay 自接力。链路若意外全断,`schedule` 看门狗(每 15 分钟)会在确认没有任何在跑/排队的 run 时重新播种。

## 验证

服务器上:

```bash
journalctl -u frps -f          # 看 exit-200xx 端口不断有节点上线/下线
ss -ltnp | grep -c ':200'      # 当前在线的出口端口数,应稳定在 ~19 上下
```

客户端多次请求,出口 IP 既在多个节点间轮询,又随时间不断换新:

```bash
curl -x socks5h://127.0.0.1:1080 https://api.ipify.org
```

## 注意

- **费用**:19 个并行 runner + 5 分钟一轮,Actions 分钟数消耗快。私有仓库会很快吃光免费额度,**建议放公开仓库**(Actions 免费不限量),但公开仓库绝不能把密钥写进代码,全部放 Secrets。
- **换批瞬间**:可能有极少数请求落到刚退出的节点上失败,客户端侧做重试即可(`round-robin` 只用当前在线节点,不会整体中断)。
- 长时间占用 runner 跑代理属灰色用法,可能触发 GitHub 滥用检测,自行评估。
