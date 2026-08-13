# Mooncake glog 重复初始化修复设计

日期：2026-08-13

## 背景

SGLang 的 Elastic EP 1P1D 服务加载最新 Mooncake wheel 时，prefill 和
decode 进程均在启动阶段退出。崩溃栈显示 `mooncake::loadGlobalConfig()`
调用 `google::InitGoogleLogging()` 时，glog 已经被同一进程中的其他原生
组件初始化：

```text
Check failed: !IsGoogleLoggingInitialized()
You called InitGoogleLogging() twice!
google::InitGoogleLogging()
mooncake::loadGlobalConfig()
mooncake::globalConfig()
```

`MC_LOG_DIR` 存在时，`mooncake-transfer-engine/src/config.cpp` 当前无条件
调用 `InitGoogleLogging()`。`globalConfig()` 内的 `std::once_flag` 只能保护
单个 TransferEngine 副本，无法代表 glog 这一进程级全局资源的所有者。
当 Mooncake 被多个 Python 原生扩展或其他已使用 glog 的宿主加载时，重复
初始化会触发 glog 的致命检查。

## 目标

- Mooncake 可安全运行在 glog 已由宿主初始化的进程中。
- 保留 Mooncake 独立运行且 glog 尚未初始化时的现有初始化行为。
- 保留 `MC_LOG_DIR` 对日志目录及相关 glog flags 的现有配置语义。
- 不增加推理热路径开销，不改变 TransferEngine 架构、ABI 或打包布局。
- 用无 GPU、无 RDMA 依赖的自动化测试覆盖本次崩溃。

## 非目标

- 不统一 SGLang、Mooncake PG、Mooncake EP 和 TransferEngine 的日志框架。
- 不将静态 `transfer_engine` 改造成共享库。
- 不调整 Python 扩展加载顺序。
- 不修改 Elastic EP 的控制面、数据面或容错状态机。

## 方案比较

### 方案 A：初始化前查询 glog 状态（采用）

在处理 `MC_LOG_DIR` 时，仅当
`google::IsGoogleLoggingInitialized()` 返回 false 才调用
`google::InitGoogleLogging()`。无论 glog 是否已初始化，后续目录合法性检查和
flags 设置保持不变。

优点：修改范围最小，兼容宿主初始化和 Mooncake 独立初始化两种场景，不影响
推理热路径。该逻辑只在全局配置首次加载时运行一次。

边界：状态查询和初始化不是跨动态库副本的通用所有权协议。当前故障发生在
Python 扩展串行加载阶段，此保护可消除已初始化后的确定性重复调用；若未来需要
支持多个原生模块并发争抢首次初始化，应由进程宿主集中管理日志初始化。

### 方案 B：Mooncake 永不初始化 glog（不采用）

只设置 flags，把 glog 初始化完全交给宿主。这会改变 Mooncake 独立程序的现有
行为，并要求所有调用方承担新的初始化责任。

### 方案 C：共享单一 TransferEngine 动态库（不采用）

让所有 Python 扩展链接同一份共享实现，可收敛部分进程内静态状态，但会涉及
CMake、wheel 布局、运行时搜索路径和 ABI，风险与本次故障不成比例。

## 实现设计

修改 `mooncake-transfer-engine/src/config.cpp` 中 `MC_LOG_DIR` 分支：

```cpp
if (!google::IsGoogleLoggingInitialized()) {
    google::InitGoogleLogging("mooncake-transfer-engine");
}
```

该检查位于现有初始化调用的原位置。目录校验、warning、`FLAGS_log_dir`、
`FLAGS_logtostderr` 和 `FLAGS_stop_logging_if_full_disk` 均不改动。

## 测试设计

新增独立的无硬件回归测试可执行文件，避免其他测试用例或 glog 全局状态影响
测试前置条件。测试执行以下步骤：

1. 宿主先调用 `google::InitGoogleLogging()`。
2. 设置 `MC_LOG_DIR` 为当前可写目录。
3. 调用 `mooncake::loadGlobalConfig()`。
4. 验证调用正常返回且日志目录配置仍被应用。
5. 清理环境变量并关闭由测试宿主创建的 glog 状态。

修复前第 3 步因 glog 重复初始化而 abort；修复后测试正常结束。测试将注册到
CTest，并与现有 `config_test` 一起运行。

回归验证分层进行：

- 本地：格式/静态检查、目标编译、独立回归测试、现有 `config_test`。
- bt 容器：拉取提交后重新构建 Mooncake，执行同一组测试。
- 集成探针：在镜像 Python 环境中先初始化一个 glog 使用者，再加载 Mooncake
  TransferEngine，验证扩展加载和全局配置不崩溃。
- 1P1D 服务：在获得独立的集群变更授权后，用原配置重新启动并检查 prefill、
  decode、router 健康状态与日志。

## 性能与生产风险

新增分支只执行于 `globalConfig()` 首次加载，不进入请求、传输或推理热路径，
对稳态性能没有可测影响。修复不新增锁、线程、RPC、内存分配或共享状态。

若 glog 已初始化，Mooncake 不再尝试取得初始化所有权，但仍按现有逻辑更新全局
flags。这与现有 `MC_LOG_DIR` 的配置目的相符，并避免重复初始化导致整个进程退出。

## 回滚

代码与测试均为独立提交，可通过 revert 修复提交恢复。设计不涉及数据迁移、配置
格式或持久化状态，因此无需运行时回滚步骤。
