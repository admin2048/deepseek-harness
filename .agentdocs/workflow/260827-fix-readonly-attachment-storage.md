# 修复只读容器附件存储

## 现状

YQZ 运维助手复用 DSH Base/Web 的图片附件能力，并在 `ReadonlyRootfs=true` 的容器中运行。部署仅将 `$DSH_HOME/attachments` 子目录挂载为可写；`attachment-local` 首次保存图片时仍会把 `$DSH_HOME` 本身作为待创建和同步的目录处理，因而触碰只读镜像层。该异常发生在稳定 `AttachmentError` 包装之前，上层将它误报为 `prompt rejected (agent-busy)`。

## 修复边界

- DSH 附件后端支持部署方显式提供一个已经持久化、独立可写的附件存储根，不要求扩大整个 `$DSH_HOME` 的写权限。
- 默认本机行为继续使用 `$DSH_HOME/attachments/v1`，并保留原有逐级目录同步保证。
- 存储根准备阶段的文件系统失败统一转换为 `ATTACHMENT_WRITE_FAILED`，使 Web RPC 返回 `attachment-error`。
- YQZ Profile 显式接入独立附件根；Compose 与控制面继续保持只读根、非 root 用户、最小挂载和其他安全约束。
- 本阶段不修改线上容器；构建和受管发布需要单独确认。

## 验收条件

- [x] 回归测试在旧实现上稳定失败，证明只读父目录与独立可写附件根的场景已被覆盖。
- [x] 默认 `$DSH_HOME` 路径的持久化顺序与已有测试保持不变。
- [x] 显式附件根不会尝试修改或同步其只读父目录，图片能够持久化并读取。
- [x] 根目录准备失败返回 `ATTACHMENT_WRITE_FAILED`，不再落入 `agent-busy`。
- [x] YQZ 组合测试验证 Profile 将 `attachment-local` 指向现有附件挂载；真实部署保存留在发布后的三重验收。
- [x] DSH 聚焦测试、类型检查、文档检查和差异检查通过。
- [x] YQZ 聚焦测试、类型检查和部署合同检查通过。
- [ ] 获得发布授权后，仅重建 `dsh-web` 并通过真实页面图片发送、Session 事件和附件对象三重验收。

## 阶段状态

- [x] 根因定位：图片在持久化前失败；模型、格式、大小与会话忙状态已排除。
- [x] 原版与 YQZ 部署差异：原版使用完整可写 `$DSH_HOME`，YQZ 使用只读根加可写子目录。
- [x] DSH RED 测试。
- [x] DSH GREEN 实现。
- [x] YQZ 接线与组合测试。
- [x] 文档与 Agent Note。
- [x] 本地验证。
- [ ] 受管发布与真实验收。
