---
name: nat64-relay
description: 基于 Tailscale 控制面的 split DNS + NAT64 中继方案，使指定后缀域名强制走中继节点，而普通域名保持直连。
---

# NAT64 中继

## 何时使用
当你需要以下能力时使用本技能：
1. 普通主机名保持原有直连路径。
2. `*.<SPLIT_DOMAIN>` 强制经过中继节点。
3. 不维护每台主机的静态映射表。
4. 典型场景示例：
   - `ping my-vps`：目标主机在国外，你自己在国内，默认直连路径可用但性能一般。
   - `ping my-vps.9929`：通过 9929 中继节点转发后，链路质量更稳定、实际性能更好。
5. 在当作转发的机器上进行以下配置，网内其他节点均可使用。

## 输入参数
- `<SPLIT_DOMAIN>`: 需要中继的专用域后缀。
- `<EDGE_NODE_TS_IP>`: 中继节点的 Tailscale IP。
- `<UPSTREAM_DNS>`: 可解析真实目标主机的上游 DNS。
- `<REAL_FQDN_SUFFIX>`: 真实目标主机的域后缀。
- `<NAT64_PREFIX>`: 中继使用的 IPv6 前缀（建议 `/96`）。
- `<NAT64_V4_POOL>`: NAT64 动态 IPv4 地址池。
- `<OVERLAY_IFACE>`: Tailscale 网卡名（通常是 `tailscale0`）。
- `<ADVERTISED_PREFIX>`: 通过 Tailscale 发布的路由前缀（通常等于 `<NAT64_PREFIX>`）。

## 核心模型
1. 客户端查询 `host.<SPLIT_DOMAIN>`。
2. Tailscale split DNS 将该查询发送到 `<EDGE_NODE_TS_IP>`。
3. 中继 DNS 在内部查询 `host.<REAL_FQDN_SUFFIX>`（通过 `<UPSTREAM_DNS>`）。
4. 中继 DNS 直接返回位于 `<NAT64_PREFIX>` 内的最终 AAAA。
5. 客户端通过 Tailscale 下发的路由将 `<ADVERTISED_PREFIX>` 流量送到中继节点。
6. 中继节点执行 NAT64 翻译并转发到真实 IPv4 目标。

## 强约束
不要把 `<REAL_FQDN_SUFFIX>` 通过 CNAME/DNAME 暴露给客户端。  
对 `*.<SPLIT_DOMAIN>` 必须直接返回最终 AAAA，避免客户端绕过中继。

## 最小实施步骤（MVP）
1. 安装最小组件：`unbound`、unbound python 模块、`tayga`、`nftables`。
2. 配置 unbound 监听 `<EDGE_NODE_TS_IP>:53`。
3. 在 unbound 的 python 逻辑中：
   - 匹配 `*.<SPLIT_DOMAIN>`。
   - 解析 `host.<REAL_FQDN_SUFFIX>` 的 A（通过 `<UPSTREAM_DNS>`）。
   - 将 IPv4 嵌入 `<NAT64_PREFIX>`，合成并返回 AAAA。
   - 对 `*.<SPLIT_DOMAIN>` 的 A 查询返回空答案（或按你的策略处理）。
4. 配置 `tayga`：设置 `<NAT64_PREFIX>` 与 `<NAT64_V4_POOL>`。
5. 在启动流程中确保 `nat64` 接口和前缀/地址池路由自动创建。
6. 持久化开启转发：
   - `net.ipv4.ip_forward=1`
   - `net.ipv6.conf.all.forwarding=1`
7. 持久化 nftables 出口 SNAT：
   - `ip saddr <NAT64_V4_POOL> oif <OVERLAY_IFACE> masquerade`
8. 在中继节点执行 `tailscale set --advertise-routes=<ADVERTISED_PREFIX>`。
9. 在 Tailscale 管理台批准该路由。
10. 在 Tailscale DNS 中配置 split DNS：
   - domain: `<SPLIT_DOMAIN>`
   - nameserver: `<EDGE_NODE_TS_IP>`

## 验证清单
1. `host.<SPLIT_DOMAIN>` 返回 `<NAT64_PREFIX>` 内 AAAA。
2. 响应中没有泄露到 `<REAL_FQDN_SUFFIX>` 的 CNAME/DNAME。
3. 客户端路由表存在 `<ADVERTISED_PREFIX>`。
4. 中继节点可观测到 `host.<SPLIT_DOMAIN>` 的转发流量。
5. 非 split 域名仍保持直连解析与连接。
6. 重启后服务、路由、sysctl、防火墙规则自动恢复。

## 故障排查
1. `REFUSED`：中继 DNS ACL 未放行客户端来源网段。
2. 查询超时：中继 DNS 未监听 `<EDGE_NODE_TS_IP>`，或 split DNS 指向错误。
3. 解析正常但业务不通：`<ADVERTISED_PREFIX>` 未批准或未下发到客户端。
4. 单向通或不稳定：缺少出口 SNAT/masquerade。
5. TCP DNS 可用但 UDP 不通：UDP 53 被拦截或未代理。
6. 解析后绕过中继：返回了暴露真实后缀的 CNAME/DNAME。
