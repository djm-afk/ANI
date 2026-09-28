# INSTANCE-VM-VOLUME-ATTACH-A

> **日期**：2026-09-28　**分支**：`hotfix/volume-block-mode`（worktree `ani-hotfix-volume-block`）　**PR**：待开　**状态**：live verified（隔离测试环境 ani-test2，镜像 `test2-20260928-volumeblock`）
> **性质**：Feature batch（VM 块存储卷挂载链路真实底座修复）
> **来源**：云盘挂载故障排查（实例 `inst_cf091602…` / 卷 `vol_39e19a6a…`）；`特有Bug修复问题清单.md` VM-05 遗留项"Filesystem 模式卷热插缺少 disk.img，需支持 Block 模式卷"

## 背景

给运行中的 VM 挂载块存储卷（云盘）后，卷状态长期停在 `pending`，盘始终进不了 guest。排查确认**K8s 侧 PVC 实际已 Bound，卡住的是 KubeVirt 热插这一步**；`pending` 是控制面记录陈旧。

实测现场（生产 ani-system，租户 `02779ed7…`，VM `testvm`，卷 `vol_39e19a6a…`）：

- PVC `vol-vol-39e19a6a-…` = **Bound**（10Gi / RWO / SC `ani-block` / `volumeMode: Filesystem` / rook-ceph RBD）；
- VM spec 该卷为 `persistentVolumeClaim` + **`hotpluggable: true`**，VMI volumeStatus 停在 `AttachedToNode`，始终不到 Ready；
- virt-handler 在 115 分钟内重复报 **6394 次**：`failed to mount filesystem hotplug volume …: lstat …/disk.img: no such file or directory`；
- 附件 Pod 内卷已挂载（`/dev/rbd8` ext4），但**只有 lost+found，没有 `disk.img`**；
- ANI 库 `storage_volumes`：`state=pending`、`reason="observed Kubernetes PVC phase Pending"`、`mount_instance_id` 空、`updated_at == created_at`（创建后从未再观测）。

## 根因

### 1. 功能层：Filesystem 模式卷无法通过 KubeVirt 热插挂载

- ANI 把块存储卷硬编码渲染为 `volumeMode: Filesystem`（`storage_renderer.go`）；
- VM 挂盘走 KubeVirt `addvolume` 子资源（`kubernetes_lifecycle_executor.go`）；
- KubeVirt v1.8.2 对 **Filesystem 卷热插**要求卷内**已存在** `disk.img`（`pkg/virt-handler/hotplug-disk/mount.go` 把 `<卷挂载点>/disk.img` bind 给 QEMU），新分配的空盘必然没有该文件 → 永久失败；
- 而 **非热插**（VM 启动路径）下 KubeVirt 会**自动创建** disk.img：`pkg/host-disk` 的 `ReplacePVCByHostDisk` 把 Filesystem PVC 换成 HostDisk，`DiskImgCreator.Create → createSparseRaw` 按 PVC 容量生成稀疏镜像。这是本批次修复方案的依据。

### 2. 展示层：控制台为什么一直显示 pending

- `ani-block` 是 `WaitForFirstConsumer`，PVC 只有被消费 Pod 挂载后才 Bound；创建瞬间观测到 Pending 即被记为 `pending`；
- 卷状态只在**读详情**时 re-observe（`LocalStorageService.GetVolume`，30s 节流），**列表不刷新**（`ListVolumes` 直接返回存量记录）→ 列表长期显示 pending；
- 热插始终失败又使 `mount_instance_id` 永远填不上（控制台据此判断"已挂载"）。

### 3. 方案取舍（为何不是"默认改 Block"）

"卷支持 Block 模式"是根治方向（Block 卷走裸设备热插，不需要 disk.img），集群侧已实测具备条件（Block PVC Bound / PV `volumeMode: Block` / 容器内出现裸设备 `/dev/<name>`，**不需要运维改集群**）。但当前实测 **13 个容器/GPU 容器实例正在把块存储卷按目录挂载**（`MountPath:/data` 形态，`Kind: shared_pvc`），容器目录挂载必须 Filesystem，因此把创建默认值改为 block 会打断既有可用流程；且 `volumeMode` 不可变、存量卷只能重建。故本批次先按"VM 挂盘不经热插"落地，`volume_mode` 契约化作为后续独立批次。

## 修复内容

### VM 挂/卸云盘改为"停机 → 重写 VM spec → 开机"（提交待定）

- **`applyKubeVirtVolume` 不再使用 `addvolume`/`removevolume` 子资源**，改为：running → stop → 等停稳 → GET VM spec → 重写 `spec.template.spec.volumes` 与 `domain.devices.disks` → server-side apply（`force=true`）→ start；VM 原本停机则只重写 spec 不启停。
- **新增 `applyKubeVirtVolumeMutation`**：按设备名替换既有条目（幂等），写入形式为普通 `persistentVolumeClaim` + virtio `disk`，**刻意不带 `hotpluggable`**——这是本修复的关键：KubeVirt 的 `shouldSkipVolumeSource` 会跳过带热插标记卷的 disk.img 创建，保留该标记等于没修。
- **detach 同路径**：removespec 中该卷的 volumes 与 disks 条目。
- **抽取公共辅助 `applyKubeVirtVMSpecMutation`**：把原先内联在 `applyKubeVirtFilesystem` 中的"停机/重写/开机/失败 best-effort 恢复开机"流程提为共用实现，卷与共享文件系统两条路径共用，行为与错误文案保持一致。
- **存量坏条目自愈**：`kubeVirtVolumeName` 对现场 VM 返回的卷名与旧 `addvolume` 写入的条目名一致（实测 `volume-vol-39e19a6a-…`），下次 attach/detach 的按名替换即可清掉遗留的 `hotpluggable` 条目，无需手工 `removevolume`。

### 已记录但未在本批次修复（独立问题）

- 控制台卷列表对 pending 卷不做 re-observe（小事，可让列表也 re-observe）；
- `CreateStorageVolumeRequest.storage_class` 契约默认值是集群不存在的 `standard`（现存 4 个 `standard` 类 Pending PVC 由此产生），默认值应改 `ani-block`。

## 变更文件

- `repo/pkg/adapters/runtime/kubernetes_lifecycle_executor.go`：`applyKubeVirtVolume` 改走 spec 重写；新增 `applyKubeVirtVolumeMutation`、`applyKubeVirtVMSpecMutation`；`applyKubeVirtFilesystem` 复用公共辅助
- `repo/pkg/adapters/runtime/kubernetes_lifecycle_executor_test.go`：两个断言旧 `addvolume`/`removevolume` 的用例改为断言 stop→PATCH→start 序列，并显式断言 manifest 不含 `hotpluggable`
- 未改：Core OpenAPI 契约、SDK/生成物、DB 迁移、Services 层、前端

## 验证汇总

**单元测试**：`go build ./...`（pkg 与 ani-gateway 两模块）、`go test ./adapters/runtime/...`（除 Windows 本机既有限制 `TestSandboxFileScriptsRejectSymlinks`（无 symlink 权限）、`TestSandboxFileScriptsAllowWorkspaceOperations`（无 python3）外全过）；`gofmt -l` 改动文件无输出、`git diff --check` 干净。

**架构/契约门禁**（本机无 `make`，等价脚本直跑）：`validate_component_imports`、`validate_inference_legacy_control_plane`、`validate_auth_gateway_contract`、`generate_gateway_authz_test`（18 tests OK）、`validate_gateway_authz_drift`（no drift）、`validate_core_gateway_authz_routes`（325 routes / 0 error）全部通过。

**真实环境 live 验证 PASS**（ani-test2，`10.10.1.66:30083`，镜像 `docker.changqingyun.cn/ani/ani-gateway:test2-20260928-volumeblock`，只 `kubectl set image` 未改 env，rollout 成功、healthz 200）：

| 检查项 | 修复前（生产现场） | 修复后（ani-test2 实测） |
|---|---|---|
| ANI 卷状态 | `pending`（observed Kubernetes PVC phase Pending） | `available` → 挂载后 `mounted by local storage profile` |
| `mount_instance_id` | 空 | 写入 `inst_b07922a1…` |
| VM spec 卷条目 | PVC + **`hotpluggable: true`** | PVC（**无 hotpluggable**）+ virtio disk |
| virt-handler | `HotplugFailed` ×6394 / 115 分钟 | **无 HotplugFailed** |
| KubeVirt 事件 | — | `ShuttingDown` → `Stopped` → **`ToleratedSmallPV`** → `Started` |
| 卷内 `disk.img` | 不存在 | **存在**：`-rw-r--r-- qemu qemu 5179449344 disk.img` |
| VM 状态 | Running（盘不可用） | `running → provisioning → running` |

`ToleratedSmallPV` 事件（"PV size too small: expected 5368709120 B, found 5179449344 B，within 10% toleration"）即 KubeVirt `ReplacePVCByHostDisk → createSparseRaw` 路径运行的指纹，证明"启动时自动创建 disk.img"机制被真实触发。

**卸载路径**同样验证：`detach_volume` → VM 再次 `running → provisioning → running`，spec 中该卷 volumes/disks 条目清零（实测残留条目数 0），卷状态 `unmounted by local storage profile` 且 `mount_instance_id` 清空。

## 遗留

- **Block 模式卷（VM-05 遗留）未做**：集群侧已验证支持（见背景 §3），需要契约新增 `volume_mode`、DB 列与迁移、渲染层按模式输出，并按消费方校验配对（VM ↔ block、容器 ↔ filesystem）；`volumeMode` 不可变，存量卷需删旧重建。
- **每次挂/卸云盘都会重启 VM**（本方案固有代价）：属产品可见行为变化，在线挂盘能力仍需 Block 方案才能达成。
- **卷删除不级联删 provider PVC**：`DELETE /volumes/{id}` 成功、控制面记录消失，但租户命名空间内 PVC 仍 Bound（本次测试对象由人工清理）。因 SC 为 `Retain`，删 PV 后 Ceph 池内会留下孤立 RBD 镜像（本次遗留一个 5Gi 稀疏镜像，环境内无 ceph CLI 权限无法回收）。
- **卷侧"已挂载"由独立接口写入**：实例 `attach_volume` 不写 `mount_instance_id`，需控制台额外调 `POST /volumes/{id}/mount`，即控制台完整挂载需两个接口配合；两侧语义合并属长期收敛项。
- 控制台卷列表 stale pending 与契约 `storage_class` 默认值两项小问题按上文"已记录未修"处理。