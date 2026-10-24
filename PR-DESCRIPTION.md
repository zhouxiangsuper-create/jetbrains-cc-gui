# fix: 避免 Claude 未启用时查询 provider 重复写配置并引发启动报错

## 问题与原因

在配置已初始化、`claude.current = ""`（Claude 未启用）的情况下，`getActiveClaudeProvider()` 会调用 `normalizeCurrentClaudeProviderId()`。原逻辑把空字符串当成不存在的 provider，又将其“修正”为同一个空字符串，并标记配置已变更，触发 `writeConfig()`。

因此，一个本应只读的 provider 查询会反复创建临时配置文件并替换 `config.json`。当 Windows 拒绝这次文件替换时，异常沿调用链上抛，界面报：

```text
[ProviderHandler] Failed to get active provider
java.nio.file.AccessDeniedException: config.json-*.tmp -> config.json
```

确定的缺陷是“不必要的配置写入”；当时具体由哪个进程或安全组件拒绝文件替换，尚未确定。

## 修复内容

- 将空字符串识别为合法的“未启用 Claude”状态，不再因该状态重复执行无效修正及配置写入。
- 保留缺失字段初始化、失效 provider 修正、内置 provider 和有效自定义 provider 的原有处理逻辑。
- 保持 Claude 未启用，不自动切换或启用其他 provider；不修改 Codex、凭据或其他用户设置。

核心调整位于 `ProviderManager.normalizeCurrentClaudeProviderId()` 的 provider 存在性判断，等价于将 `providers.has(current)` 替换为：

```java
current != null && (current.isEmpty() || providers.has(current))
```

配置结构不完整时，原有必要的初始化写入仍会执行；此次修改不是对所有配置写入一律跳过。

## 验证结果

- 使用 JDK 17 对原版及补丁版的真实 `ProviderManager` 类运行 16 项离线用例；使用合成配置和测试日志桩，不写真实用户配置。
- 连续查询未启用的 Claude provider 1,000 次：原版写配置 **1,000 次**，补丁版写配置 **0 次**。
- 模拟配置写入被拒绝：原版复现 `AccessDeniedException` 的传播路径；补丁版不触发该无效写入，也不再抛出该异常。
- 覆盖显式未启用但已保存 provider、内置选择、自定义选择、失效选择、缺失或 null 字段，以及其他配置保持不变等场景。
- 安装后重新运行 16 项用例，全部通过；已安装 JAR 的校验值与补丁一致，运行中的 IDEA JVM 已加载补丁辅助类。
- 现场验证：2026-10-24 02:20:18–02:25:28 UTC，本次重启并加载插件后，原 provider 获取失败、配置替换访问拒绝及配置写入失败均为 **0 次**。

## 实现与范围说明

当前交付为 CC GUI `0.6.2` 的本地热补丁：仅修改 `ProviderManager.class` 中上述判断对应的一个调用点，新增 `ProviderNormalizationHotfix` 辅助类及补丁标记。另附构建、回归验证、安装和回滚工具；安装包含版本及 SHA-256 校验、原版备份和 IDEA 运行状态检查。

没有修改 `writeConfig()` 的原子替换机制，也没有增加配置写入重试、放宽权限、吞掉异常或关闭安全软件。因此，这不是通用的 Windows 文件访问拒绝修复；真正需要落盘的操作若仍被拒绝，需要另外排查。

若将修复合入插件源代码，应在对应的 provider 归一化判断中落地上述等价逻辑；本地 JAR 修补工具不是源代码修复本身，也不应直接套用于其他插件版本。
