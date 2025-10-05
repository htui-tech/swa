# SWA 插件编译指南

本文档介绍如何使用编译脚本构建 SWA 插件到不同平台。

## 快速开始

### 使用 Makefile (推荐)

```bash
# 编译当前平台版本
make build-bas-site

# 编译 macOS 版本
make build-macos

# 编译 Linux 版本
make build-linux

# 编译所有平台版本
make all

# 编译发布版本
make release-macos
make release-linux

# 清理编译缓存
make clean

# 查看帮助
make help
```

### 使用编译脚本

```bash
# 基本用法
./build-plugin.sh [platform] [plugin_name] [options]

# 编译 bas-site 插件到 macOS
./build-plugin.sh macos bas-site

# 编译 bas-site 插件到 Linux (发布版本)
./build-plugin.sh linux bas-site --release

# 编译前清理缓存
./build-plugin.sh macos bas-site --clean

# 查看帮助
./build-plugin.sh --help
```

## 支持的平台

| 平台名称 | 目标架构 | 描述 |
|---------|---------|------|
| `macos` | `x86_64-apple-darwin` | macOS x86_64 |
| `macos-arm` | `aarch64-apple-darwin` | macOS ARM64 (Apple Silicon) |
| `linux` | `x86_64-unknown-linux-gnu` | Linux x86_64 |
| `linux-musl` | `x86_64-unknown-linux-musl` | Linux musl (静态链接) |

## 支持的插件

| 插件名称 | 描述 | 路径 |
|---------|------|------|
| `bas-site` | 基础网站插件 | `swa/plugins/bas-site` |
| `sdk` | 插件SDK | `swa/plugins/sdk` |

## 编译选项

### 编译模式

- `debug` (默认): 调试版本，包含调试信息，编译速度快
- `release`: 发布版本，优化性能，编译速度慢

### 其他选项

- `--clean`: 编译前清理缓存
- `--release`: 编译发布版本
- `--help`: 显示帮助信息

## 输出目录结构

编译完成后，输出文件会保存在 `dist/` 目录下：

```
dist/
├── bas-site/
│   ├── x86_64-apple-darwin/
│   │   ├── libswa_plugin_bas_site.dylib
│   │   ├── Cargo.toml
│   │   └── version.txt
│   ├── aarch64-apple-darwin/
│   │   ├── libswa_plugin_bas_site.dylib
│   │   ├── Cargo.toml
│   │   └── version.txt
│   └── x86_64-unknown-linux-gnu/
│       ├── libswa_plugin_bas_site.so
│       ├── Cargo.toml
│       └── version.txt
└── sdk/
    └── ...
```

## 环境要求

### 必需工具

1. **Rust 工具链**
   ```bash
   # 安装 Rust
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
   source ~/.cargo/env
   
   # 验证安装
   rustc --version
   cargo --version
   ```

2. **目标平台工具链**
   ```bash
   # 脚本会自动安装，也可以手动安装
   rustup target add x86_64-apple-darwin
   rustup target add aarch64-apple-darwin
   rustup target add x86_64-unknown-linux-gnu
   rustup target add x86_64-unknown-linux-musl
   ```

### 可选工具

- **musl 工具链** (用于 Linux musl 编译)
  ```bash
  # macOS
  brew install musl-cross
  
  # Ubuntu/Debian
  sudo apt-get install musl-tools
  ```

## 常见问题

### 1. 编译失败

**问题**: 编译时出现错误
**解决**: 
- 检查 Rust 工具链是否正确安装
- 确保目标平台工具链已安装
- 尝试清理缓存后重新编译: `./build-plugin.sh macos bas-site --clean`

### 2. 找不到目标平台

**问题**: `error: target 'xxx' not found`
**解决**: 
```bash
# 安装目标平台工具链
rustup target add x86_64-apple-darwin
rustup target add aarch64-apple-darwin
rustup target add x86_64-unknown-linux-gnu
rustup target add x86_64-unknown-linux-musl
```

### 3. 权限问题

**问题**: `Permission denied`
**解决**: 
```bash
# 给脚本添加执行权限
chmod +x build-plugin.sh
```

### 4. 依赖问题

**问题**: 编译时依赖解析失败
**解决**: 
- 确保在项目根目录运行脚本
- 检查 `Cargo.toml` 中的依赖配置
- 尝试更新依赖: `cargo update`

## 高级用法

### 自定义编译配置

编辑 `build-config.json` 文件来自定义编译选项：

```json
{
  "build": {
    "default_mode": "release",
    "output_dir": "custom-dist",
    "clean_before_build": true
  }
}
```

### 批量编译

```bash
# 编译所有平台的所有插件
for platform in macos macos-arm linux linux-musl; do
  for plugin in bas-site sdk; do
    ./build-plugin.sh $platform $plugin --release
  done
done
```

### CI/CD 集成

在 GitHub Actions 或其他 CI 系统中使用：

```yaml
- name: Build Plugin
  run: |
    chmod +x build-plugin.sh
    ./build-plugin.sh linux bas-site --release
```

## 开发建议

1. **本地开发**: 使用 `make build-bas-site` 快速编译当前平台版本
2. **测试**: 使用 debug 版本进行测试，release 版本用于生产
3. **发布**: 使用 `make release-all` 编译所有平台的发布版本
4. **清理**: 定期使用 `make clean` 清理编译缓存

## 更新日志

- v1.0.0: 初始版本，支持 macOS 和 Linux 平台编译
- 支持 bas-site 和 sdk 插件
- 提供 Makefile 和 shell 脚本两种使用方式
