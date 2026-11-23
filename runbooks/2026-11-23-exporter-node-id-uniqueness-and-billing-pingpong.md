# Infra Runbook: `EXPORTER_NODE_ID` 唯一性约束与重复标识引发账单乒乓循环处置

- **日期**: 2026-11-23
- **所属组织**: `ai-workspace-infra`
- **关联代码 PR**:
  - [`ai-workspace-infra/playbooks#1046`](https://github.com/ai-workspace-infra/playbooks/pull/1046) (`roles/vhosts/xray-exporter` 强制 `EXPORTER_NODE_ID == AGENT_PROXY_DOMAIN` 断言与变量绑定)
  - [`ai-workspace-xstream/xconnect-edge-agent#167`](https://github.com/ai-workspace-xstream/xconnect-edge-agent/pull/167) (`setup-observability.sh` 显式传入 `agent_proxy_domain` 与 `xray_exporter_node_id`)
- **关联服务与数据文档**:
  - `ai-workspace-services/knowledge`: `docs/02-accounts/Runbook/Fix-Duplicate-Node-ID-Billing-Pingpong.md`
  - `ai-workspace-xstream/docs`: `incidents/2026-11-23-duplicate-exporter-node-id-billing-pingpong.md`

---

## 1. 背景与约束

`ai-workspace-infra/playbooks` 保持纯粹的可复用 Ansible 标准 roles 结构；运维故障案例与跨仓库治理记录集中归档于各 ORG 文档仓库。

在 2026-11-23 的生产排查中发现：`jp-xconnect-01.svc.plus` 上的 `/etc/xray-exporter.env` 与 `/etc/systemd/system/xray-exporter-{xhttp,tcp}.service` 错误保留了 `EXPORTER_NODE_ID=jp-xconnect-02.svc.plus`，导致两台物理节点向 `billing-service` 推送相同的 `(prod:jp-xconnect-02.svc.plus-xhttp, account_uuid)` checkpoint 主键，触发每分钟重置并全量重扣差值的乒乓循环（24h 虚增至 `75.82 GiB`，本月虚增至 `482.84 GiB`）。

---

## 2. Ansible / 基础设施侧强制校验规则（`playbooks#1046`）

1. `roles/vhosts/xray-exporter/defaults/main.yml`、`deploy_observability_agent.yml`、`deploy_xray_exporter.yml` 优先从 `agent_proxy_domain` / `AGENT_PROXY_DOMAIN` 绑定 `xray_exporter_node_id`。
2. `roles/vhosts/xray-exporter/templates/xray-exporter.env.j2` 同步写入 `AGENT_PROXY_DOMAIN` 与 `EXPORTER_NODE_ID`，作为配置对比锚点。
3. `roles/vhosts/xray-exporter/tasks/main.yml` 在渲染 `.env` 前强制执行断言：
   - `xray_exporter_node_id` 非空且不得为 `localhost` / `localhost.localdomain` / `127.0.0.1`；
   - 若存在 `AGENT_PROXY_DOMAIN`，则 `xray_exporter_node_id` 必须与其完全相等。

---

## 3. 全量节点巡检命令（只读验证）

```bash
for host in \
  jp-xconnect-01.svc.plus \
  jp-xconnect-02.svc.plus \
  jp-xconnect-free-01.svc.plus \
  ph-xconnect.svc.plus \
  us-xconnect-los-angeles-custom-01.svc.plus \
  us-xconnect.svc.plus; do
  echo "=== $host ==="
  ssh -o ConnectTimeout=8 root@"$host" '
    grep -E "^(EXPORTER_NODE_ID|AGENT_PROXY_DOMAIN|EXPORTER_ENV)=" /etc/xray-exporter.env
    grep -h "ExecStart=" /etc/systemd/system/xray-exporter*.service
    systemctl is-active xray-exporter-xhttp xray-exporter-tcp vector
  '
done
```

预期标准：每台主机的 `AGENT_PROXY_DOMAIN`、`EXPORTER_NODE_ID` 与 systemd `--node-id` 前缀三者严格一致且全局唯一。

---

## 4. 节点迁移与重装操作纪律

1. 严禁把一台节点的 `/etc/xray-exporter.env` 或 `/etc/systemd/system/xray-exporter*.service` 原封不动复制到新节点。
2. 新旧节点切换同一域名时，**必须先在旧节点停止并禁用 `xray-exporter-*` 与 `vector` 后再启用新节点导出器**，避免双活期写入同一 `traffic_stat_checkpoints` 主键。
3. 若曾在生产数据库产生异常污染，操作前须在 `postgresql-prod` 保留备份表（例如本次 `public.traffic_minute_buckets_backup_20261123`），并在单事务中完成历史桶清洗与 `account_quota_states` 重算。
