# QUOTA-READ-REPAIR-A — GPU 配额读修复旁路闭环（read-repair 委托 ReconcileNow，TCC + outbox 同事务）

> 日期：2026-09-28　分支：`hotfix/metering-gpu-lifecycle-events`（叠加于 METERING-LIFECYCLE-EVENTS-A / PR #191 之上）
> 状态：live verified（ani-test2 30083，镜像 `test2-20260928-quota-readrepair`）
> 关联：闭合 METERING-LIFECYCLE-EVENTS-A 遗留项「创建 confirmed 事件在详情轮询场景被 live 状态合成抢先落库绕过」

## 1. 用户报障

1. BOSS 台账 `/api/v1/quotas`：租户 `metering-e2e-20260924113158` 的「处理中」（`resource_quota.reserved`）恒为 1，已用数与真实不符。
2. `/api/v1/gpu-inventory`：实例占用对象不对、租户占用回显不对。

## 2. ani-system 实测根因链（2026-09-28）

### A. Confirm 链路断裂（部署面 + 结构面）

- gateway `GPU_QUOTA_ENABLED=true` → 创建时三道闸 + Try 正常（`resource_reservations` 写入）；
- 但 reconcile 进程缺 quota 装配（ani-reconcile-worker 无 `GPU_QUOTA_ENABLED`，镜像 `model-repository-live-20260902`）→ Confirm/Cancel/Release/selfHeal 全旁路；
- 数据库证据：泄漏流水 `743a3dd8`（metering-e2e 租户）对应实例 `inst_a32e10a2` **一直 running**，`expires_at` 过期 3 天 17 小时无人清扫；两条 confirmed 流水时间戳精确同毫秒（08:47:07.334263）= e2e 手工补 Confirm，之后创建的实例没人补；
- 全平台 `state='reserved'` 泄漏 2 条（metering-e2e、tenant-qa-full-001），**已数据修复**（流水转 confirmed + 回填 resource_ref + reserved→used 转账，修复后全平台 reserved 泄漏清零）；
- reconcile 循环对存量实例大量报 `audit id is required before status reconcile`（旧 sandbox 记录 `audit_id` 为空），`selfHealConfirm` 无法触达。

### B. 读修复旁路（本批次代码修复核心）

gateway 路由层 `refreshOneStoreStatus` / `refreshOneVMStoreStatus`（GET/list 的 live 状态回读）用**裸 `UpsertStatus`** 落库观测状态：

- 绕过 TCC（无 Confirm/Cancel/Release）、绕过 outbox（metering 无事件）；
- reconcile 循环 `ListReconcileTargets` 只捞 `updated_at` 早于 `StaleThresholdSeconds` 的实例，而 Console/BOSS 持续轮询 GET/list → `updated_at` 不断刷新 → 实例**永远不进 reconcile 列表** → 两条收口路径全部失效；
- ani-test2 复现（PR #191 镜像 + `GPU_QUOTA_ENABLED=true`）：带 `spec_id` 的 GPU 实例 Try 正常，running 后 `used=0 / reserved=1` 停留，该实例 outbox 0 条；删除仅触发 Cancel（流水 cancelled、reserved 回 0），Confirm 全程未发生；
- 这同时解释 A 中 metering-e2e 实例「running 但预占停留 reserved」：详情轮询的 live 状态合成抢先落库。

### C. gpu-inventory 占用口径（实测确认，未在本批次修，待独立批次）

1. 占用标记按设备索引顺序猜（`planning` 未持久化 device index），多实例共节点时 `instance_id` 取字典序最小——回显对象本质任意；
2. 平台 token 的租户上下文 fallback `demo-tenant`，真实租户的 GPU Pod 不参与平台视角占用标记；
3. 不过滤非 GPU Pod：labelSelector 只有 `ani.kubercloud.io/tenant-id`，CPU 实例/Kubeflow Pod 一样计 `PodCount`（vGPU 资源表达式是 `volcano.sh/vgpu-number`，非 `nvidia.com/gpu`）；
4. 1 Pod = 1 设备记录假设对多卡 Pod 少算。

## 3. 修复（方案 A：读修复委托 ReconcileNow）

- `router.InstanceRuntime` 新增 `ReconcileController`；gateway `main.go` 注入 `instanceRuntime.ReconcileController`（`ConnectInstanceService` 已装配，`GPU_QUOTA_ENABLED=true` 时带 quotaService）；
- 新增 `instanceAPI.commitReadRepairTransition`：read-repair 检测到 `previous != next` 且 `QuotaTxIDs` 非空时，调 `ReconcileNow`（reconcile 控制器 `applyStateTransition` 矩阵：provisioning→running=Confirm、→failed=Cancel+Release、deleting→deleted=Cancel+Release，且 outbox 事件同事务写入——完整复用 METERING-LIFECYCLE-EVENTS-A 语义，不重复实现）；委托失败仅 WARN 并回退裸写（不阻塞读路径）；
- `refreshOneStoreStatus`（Deployment 消失→failed 路径 + 正常观测路径）与 `refreshOneVMStoreStatus` 三处接入；委托后仍执行原有 display 合并落库（node/ip/access 字段），行为向后兼容；
- 无 QuotaTxIDs 的实例零行为变化（非配额实例 / `GPU_QUOTA_ENABLED=false` 前创建的实例）。

单测 +4（`instances_test.go`，stub `ports.WorkloadReconcileController`）：QuotaTxIDs 委托断言（含 previous 状态）、无 QuotaTxIDs 不委托（回归守护）、Deployment 消失→failed 委托、VM 路径委托。

## 4. 门禁

`go vet ./services/ani-gateway/...` 全绿；`go test ./services/ani-gateway/...` 全绿（router 包 14 个 refresh 系用例含 4 新增）；`gofumpt -w` 两文件；`make test` / `make validate-architecture` / `git diff --check` 提交前统一执行。

## 5. live 验证（ani-test2 30083）

镜像 `test2-20260928-quota-readrepair`（PR #191 六层修复 + 本批次，构建机独立目录 `/root/ani-build/quotarr-20260928`）+ `GPU_QUOTA_ENABLED=true`（部署沿用；rollout 期间数据库多连接池慢启动致探针重启 6 次后稳定，`/healthz` 200）。

vGPU 切片规格 `rtx4090-12g-4` 实例 E2E：

| 阶段 | `resource_reservations` | `resource_quota`（gpu_count） | outbox |
|---|---|---|---|
| 创建后 | `a5b3e7e6 reserved` | used=0 / reserved=1 | — |
| **GET 轮询见 running（修复点）** | `a5b3e7e6 confirmed`（resource_ref 回填） | **used=1 / reserved=0** | `instance.confirmed published=t` |
| 删除后 | `a5b3e7e6 released` | used=0 / reserved=0 | `instance.deleted published=t` |

对照修复前同环境同用例：running 后 used=0/reserved=1 停留、outbox 0 条、删除仅 Cancel。测试实例已清理，tenant-a 台账归零，无泄漏流水。

## 6. 遗留

1. C（gpu-inventory 占用口径四缺陷）未修，待独立批次（精确到卡需 planning 持久化 device index 或读 device plugin 分配信息）；
2. ani-system 未部署本批次镜像（当前被 `emailfix-20260928` 改动线覆盖，需协调并集）；reconcile-worker 仍缺 `GPU_QUOTA_ENABLED` 与新镜像；
3. `PROVISIONING_TIMEOUT_MIN` 未配置时代码不启用超时兜底（`cfg.ProvisioningTimeoutMin > 0` 才挂 `WithProvisioningTimeoutMin`），provisioning 卡死无保护，建议两环境配置；
4. 存量实例 `audit_id` 为空导致 reconcile 校验失败（旧 sandbox 记录），未处理；
5. 预留 10 分钟 TTL 仍无清扫者（`expires_at` 无 sweeper），悬挂预留依赖人工/超时兜底。
