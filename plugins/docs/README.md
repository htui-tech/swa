# SWA 插件开发指南

## 📖 概述

SWA (Simple Web Application) 是一个基于 Rust 的插件化 Web 服务器，支持动态加载插件来扩展功能。插件系统提供了两种开发方式：

- **原生插件**：使用 Rust 和 SWA Plugin SDK 开发，性能最佳
- **JSON 适配器插件**：使用 C ABI 接口，支持跨语言开发

## 🏗️ 架构设计

```
SWA 主机应用
├── 插件注册表 (PluginRegistry)
├── 动态库加载器 (libloading)
├── 模板引擎 (Handlebars)
└── API 分发器

插件系统
├── SDK (swa-plugin-sdk)     # 统一的开发接口
├── 原生插件                 # 使用 SDK 的 Rust 插件
└── JSON 适配器插件          # 跨语言兼容插件
```

## 🚀 快速开始

### 1. 创建插件项目

```bash
# 在 plugins/ 目录下创建新插件
mkdir plugins/my-plugin
cd plugins/my-plugin
```

### 2. 配置 Cargo.toml

```toml
[package]
name = "swa-plugin-my-plugin"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]  # 编译为动态库

[dependencies]
swa-plugin-sdk = { path = "../sdk" }
# 其他依赖...
```

### 3. 实现插件

```rust
// src/lib.rs
use swa_plugin_sdk::{SwaPlugin, value, Value, Result, RequestParameter};

pub struct MyPlugin;

impl SwaPlugin for MyPlugin {
    fn name(&self) -> &'static str {
        "my-plugin"
    }
    
    fn data_for(&self, path: &str) -> Option<Value> {
        match path {
            "index" => Some(value!({
                "title": "我的插件",
                "content": "这是插件提供的数据"
            })),
            _ => None
        }
    }
    
    fn api_dispatch(&self, app: &str, _param: &RequestParameter) -> Option<Result<Value>> {
        match app {
            "myapp" => Some(Ok(value!({
                "ok": true,
                "data": {"message": "Hello from plugin!"}
            }))),
            _ => None
        }
    }
}

impl MyPlugin {
    pub fn new() -> Self {
        Self
    }
}

#[no_mangle]
pub extern "C" fn swa_plugin_create() -> *mut dyn SwaPlugin {
    Box::into_raw(Box::new(MyPlugin::new()))
}
```

### 4. 编译和部署

```bash
# 编译插件
cargo build --release

# 复制到插件目录
cp target/release/libswa_plugin_my_plugin.dylib ../../plugins/
```

## 📚 SDK 参考

### 核心 Trait

#### SwaPlugin

所有插件都必须实现的核心接口：

```rust
pub trait SwaPlugin: Send + Sync {
    /// 返回插件名称
    fn name(&self) -> &'static str;
    
    /// 为模板提供数据
    fn data_for(&self, path: &str) -> Option<Value> { None }
    
    /// 处理 API 请求
    fn api_dispatch(&self, app: &str, param: &RequestParameter) -> Option<Result<Value>> { None }
}
```

#### PluginMetadata (可选)

提供插件元数据信息：

```rust
pub trait PluginMetadata {
    fn info(&self) -> PluginInfo;
    fn capabilities(&self) -> Vec<&'static str>;
    fn supports(&self, capability: &str) -> bool;
}
```

#### PluginLifecycle (可选)

处理插件生命周期事件：

```rust
pub trait PluginLifecycle {
    fn on_init(&self) -> Result<()> { Ok(()) }
    fn on_destroy(&self) -> Result<()> { Ok(()) }
    fn on_reload(&self) -> Result<()> { Ok(()) }
}
```

### 数据类型

#### Value

SWA 使用 `tube::Value` 作为统一的数据类型：

```rust
use swa_plugin_sdk::value;

// 创建各种类型的值
let string_val = value!("Hello World");
let number_val = value!(42);
let bool_val = value!(true);
let array_val = value!([1, 2, 3]);
let object_val = value!({
    "name": "John",
    "age": 30,
    "hobbies": ["reading", "coding"]
});
```

#### RequestParameter

API 请求参数结构：

```rust
pub struct RequestParameter {
    pub app: String,           // API 应用名称
    pub module: String,        // 模块名称
    pub method: String,        // 方法名称
    pub parameter: Value,      // 请求参数
    pub authorizer: Option<Authorizer>, // 授权信息
    // ... 其他字段
}
```

## 🔧 配置

### API 插件配置

在 `conf/api_plugins.yml` 中配置 API 路由：

```yaml
# API 应用名称: 插件名称
myapp: "my-plugin"
cms: "cms-plugin"
user: "user-plugin"
```

### 模板配置

插件可以为模板提供数据，支持以下路径：

- `index` 或 `""` - 首页
- `about` - 关于页面
- 其他自定义路径

## 🌐 API 访问

### 请求格式

```
POST /api/{app}/{module}/{method}
Content-Type: application/json

{
  "param1": "value1",
  "param2": "value2"
}
```

### 响应格式

```json
{
  "code": 200,
  "result": {
    "ok": true,
    "data": {
      "message": "Success"
    }
  },
  "message": ""
}
```

### 示例

```bash
# 访问插件 API
curl -X POST "http://localhost:3000/api/myapp/user/info" \
  -H "Content-Type: application/json" \
  -d '{"user_id": 123}'
```

## 📝 开发最佳实践

### 1. 错误处理

```rust
fn api_dispatch(&self, app: &str, param: &RequestParameter) -> Option<Result<Value>> {
    match app {
        "myapp" => {
            // 参数验证
            let user_id = param.parameter.get("user_id")
                .and_then(|v| v.as_u64())
                .ok_or_else(|| tube::error!("缺少 user_id 参数"))?;
            
            // 业务逻辑
            if user_id == 0 {
                return Some(Err(tube::error!("无效的用户ID")));
            }
            
            Some(Ok(value!({
                "ok": true,
                "data": {"user_id": user_id}
            })))
        },
        _ => None
    }
}
```

### 2. 日志记录

```rust
fn data_for(&self, path: &str) -> Option<Value> {
    println!("[my-plugin] 处理路径: {}", path);
    
    match path {
        "index" => {
            println!("[my-plugin] 为主页提供数据");
            Some(value!({
                "title": "我的插件",
                "timestamp": chrono::Utc::now().timestamp()
            }))
        },
        _ => {
            println!("[my-plugin] 未知路径: {}", path);
            None
        }
    }
}
```

### 3. 配置管理

```rust
use std::collections::HashMap;

pub struct MyPlugin {
    config: HashMap<String, String>,
}

impl MyPlugin {
    pub fn new() -> Self {
        let mut config = HashMap::new();
        config.insert("api_key".to_string(), "default_key".to_string());
        config.insert("timeout".to_string(), "30".to_string());
        
        Self { config }
    }
    
    fn get_config(&self, key: &str) -> Option<&String> {
        self.config.get(key)
    }
}
```

### 4. 数据库连接

```rust
use sqlx::{Pool, Postgres};

pub struct MyPlugin {
    db_pool: Option<Pool<Postgres>>,
}

impl MyPlugin {
    pub async fn new() -> Self {
        let db_pool = match sqlx::PgPool::connect("postgresql://...").await {
            Ok(pool) => Some(pool),
            Err(e) => {
                eprintln!("[my-plugin] 数据库连接失败: {}", e);
                None
            }
        };
        
        Self { db_pool }
    }
    
    async fn query_data(&self, query: &str) -> Result<Value> {
        if let Some(pool) = &self.db_pool {
            let rows = sqlx::query(query)
                .fetch_all(pool)
                .await
                .map_err(|e| tube::error!("查询失败: {}", e))?;
            
            // 处理查询结果...
            Ok(value!({"data": "query_result"}))
        } else {
            Err(tube::error!("数据库未连接"))
        }
    }
}
```

## 🔍 调试和测试

### 1. 日志调试

插件可以使用 `println!` 或 `eprintln!` 输出调试信息：

```rust
fn api_dispatch(&self, app: &str, param: &RequestParameter) -> Option<Result<Value>> {
    println!("[my-plugin] API请求: app={}, module={}, method={}", 
             app, param.module, param.method);
    println!("[my-plugin] 请求参数: {:?}", param.parameter);
    
    // 处理逻辑...
}
```

### 2. 单元测试

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_plugin_name() {
        let plugin = MyPlugin::new();
        assert_eq!(plugin.name(), "my-plugin");
    }
    
    #[test]
    fn test_data_for() {
        let plugin = MyPlugin::new();
        let data = plugin.data_for("index");
        assert!(data.is_some());
        
        let value = data.unwrap();
        assert!(value.get("title").is_some());
    }
}
```

### 3. 集成测试

```bash
# 启动 SWA 服务器
./target/release/swa --port 3000 --auto-routes

# 测试模板数据
curl http://localhost:3000/

# 测试 API
curl -X POST "http://localhost:3000/api/myapp/test/method" \
  -H "Content-Type: application/json" \
  -d '{"test": "data"}'
```

## 🚀 部署

### 1. 编译优化

```toml
# Cargo.toml
[profile.release]
opt-level = 3
lto = true
codegen-units = 1
panic = "abort"
```

### 2. 插件目录结构

```
plugins/
├── sdk/                           # SDK 源码
├── my-plugin/                     # 插件源码
│   ├── Cargo.toml
│   └── src/lib.rs
├── libswa_plugin_my_plugin.dylib  # 编译后的动态库
└── conf/
    └── api_plugins.yml            # API 配置
```

### 3. 热重载

SWA 支持插件热重载，修改插件后重新编译并复制动态库即可：

```bash
# 重新编译
cargo build --release

# 复制新版本
cp target/release/libswa_plugin_my_plugin.dylib ../../plugins/

# SWA 会自动重新加载插件
```

## 📋 常见问题

### Q: 插件编译失败怎么办？

A: 检查以下几点：
1. 确保 `crate-type = ["cdylib"]` 配置正确
2. 检查依赖版本是否兼容
3. 确保实现了 `swa_plugin_create` 函数

### Q: 插件加载失败怎么办？

A: 检查以下几点：
1. 动态库文件是否存在且可执行
2. 插件名称是否与配置文件匹配
3. 查看 SWA 启动日志中的错误信息

### Q: API 请求返回 404 怎么办？

A: 检查以下几点：
1. 确认 `conf/api_plugins.yml` 配置正确
2. 插件是否实现了 `api_dispatch` 方法
3. API 路径格式是否正确：`/api/{app}/{module}/{method}`

### Q: 模板数据不显示怎么办？

A: 检查以下几点：
1. 插件是否实现了 `data_for` 方法
2. 路径匹配是否正确
3. 返回的数据格式是否符合模板要求

## 📞 支持

如有问题，请：

1. 查看 SWA 启动日志
2. 检查插件编译输出
3. 参考示例插件代码
4. 提交 Issue 到项目仓库

---

**Happy Coding! 🎉**
