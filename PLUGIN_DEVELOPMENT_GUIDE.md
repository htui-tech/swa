# SWA 插件开发指南

## 📖 概述

SWA 插件系统允许开发者通过动态库的方式扩展应用功能。插件可以处理特定的路由、提供全局数据、处理API请求等。

## 🏗️ 插件架构

### 插件接口

所有插件都必须实现 `SwaPlugin` trait：

```rust
pub trait SwaPlugin {
    /// 插件名称
    fn name(&self) -> &'static str;
    
    /// 插件初始化
    async fn init(&mut self, config: Option<PluginConfig>) -> tube::Result<()>;
    
    /// 为指定路径提供模板数据
    async fn data_for(&self, path: &str, param: &RequestParameter) -> Option<tube::Result<tube::Value>>;
    
    /// 获取全局数据
    async fn global_data(&self, param: &RequestParameter) -> Option<tube::Result<tube::Value>>;
    
    /// 处理API请求分发
    async fn api_dispatch(&self, app: &str, param: &RequestParameter) -> Option<tube::Result<tube::Value>>;
}
```

## 🚀 快速开始

### 1. 创建插件项目

```bash
cd swa/plugins
cargo new my-plugin --lib
```

### 2. 配置 Cargo.toml

```toml
[package]
name = "swa-plugin-my-plugin"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
swa-plugin-sdk = { path = "../sdk" }
tube = { workspace = true }
tokio = { workspace = true }
serde = { workspace = true }
serde_json = { workspace = true }

# 业务依赖
msa_cms = { workspace = true }
msa_sdk = { workspace = true }
```

### 3. 实现插件

```rust
// src/lib.rs
use swa_plugin_sdk::{SwaPlugin, RequestParameter, PluginConfig, value};
use tube::Value;

pub struct MyPlugin {
    config: Option<PluginConfig>,
}

impl MyPlugin {
    pub fn new() -> Self {
        Self { config: None }
    }
}

impl SwaPlugin for MyPlugin {
    fn name(&self) -> &'static str {
        "my-plugin"
    }
    
    async fn init(&mut self, config: Option<PluginConfig>) -> tube::Result<()> {
        self.config = config;
        log!("[my-plugin] 插件初始化完成");
        Ok(())
    }
    
    async fn data_for(&self, path: &str, param: &RequestParameter) -> Option<tube::Result<tube::Value>> {
        log!("[my-plugin] 处理路径: {}", path);
        
        match path {
            "my-page" => {
                Some(Ok(value!({
                    "title": "我的页面",
                    "content": "这是插件提供的内容",
                    "timestamp": chrono::Utc::now().timestamp()
                })))
            }
            _ => None
        }
    }
    
    async fn global_data(&self, param: &RequestParameter) -> Option<tube::Result<tube::Value>> {
        Some(Ok(value!({
            "pluginInfo": {
                "name": "my-plugin",
                "version": "1.0.0",
                "author": "开发者"
            }
        })))
    }
    
    async fn api_dispatch(&self, app: &str, param: &RequestParameter) -> Option<tube::Result<tube::Value>> {
        match app {
            "my-api" => {
                Some(Ok(value!({
                    "status": "success",
                    "data": {
                        "message": "来自插件的API响应"
                    }
                })))
            }
            _ => None
        }
    }
}

// 插件创建函数
#[no_mangle]
pub extern "C" fn swa_plugin_create() -> *mut dyn SwaPlugin {
    Box::into_raw(Box::new(MyPlugin::new()))
}
```

### 4. 编译插件

```bash
# Debug模式
./build-plugin-simple.sh macos my-plugin debug

# Release模式
./build-plugin-simple.sh macos my-plugin release
```

## 📋 插件开发最佳实践

### 1. 错误处理

```rust
async fn data_for(&self, path: &str, param: &RequestParameter) -> Option<tube::Result<tube::Value>> {
    match path {
        "my-page" => {
            // 使用 match 而不是 ? 操作符，提供更好的错误处理
            let mut data = value!({});
            
            match some_service_call(param) {
                Ok(result) => {
                    data["serviceData"] = result;
                }
                Err(e) => {
                    err_log!("[my-plugin] 服务调用失败: {}", e);
                    data["serviceData"] = Value::Null;
                }
            }
            
            Some(Ok(data))
        }
        _ => None
    }
}
```

### 2. 配置管理

```rust
async fn init(&mut self, config: Option<PluginConfig>) -> tube::Result<()> {
    if let Some(config) = config {
        // 读取插件特定配置
        if let Some(plugin_config) = config.plugin_data.get("myPlugin") {
            let api_key = plugin_config.get_string("apiKey");
            let debug_mode = plugin_config.get_bool("debug", false);
            
            log!("[my-plugin] API Key: {}", api_key);
            log!("[my-plugin] Debug Mode: {}", debug_mode);
        }
        
        // 读取应用全局配置
        if let Some(db_config) = config.app_config.get("database") {
            let host = db_config.get_string("host");
            log!("[my-plugin] 数据库主机: {}", host);
        }
    }
    
    Ok(())
}
```

### 3. 缓存使用

```rust
use std::sync::{Arc, RwLock};
use std::time::{Duration, Instant};

// 插件级别的缓存
static PLUGIN_CACHE: RwLock<Option<CacheItem>> = RwLock::new(None);

#[derive(Clone)]
struct CacheItem {
    data: Value,
    expires_at: Instant,
}

async fn data_for(&self, path: &str, param: &RequestParameter) -> Option<tube::Result<tube::Value>> {
    // 检查缓存
    if let Ok(cache_guard) = PLUGIN_CACHE.read() {
        if let Some(cache_item) = cache_guard.as_ref() {
            if cache_item.expires_at > Instant::now() {
                log!("[my-plugin] 从缓存返回数据");
                return Some(Ok(cache_item.data.clone()));
            }
        }
    }
    
    // 获取新数据
    let data = self.fetch_data(path, param).await?;
    
    // 写入缓存
    let cache_item = CacheItem {
        data: data.clone(),
        expires_at: Instant::now() + Duration::from_secs(300), // 5分钟
    };
    
    if let Ok(mut cache_guard) = PLUGIN_CACHE.write() {
        *cache_guard = Some(cache_item);
    }
    
    Some(Ok(data))
}
```

### 4. 路径处理

```rust
async fn data_for(&self, path: &str, param: &RequestParameter) -> Option<tube::Result<tube::Value>> {
    match path {
        // 处理处理器名称
        "content" => {
            let content_id = param.url_path
                .split('/')
                .last()
                .and_then(|s| s.parse::<i64>().ok())
                .unwrap_or(0);
            
            self.handle_content(content_id, param).await
        }
        // 处理完整路径
        path if path.starts_with("/api/") => {
            let api_path = path.strip_prefix("/api/").unwrap_or("");
            self.handle_api(api_path, param).await
        }
        _ => None
    }
}
```

## 🔧 插件配置

### 插件配置文件 (plugins/config.json)

```json
{
  "plugins": [
    {
      "name": "my-plugin",
      "enabled": true,
      "config": {
        "apiKey": "your-api-key",
        "debug": true,
        "cacheTimeout": 300
      }
    }
  ]
}
```

### 环境变量配置

```bash
# 插件特定环境变量
export MY_PLUGIN_API_KEY="your-api-key"
export MY_PLUGIN_DEBUG="true"
```

## 🧪 测试插件

### 单元测试

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[tokio::test]
    async fn test_data_for() {
        let plugin = MyPlugin::new();
        let param = RequestParameter::default();
        
        let result = plugin.data_for("my-page", &param).await;
        assert!(result.is_some());
        
        let data = result.unwrap().unwrap();
        assert_eq!(data.get_string("title"), "我的页面");
    }
}
```

### 集成测试

```rust
#[tokio::test]
async fn test_plugin_integration() {
    // 加载插件
    let plugin = unsafe { 
        let ptr = swa_plugin_create();
        Box::from_raw(ptr)
    };
    
    // 测试插件功能
    let param = RequestParameter::default();
    let result = plugin.data_for("my-page", &param).await;
    
    assert!(result.is_some());
}
```

## 📦 插件打包和分发

### 1. 构建插件

```bash
# 构建多个平台
./build-plugin-simple.sh linux my-plugin release
./build-plugin-simple.sh macos my-plugin release
./build-plugin-simple.sh macos-arm my-plugin release
```

### 2. 插件包结构

```
my-plugin/
├── libswa_plugin_my_plugin.so      # Linux版本
├── libswa_plugin_my_plugin.dylib   # macOS版本
├── config.json                     # 插件配置
├── README.md                       # 插件说明
└── docs/                          # 插件文档
```

### 3. 插件安装

```bash
# 复制插件文件
cp libswa_plugin_my_plugin.dylib plugins/

# 更新插件配置
echo '{"name": "my-plugin", "enabled": true}' >> plugins/config.json
```

## 🐛 调试插件

### 1. 日志调试

```rust
// 使用统一的日志宏
log!("[my-plugin] 普通信息日志");
err_log!("[my-plugin] 错误日志: {}", error);
```

### 2. 调试模式构建

```bash
# 构建调试版本
./build-plugin-simple.sh macos my-plugin debug

# 启用详细日志
RUST_LOG=debug cargo run --bin swa
```

### 3. 常见问题

**问题**: 插件加载失败
```bash
# 检查插件文件权限
ls -la plugins/libswa_plugin_*.dylib

# 检查依赖
ldd plugins/libswa_plugin_*.so
```

**问题**: 插件数据不显示
```rust
// 检查路径匹配
log!("[my-plugin] 请求路径: {}", path);
log!("[my-plugin] 参数: {:?}", param);
```

## 📚 参考资源

- [插件SDK文档](swa/plugins/sdk/src/lib.rs)
- [示例插件](swa/plugins/bas-site/)
- [API参考](swa/plugins/docs/API_REFERENCE.md)
- [最佳实践](swa/plugins/docs/BEST_PRACTICES.md)

## 🤝 贡献插件

1. Fork 项目
2. 创建插件分支
3. 实现插件功能
4. 添加测试和文档
5. 提交 Pull Request

---

**Happy Plugin Development!** 🚀
