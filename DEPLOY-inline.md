# 部署:内联节点模式的 SubConverter-Extended(适配 FlClash)

本仓库是 [Aethersailor/SubConverter-Extended](https://github.com/Aethersailor/SubConverter-Extended) 的补丁版,新增 `inline=true` 请求参数:让 `url=` 里的 http 订阅被**内联解析成节点**而不是生成 `proxy-providers`。这样输出的 Clash 配置自带 `proxies`,FlClash 等不解析 `use:` 引用的客户端能直接显示和选择节点。

## 1. 服务端部署(Docker)

```bash
docker compose -f docker-compose.inline.yml up -d
```

前置准备:

1. 一个公网可达域名(例如 `sub.example.com`)指向这台 VPS,配好 HTTPS(Caddy/Nginx 反代到 `127.0.0.1:25500`)。
2. 编辑 `docker-compose.inline.yml`,把 `MANAGED_CONFIG_PREFIX` 改成你的公网域名(否则生成的 rule-provider 指向 `127.0.0.1`,客户端拉不到规则)。
3. 镜像来自 `ghcr.io/D3-3109/subconverter-inline:master`,由 GitHub Actions 在每次 push 到 master 后自动构建。

**可选**:把一份 `pref.toml` 挂到 `/base/pref.toml` 做自定义(端口、规则缓存等);不挂则用镜像内置默认。

## 2. 订阅 URL(给使用者)

FlClash → 配置 → 导入 → 从 URL 导入,填:

```
https://sub.example.com/sub?target=clash
  &inline=true
  &url=<URLEncode(你的订阅或自建节点列表)>
  &config=<URLEncode(你的 ACL4SSR 配置 ini)>
  &scv=true&new_name=true&emoji=true
```

关键点:
- `inline=true` — 触发内联节点解析(本补丁新增)。
- `url=` 可以是你的 subhub 订阅、单个节点列表、或多个用 `,`/`|` 分隔的 URL。
- `config=` 指向你的 `ACL4SSR_ProxyBili.ini` 之类的规则模板。

输出是一份自包含 yaml:`proxies`(完整 anytls/ss 字段)+ `proxy-groups`(节点名直接列出)+ 内联或外部 `rule-providers`(取决于 `MANAGED_CONFIG_PREFIX`)。

## 3. 为什么这样部署而不是改客户端 / 换转换器

- FlClash 内核是 vendored mihomo,支持 anytls,但 UI 不渲染 provider(`use:`) 节点。
- 上游 subconverter 解析 anytls 只保留 password+sni,会丢 `jls-opts`/`idle-session-*`/`client-fingerprint` → 自建节点认证失效。
- 本 fork 保留了 anytls 全字段;补丁只加了一个开关让订阅走已有的 `addNodes` 内联解析路径,不破坏原有 provider 行为(`inline` 不传时一切照旧)。

## 4. 升级流程

```bash
cd 你的仓库
git remote add upstream https://github.com/Aethersailor/SubConverter-Extended.git
git fetch upstream
git merge upstream/master        # 解决可能的冲突(补丁点很小)
git push origin master           # Actions 自动重建镜像
# 服务器上
docker compose -f docker-compose.inline.yml pull && docker compose -f docker-compose.inline.yml up -d
```

## 5. 安全提醒

- 订阅 token 会出现在 `url=` 查询串里,务必走 HTTPS。
- 建议给 subconverter 加访问限制:反代层加 Basic Auth / 只允许你妹妹的 IP,或在 `pref.toml` 里限制可拉取的 host,避免被当成公开转换服务滥用。
