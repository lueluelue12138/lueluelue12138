# rules_boost 版本升级与风险阻断迁移指南

> **文档版本**: v1.0  
> **适用项目**: 自动驾驶大型 C++ 单体仓库  
> **作者**: 构建架构团队  
> **创建日期**: 2025-12-15

---

## 目录

1. [执行摘要](#执行摘要)
2. [版本差异深度分析](#版本差异深度分析)
   - [Boost 版本对比](#boost-版本对比)
   - [rules_boost 构建规则变化](#rules_boost-构建规则变化)
   - [核心模块 Breaking Changes](#核心模块-breaking-changes)
3. [分步迁移方案](#分步迁移方案)
   - [WORKSPACE 迁移方案](#workspace-迁移方案)
   - [MODULE.bazel 迁移方案](#modulebazel-迁移方案)
   - [锁定 Boost 版本策略](#锁定-boost-版本策略)
4. [风险评估与排查](#风险评估与排查)
   - [编译期风险](#编译期风险)
   - [链接期风险](#链接期风险)
   - [运行时风险](#运行时风险)
5. [验证与测试策略](#验证与测试策略)
   - [依赖分析与查询](#依赖分析与查询)
   - [灰度升级方案](#灰度升级方案)
   - [双版本并存测试](#双版本并存测试)
6. [附录](#附录)

---

## 执行摘要

| 维度 | 当前版本 | 目标版本 |
|------|----------|----------|
| **rules_boost commit** | `1e3a69bf2d5cd10c34b74f066054cd335d033d71` | `6246371b8096ffbb57cc21b6dfef56b381582b68` (Latest HEAD) |
| **Commit 日期** | 2020-06-01 | 2025-12-14 |
| **Boost 版本** | 1.71.0 | 1.84.0 |
| **构建系统** | WORKSPACE | MODULE.bazel (Bzlmod) |
| **时间跨度** | - | ~5.5 年 |

**关键影响评估**:
- 🔴 **高风险**: Boost.Asio 行为变化、Boost.Filesystem API 变更
- 🟡 **中风险**: C++ 标准兼容性、ABI 不兼容
- 🟢 **低风险**: 编译 flag 变化、头文件路径调整

---

## 版本差异深度分析

### Boost 版本对比

| 依赖项 | 旧版本 (1e3a69b) | 新版本 (Latest) | 变化说明 |
|--------|------------------|-----------------|----------|
| **Boost** | 1.71.0 | 1.84.0 | 主要升级，跨越13个次版本 (1.71 → 1.84) |
| **zlib** | 1.2.11 | 1.3.1.bcr.8 | BCR 托管版本 |
| **bzip2** | 1.0.6 | 1.0.8.bcr.3 | BCR 托管版本 |
| **xz (lzma)** | 5.2.3 | 5.4.5.bcr.6 | 重命名为 xz |
| **zstd** | 1.4.4 | 1.5.7 | 压缩算法更新 |
| **boringssl** | 758e4ab0 | 0.20251124.0 | 版本格式变化 |
| **bazel_skylib** | 0.9.0 | 1.8.2 | 基础库升级 |
| **rules_cc** | 内置 | 0.0.10 | 新增显式依赖 |
| **platforms** | 内置 | 1.0.0 | 新增显式依赖 |

### rules_boost 构建规则变化

#### 1. 构建系统迁移 (WORKSPACE → Bzlmod)

**旧版本 (WORKSPACE 方式)**:
```python
# WORKSPACE 文件
http_archive(
    name = "com_github_nelhage_rules_boost",
    sha256 = "...",
    strip_prefix = "rules_boost-1e3a69bf2d5cd10c34b74f066054cd335d033d71",
    urls = ["https://github.com/nelhage/rules_boost/archive/1e3a69b....tar.gz"],
)

load("@com_github_nelhage_rules_boost//:boost/boost.bzl", "boost_deps")
boost_deps()
```

**新版本 (MODULE.bazel 方式)**:
```python
# MODULE.bazel 文件
bazel_dep(name = "rules_boost", repo_name = "com_github_nelhage_rules_boost")
archive_override(
    module_name = "rules_boost",
    urls = ["https://github.com/nelhage/rules_boost/archive/refs/heads/master.tar.gz"],
    strip_prefix = "rules_boost-master",
)

non_module_boost_repositories = use_extension(
    "@com_github_nelhage_rules_boost//:boost/repositories.bzl", 
    "non_module_dependencies"
)
use_repo(non_module_boost_repositories, "boost")
```

#### 2. 编译选项变化

| 选项 | 旧版本 | 新版本 | 影响 |
|------|--------|--------|------|
| **警告抑制** | `-Wno-unused-value` (仅 Linux) | `-w` (全平台) | 更全面的警告抑制 |
| **Windows 选项** | 无 | `/W0` | 新增 Windows 支持 |
| **cc_library 加载** | `native.cc_library` | `@rules_cc//cc:cc_library.bzl` | 显式规则加载 |

#### 3. 头文件路径变化

**旧版本目录结构**:
```
boost/
├── algorithm.hpp
├── asio.hpp
└── ...
libs/
├── algorithm/
│   └── src/
└── asio/
    └── src/
```

**新版本目录结构** (libs 内 include 子目录):
```
libs/
├── algorithm/
│   └── include/
│       └── boost/
│           └── algorithm.hpp
└── asio/
    └── include/
        └── boost/
            └── asio.hpp
```

> ⚠️ **重要**: `includes` 路径从 `include_pattern % library_name` 变更为 `libs/%s/include`

### 核心模块 Breaking Changes

#### Boost.Asio (1.71.0 → 1.84.0)

| 版本 | 关键变更 |
|------|----------|
| **1.72.0** | `io_context::executor_type` 更改为符合 Networking TS |
| **1.74.0** | 弃用 `buffer()` 返回 `mutable_buffer` |
| **1.75.0** | 引入 `any_completion_executor` |
| **1.77.0** | `io_context::run()` 默认行为变化 |
| **1.78.0** | 新增 `file_base` 和相关文件 I/O 支持 |
| **1.80.0** | `cancel()` 对某些操作的行为变化 |
| **1.82.0** | 移除已弃用的 `asio::strand` 构造函数 |
| **1.83.0** | `ip::tcp::socket` 默认构造函数变更 |

**自动驾驶场景高危变更**:

```cpp
// ❌ 旧代码 (1.71.0)
boost::asio::io_context io;
boost::asio::strand<boost::asio::io_context::executor_type> strand(io);

// ✅ 新代码 (1.84.0)
boost::asio::io_context io;
boost::asio::strand<boost::asio::io_context::executor_type> strand(io.get_executor());
```

```cpp
// ❌ 旧代码 - 可能导致死锁的变化
socket.cancel();  // 1.80+ 对已关闭套接字的行为变化

// ✅ 推荐做法
if (socket.is_open()) {
    boost::system::error_code ec;
    socket.cancel(ec);
}
```

#### Boost.Filesystem (1.71.0 → 1.84.0)

| 版本 | 关键变更 |
|------|----------|
| **1.72.0** | `path::lexically_normal()` 行为修正 |
| **1.75.0** | 弃用 `codecvt()` 相关接口 |
| **1.76.0** | `directory_iterator` 异常行为变化 |
| **1.79.0** | 引入 `directory_entry::refresh()` |
| **1.81.0** | `path` 构造函数对空字符串处理变化 |

```cpp
// ⚠️ 潜在问题
boost::filesystem::path p("");  // 1.81+ 行为可能不同
```

#### Boost.Serialization (1.71.0 → 1.84.0)

| 版本 | 关键变更 |
|------|----------|
| **1.74.0** | 类版本控制 (class versioning) 变化 |
| **1.77.0** | `BOOST_SERIALIZATION_NVP` 宏行为变化 |
| **1.79.0** | 二进制归档格式微调 |

> 🚨 **严重警告**: 序列化格式可能不兼容！已持久化的数据可能无法正确反序列化。

#### Boost.Thread (1.71.0 → 1.84.0)

| 版本 | 关键变更 |
|------|----------|
| **1.73.0** | `shared_mutex` 实现变化 |
| **1.78.0** | `thread::hardware_concurrency()` 返回值类型精确化 |

#### Boost.Beast (1.71.0 → 1.84.0)

| 版本 | 关键变更 |
|------|----------|
| **1.75.0** | WebSocket `async_read` 重载变化 |
| **1.79.0** | HTTP parser 默认限制变化 |
| **1.83.0** | `beast::string_view` 默认类型变化 |

---

## 分步迁移方案

### WORKSPACE 迁移方案

如果您仍需使用 WORKSPACE (例如 Bazel 5.x/6.x 兼容性)，使用以下配置:

```python
# WORKSPACE
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

# Step 1: 引入 rules_cc (新版本必需)
http_archive(
    name = "rules_cc",
    sha256 = "d75a040c32954da0d308d3f2ea2ba735490f49b3a7aa3e4b40259ca4b814f825",
    urls = ["https://github.com/bazelbuild/rules_cc/releases/download/0.0.10/rules_cc-0.0.10.tar.gz"],
    strip_prefix = "rules_cc-0.0.10",
)

# Step 2: 引入 platforms
http_archive(
    name = "platforms",
    sha256 = "5308fc1d8865406a49427ba24a9ab53087f17f5266a7aabbfc28823f3916e1ca",
    urls = ["https://github.com/bazelbuild/platforms/releases/download/0.0.8/platforms-0.0.8.tar.gz"],
)

# Step 3: 引入 bazel_skylib (新版本)
http_archive(
    name = "bazel_skylib",
    sha256 = "cd55a062e763b9349921f0f5db8c3933288dc8ba4f76dd9416aac68acee3cb94",
    urls = ["https://github.com/bazelbuild/bazel-skylib/releases/download/1.5.0/bazel-skylib-1.5.0.tar.gz"],
)

# Step 4: 引入 rules_boost (最新版本)
http_archive(
    name = "com_github_nelhage_rules_boost",
    sha256 = "REPLACE_WITH_ACTUAL_SHA256",  # 计算最新 commit 的 sha256
    strip_prefix = "rules_boost-master",
    urls = ["https://github.com/nelhage/rules_boost/archive/refs/heads/master.tar.gz"],
)

load("@com_github_nelhage_rules_boost//:boost/boost.bzl", "boost_deps")
boost_deps()
```

### MODULE.bazel 迁移方案

推荐使用 Bzlmod (Bazel 6.0+)，这是 rules_boost 现在支持的主要方式:

```python
# MODULE.bazel

module(
    name = "your_project_name",
    version = "1.0.0",
)

# Step 1: 添加 rules_boost 依赖
bazel_dep(name = "rules_boost", repo_name = "com_github_nelhage_rules_boost")

# Step 2: 使用 archive_override 锁定到特定 commit
archive_override(
    module_name = "rules_boost",
    urls = ["https://github.com/nelhage/rules_boost/archive/6246371b8096ffbb57cc21b6dfef56b381582b68.tar.gz"],
    strip_prefix = "rules_boost-6246371b8096ffbb57cc21b6dfef56b381582b68",
    integrity = "sha256-REPLACE_WITH_ACTUAL_HASH",  # 计算实际 hash
)

# Step 3: 注册 Boost 仓库扩展
non_module_boost_repositories = use_extension(
    "@com_github_nelhage_rules_boost//:boost/repositories.bzl", 
    "non_module_dependencies"
)
use_repo(
    non_module_boost_repositories,
    "boost",
)
```

### 锁定 Boost 版本策略

**目标**: 升级 rules_boost 的同时，暂时锁定 Boost 版本以减少代码修改量。

#### 方法一: 覆盖 boost_deps 函数

创建自定义的 `boost_deps` 覆盖:

```python
# third_party/boost/boost_override.bzl
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")
load("@bazel_tools//tools/build_defs/repo:utils.bzl", "maybe")

def boost_deps_pinned():
    """使用固定版本的 Boost 1.71.0 而非 rules_boost 默认的 1.84.0"""
    maybe(
        http_archive,
        name = "boost",
        build_file = "@com_github_nelhage_rules_boost//:boost.BUILD",
        patch_cmds = ["rm -f doc/pdf/BUILD"],
        patch_cmds_win = ["Remove-Item -Force doc/pdf/BUILD"],
        # 锁定到 Boost 1.71.0
        url = "https://archives.boost.io/release/1.71.0/source/boost_1_71_0.tar.bz2",
        sha256 = "d73a8da01e8bf8c7eda40b4c84915071a8c8a0df4a6734537ddde4a8580524ee",
        strip_prefix = "boost_1_71_0",
    )
```

```python
# WORKSPACE (使用自定义版本)
load("//third_party/boost:boost_override.bzl", "boost_deps_pinned")
boost_deps_pinned()
```

#### 方法二: MODULE.bazel 中使用 single_version_override

```python
# MODULE.bazel

# 创建自定义仓库规则来覆盖 Boost 版本
boost_override = use_repo_rule("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

# 在 repositories.bzl 扩展之前声明 boost
boost_override(
    name = "boost",
    build_file = "@com_github_nelhage_rules_boost//:boost.BUILD",
    url = "https://archives.boost.io/release/1.71.0/source/boost_1_71_0.tar.bz2",
    sha256 = "d73a8da01e8bf8c7eda40b4c84915071a8c8a0df4a6734537ddde4a8580524ee",
    strip_prefix = "boost_1_71_0",
)
```

#### 方法三: 渐进式版本升级

```python
# 使用中间版本进行阶梯式升级
BOOST_VERSIONS = {
    "1.71.0": {
        "sha256": "d73a8da01e8bf8c7eda40b4c84915071a8c8a0df4a6734537ddde4a8580524ee",
        "strip_prefix": "boost_1_71_0",
    },
    "1.76.0": {
        "sha256": "f0397ba6e982c4450f27bf32a2a83292aba035b827a5623a14636ea583318c41",
        "strip_prefix": "boost_1_76_0",
    },
    "1.80.0": {
        "sha256": "1e19565d82e43bc59209a168f5ac899d3ba471d55c7610c677d4ccf2c9c500c0",
        "strip_prefix": "boost_1_80_0",
    },
    "1.84.0": {
        "sha256": "4d27e9efed0f6f152dc28db6430b9d3dfb40c0345da7342eaa5a987dde57bd95",
        "strip_prefix": "boost-1.84.0",
    },
}

# 当前阶段使用的版本
CURRENT_BOOST_VERSION = "1.76.0"  # 逐步升级
```

---

## 风险评估与排查

### 编译期风险

#### 1. C++ 标准兼容性

| 风险级别 | 问题描述 | 解决方案 |
|----------|----------|----------|
| 🔴 高 | Boost 1.80+ 部分组件需要 C++14 | 确保 `--cxxopt=-std=c++14` 或更高 |
| 🟡 中 | 模板实例化错误增加 | 审查编译日志，逐个修复 |
| 🟢 低 | 弃用警告增多 | 使用 `-Wno-deprecated-declarations` 临时抑制 |

**检查当前 C++ 标准**:
```bash
# 在 .bazelrc 中确认
grep -r "cxxopt" .bazelrc

# 或在 BUILD 文件中
bazel query 'attr(copts, ".*std=c\+\+.*", //...)'
```

**推荐 .bazelrc 配置**:
```bash
# .bazelrc
build --cxxopt=-std=c++17
build --host_cxxopt=-std=c++17
```

#### 2. 头文件路径变化

**常见编译错误**:
```
error: boost/detail/fwd.hpp: No such file or directory
error: 'boost::detail::xxx' has not been declared
```

**排查命令**:
```bash
# 查找所有直接包含 boost/detail 的文件
grep -rn "boost/detail/" --include="*.cpp" --include="*.hpp" .

# 查找使用旧路径的 includes
bazel query 'attr(includes, ".*boost.*", //...)'
```

#### 3. 弃用 API 使用

**自动检测脚本**:
```bash
#!/bin/bash
# check_deprecated_apis.sh

DEPRECATED_PATTERNS=(
    "boost::asio::strand<"              # 需要更新构造
    "boost::filesystem::codecvt"        # 已弃用
    "BOOST_THROW_EXCEPTION"             # 行为变化
    "boost::asio::buffer("              # 返回类型变化
)

for pattern in "${DEPRECATED_PATTERNS[@]}"; do
    echo "=== Checking: $pattern ==="
    grep -rn "$pattern" --include="*.cpp" --include="*.hpp" .
done
```

### 链接期风险

#### 1. 符号丢失

**症状**:
```
undefined reference to `boost::xxx::yyy()'
```

**原因分析**:
- Boost 库内部符号重命名
- 模板特化变化导致符号变化
- 链接顺序敏感性增加

**排查步骤**:
```bash
# 1. 检查符号是否存在
nm -C bazel-bin/external/boost/libboost_xxx.a | grep "symbol_name"

# 2. 检查链接顺序
bazel query 'deps(//your:target)' --output=graph | grep boost

# 3. 使用 --linkopt 增加调试信息
bazel build //your:target --linkopt=-Wl,--verbose
```

#### 2. ABI 不兼容

**高风险场景**:
- 跨编译单元传递 Boost 对象
- 共享库边界的 Boost 类型
- 预编译的第三方库依赖特定 Boost 版本

**验证方法**:
```cpp
// 在关键模块添加 ABI 版本检查
#include <boost/version.hpp>
static_assert(BOOST_VERSION == 108400, "Boost version mismatch!");
```

**检测脚本**:
```bash
# 检查所有预编译库的 Boost 依赖
for lib in $(find . -name "*.so" -o -name "*.a"); do
    echo "=== $lib ==="
    nm -C "$lib" 2>/dev/null | grep -c "boost::" || echo "No boost symbols"
done
```

### 运行时风险

#### 1. Boost.Asio 行为变化 (自动驾驶高危)

**风险场景**:

| 场景 | 风险描述 | 验证方法 |
|------|----------|----------|
| 网络通信 | `io_context::run()` 返回行为变化 | 压力测试 + 日志监控 |
| 定时器 | `steady_timer` 精度变化 | 时序测试套件 |
| 信号处理 | `signal_set` 行为变化 | 信号处理集成测试 |
| SSL/TLS | BoringSSL 版本更新 | HTTPS 连接测试 |

**关键验证代码**:
```cpp
// 验证 io_context 行为
void verify_io_context_behavior() {
    boost::asio::io_context io;
    boost::asio::steady_timer timer(io, std::chrono::milliseconds(100));
    
    bool timer_fired = false;
    timer.async_wait([&](const boost::system::error_code& ec) {
        timer_fired = true;
        assert(!ec);
    });
    
    auto handlers = io.run();
    assert(handlers == 1);
    assert(timer_fired);
}
```

#### 2. Boost.Serialization 数据兼容性

**验证检查清单**:
- [ ] 测试反序列化旧版本 (1.71.0) 创建的数据
- [ ] 验证所有序列化类型的版本号
- [ ] 检查自定义 `serialize()` 函数的行为

**兼容性测试框架**:
```cpp
// test_serialization_compat.cpp
#include <boost/archive/binary_iarchive.hpp>
#include <boost/archive/binary_oarchive.hpp>
#include <sstream>

template<typename T>
bool test_backward_compatibility(const std::string& legacy_data_path) {
    std::ifstream ifs(legacy_data_path, std::ios::binary);
    if (!ifs) return false;
    
    try {
        boost::archive::binary_iarchive ia(ifs);
        T obj;
        ia >> obj;
        return obj.validate();  // 自定义验证
    } catch (const boost::archive::archive_exception& e) {
        std::cerr << "Deserialization failed: " << e.what() << std::endl;
        return false;
    }
}
```

#### 3. 内存布局变化

**风险评估**:
```cpp
// 编译时检查关键结构体大小
static_assert(sizeof(boost::asio::ip::tcp::endpoint) == EXPECTED_SIZE, 
              "TCP endpoint size changed - ABI break!");
```

**运行时验证**:
```cpp
void verify_memory_layout() {
    // 检查关键对象的内存布局
    std::cout << "sizeof(ip::tcp::socket) = " << sizeof(boost::asio::ip::tcp::socket) << std::endl;
    std::cout << "sizeof(steady_timer) = " << sizeof(boost::asio::steady_timer) << std::endl;
    // 与基线对比
}
```

---

## 验证与测试策略

### 依赖分析与查询

#### 1. 查找所有依赖 Boost 的 Target

```bash
# 直接依赖 @boost 的 targets
bazel query 'rdeps(//..., @boost//...)' --output=label 2>/dev/null

# 或使用更精确的查询
bazel query 'kind("cc_.*", rdeps(//..., @boost//:all))' --output=label

# 按模块分类
bazel query 'rdeps(//..., @boost//:asio)' --output=label
bazel query 'rdeps(//..., @boost//:filesystem)' --output=label
bazel query 'rdeps(//..., @boost//:serialization)' --output=label
```

#### 2. 生成依赖图

```bash
# 生成 Graphviz 格式的依赖图
bazel query 'allpaths(//your/target, @boost//...)' --output=graph > boost_deps.dot
dot -Tpng boost_deps.dot -o boost_deps.png

# 统计依赖 Boost 的模块数量
bazel query 'rdeps(//..., @boost//...)' --output=label | wc -l
```

#### 3. 使用 Aspect 进行深度分析

```python
# boost_deps_aspect.bzl
def _boost_deps_aspect_impl(target, ctx):
    deps = []
    if hasattr(ctx.rule.attr, 'deps'):
        for dep in ctx.rule.attr.deps:
            if str(dep.label).startswith("@boost"):
                deps.append(str(dep.label))
    
    if deps:
        print("Target {} depends on Boost: {}".format(target.label, deps))
    
    return []

boost_deps_aspect = aspect(
    implementation = _boost_deps_aspect_impl,
    attr_aspects = ['deps'],
)
```

```bash
# 运行 aspect
bazel build //... --aspects=//tools:boost_deps_aspect.bzl%boost_deps_aspect
```

### 灰度升级方案

#### 阶段一: 准备阶段 (1-2 周)

```bash
# 1. 创建升级分支
git checkout -b feature/boost-upgrade-prep

# 2. 记录当前状态
bazel query 'rdeps(//..., @boost//...)' > baseline_boost_deps.txt
bazel build //... 2>&1 | tee baseline_build.log

# 3. 运行全量测试作为基线
bazel test //... 2>&1 | tee baseline_test.log
```

#### 阶段二: 升级 rules_boost (保持 Boost 版本)

```bash
# 1. 升级 rules_boost，但锁定 Boost 1.71.0
git checkout -b feature/upgrade-rules-boost-only

# 2. 修改 WORKSPACE/MODULE.bazel (参见上文锁定版本策略)

# 3. 构建验证
bazel build //...

# 4. 测试验证
bazel test //...
```

#### 阶段三: 阶梯式 Boost 版本升级

```bash
# 升级路径: 1.71.0 → 1.76.0 → 1.80.0 → 1.84.0

for version in "1.76.0" "1.80.0" "1.84.0"; do
    git checkout -b feature/boost-$version
    
    # 更新 Boost 版本
    # 编辑 boost_override.bzl
    
    # 构建测试
    bazel build //... 2>&1 | tee build_$version.log
    bazel test //... 2>&1 | tee test_$version.log
    
    # 对比与基线的差异
    diff baseline_build.log build_$version.log > build_diff_$version.txt
    diff baseline_test.log test_$version.log > test_diff_$version.txt
done
```

#### 阶段四: 全量验证

```bash
# 1. 执行完整的 CI 流水线
# 2. 运行性能基准测试
# 3. 执行集成测试
# 4. 进行长时间稳定性测试 (24-48小时)
```

### 双版本并存测试

#### 方法一: 使用 Bazel 配置条件

```python
# BUILD.bazel
config_setting(
    name = "use_new_boost",
    define_values = {"boost_version": "new"},
)

cc_library(
    name = "my_lib",
    deps = select({
        ":use_new_boost": ["@boost_new//:asio"],
        "//conditions:default": ["@boost_old//:asio"],
    }),
)
```

```bash
# 使用新版本构建
bazel build //... --define=boost_version=new

# 使用旧版本构建 (默认)
bazel build //...
```

#### 方法二: 双仓库配置

```python
# WORKSPACE
# 旧版本
http_archive(
    name = "com_github_nelhage_rules_boost_old",
    strip_prefix = "rules_boost-1e3a69bf2d5cd10c34b74f066054cd335d033d71",
    urls = ["https://github.com/nelhage/rules_boost/archive/1e3a69b....tar.gz"],
)

# 新版本  
http_archive(
    name = "com_github_nelhage_rules_boost_new",
    strip_prefix = "rules_boost-master",
    urls = ["https://github.com/nelhage/rules_boost/archive/refs/heads/master.tar.gz"],
)
```

#### 方法三: A/B 测试框架

```cpp
// boost_version_test.hpp
#pragma once

#ifdef USE_BOOST_1_84
    #include "boost_1_84/asio.hpp"
    namespace boost_version = boost_1_84;
#else
    #include "boost_1_71/asio.hpp"
    namespace boost_version = boost_1_71;
#endif

// 统一接口
class NetworkClient {
    void connect() {
        boost_version::asio::io_context io;
        // ...
    }
};
```

---

## 附录

### A. 常见编译错误对照表

| 错误信息 | 原因 | 解决方案 |
|----------|------|----------|
| `error: 'io_service' is not a member of 'boost::asio'` | `io_service` 重命名为 `io_context` | 搜索替换 `io_service` → `io_context` |
| `error: no matching function for call to 'strand<...>::strand(io_context&)'` | strand 构造函数变化 | 使用 `io.get_executor()` |
| `error: 'boost::filesystem::wpath' was not declared` | `wpath` 已移除 | 使用 `boost::filesystem::path` |
| `error: 'boost/detail/xxx.hpp' file not found` | 内部头文件重组 | 查找替代公共头文件 |

### B. Bazel 版本兼容性矩阵

| Bazel 版本 | WORKSPACE 支持 | Bzlmod 支持 | 推荐 |
|------------|----------------|-------------|------|
| 5.x | ✅ | ⚠️ 实验性 | WORKSPACE |
| 6.x | ✅ | ✅ | Bzlmod (可选) |
| 7.x | ⚠️ 弃用警告 | ✅ | Bzlmod |
| 8.x+ | ❌ | ✅ | Bzlmod (必须) |

### C. 升级前检查清单

- [ ] 确认 Bazel 版本 ≥ 6.0 (推荐 7.x)
- [ ] 备份当前 WORKSPACE/MODULE.bazel
- [ ] 记录当前构建和测试基线
- [ ] 识别所有依赖 Boost 的 target
- [ ] 审查代码中的 Boost API 使用
- [ ] 检查序列化数据的兼容性
- [ ] 准备回滚方案
- [ ] 安排足够的测试时间

### D. 有用的命令集合

```bash
# 1. 计算 tar.gz 的 sha256
curl -sL "URL" | sha256sum

# 2. 清除 Bazel 缓存强制重新下载
bazel clean --expunge

# 3. 查看 Boost 版本
bazel run @boost//:version_check  # 如果有的话

# 4. 导出所有外部依赖
bazel query '@*//...' --output=label

# 5. 验证 MODULE.bazel 语法
bazel mod graph

# 6. 生成 lockfile
bazel mod deps --lockfile_mode=update
```

### E. 联系与支持

- **rules_boost 官方仓库**: https://github.com/nelhage/rules_boost
- **Boost 官方文档**: https://www.boost.org/doc/
- **Bazel Central Registry**: https://registry.bazel.build/
- **Bazel 社区 Slack**: https://slack.bazel.build/

---

> **文档维护**: 本文档应随项目进展持续更新。升级完成后，请记录实际遇到的问题和解决方案，以便后续参考。
