# SWA 插件 API 参考

## 📋 目录

- [核心接口](#核心接口)
- [数据类型](#数据类型)
- [配置系统](#配置系统)
- [错误处理](#错误处理)
- [示例代码](#示例代码)

## 🔧 核心接口

### SwaPlugin Trait

```rust
pub trait SwaPlugin: Send + Sync {
    /// 返回插件名称，用于标识和日志记录
    fn name(&self) -> &'static str;
    
    /// 为指定路径提供模板数据
    /// 
    /// # 参数
    /// - `path`: 请求路径，如 "index", "about", "contact" 等
    /// 
    /// # 返回值
    /// - `Some(Value)`: 提供的数据，会被合并到模板上下文中
    /// - `None`: 不为此路径提供数据
    fn data_for(&self, path: &str) -> Option<Value> { None }
    
    /// 处理 API 请求分发
    /// 
    /// # 参数
    /// - `app`: API 应用名称，如 "cms", "user", "order" 等
    /// - `param`: 请求参数，包含完整的请求信息
    /// 
    /// # 返回值
    /// - `Some(Ok(Value))`: 成功处理，返回数据
    /// - `Some(Err(Result))`: 处理失败，返回错误
    /// - `None`: 不处理此 API 请求
    fn api_dispatch(&self, app: &str, param: &RequestParameter) -> Option<Result<Value>> { None }
}
```

### PluginMetadata Trait (可选)

```rust
pub trait PluginMetadata {
    /// 返回插件信息
    fn info(&self) -> PluginInfo;
    
    /// 返回插件支持的功能列表
    fn capabilities(&self) -> Vec<&'static str> { vec![] }
    
    /// 检查插件是否支持某个功能
    fn supports(&self, capability: &str) -> bool { false }
}

#[derive(Debug, Clone)]
pub struct PluginInfo {
    pub name: &'static str,
    pub version: &'static str,
    pub description: &'static str,
    pub author: &'static str,
}
```

### PluginLifecycle Trait (可选)

```rust
pub trait PluginLifecycle {
    /// 插件初始化时调用
    fn on_init(&self) -> Result<()> { Ok(()) }
    
    /// 插件卸载时调用
    fn on_destroy(&self) -> Result<()> { Ok(()) }
    
    /// 插件重新加载时调用
    fn on_reload(&self) -> Result<()> { Ok(()) }
}
```

## 📊 数据类型

### Value 类型

SWA 使用 `tube::Value` 作为统一的数据类型，支持以下类型：

```rust
use swa_plugin_sdk::value;

// 基本类型
let null_val = value!(null);
let bool_val = value!(true);
let int_val = value!(42);
let float_val = value!(3.14);
let string_val = value!("Hello World");

// 复合类型
let array_val = value!([1, 2, 3, "four"]);
let object_val = value!({
    "name": "John",
    "age": 30,
    "active": true,
    "tags": ["developer", "rust"]
});

// 嵌套结构
let complex_val = value!({
    "user": {
        "id": 123,
        "profile": {
            "name": "John Doe",
            "email": "john@example.com"
        }
    },
    "permissions": ["read", "write"],
    "metadata": {
        "created_at": "2024-01-01T00:00:00Z",
        "version": 1
    }
});
```

### RequestParameter 结构

```rust
pub struct RequestParameter {
    /// API 应用名称
    pub app: String,
    
    /// 模块名称
    pub module: String,
    
    /// 方法名称
    pub method: String,
    
    /// 请求参数 (JSON 格式)
    pub parameter: Value,
    
    /// 授权信息 (可选)
    pub authorizer: Option<Authorizer>,
    
    /// 请求头信息
    pub headers: HashMap<String, String>,
    
    /// 查询参数
    pub query: HashMap<String, String>,
    
    /// 错误信息
    pub error: ErrorInfo,
}

pub struct Authorizer {
    /// 访问者信息
    pub visitor: Visitor,
    
    /// 权限信息
    pub permissions: Vec<String>,
}

pub struct Visitor {
    /// 用户ID
    pub id: u64,
    
    /// 用户名
    pub name: String,
    
    /// 平台类型
    pub platform: u8,
}
```

## ⚙️ 配置系统

### API 插件配置

文件位置：`conf/api_plugins.yml`

```yaml
# API 应用名称: 插件名称
cms: "cms-plugin"
user: "user-plugin"
order: "order-plugin"
payment: "payment-plugin"

# 支持多个应用映射到同一个插件
admin: "cms-plugin"
dashboard: "cms-plugin"
```

### 插件配置示例

```rust
use std::collections::HashMap;

pub struct MyPlugin {
    config: HashMap<String, String>,
}

impl MyPlugin {
    pub fn new() -> Self {
        let mut config = HashMap::new();
        
        // 从环境变量读取配置
        config.insert("api_key".to_string(), 
            std::env::var("MY_PLUGIN_API_KEY").unwrap_or_default());
        config.insert("timeout".to_string(), 
            std::env::var("MY_PLUGIN_TIMEOUT").unwrap_or_else(|_| "30".to_string()));
        config.insert("debug".to_string(), 
            std::env::var("MY_PLUGIN_DEBUG").unwrap_or_else(|_| "false".to_string()));
        
        Self { config }
    }
    
    fn get_config(&self, key: &str) -> Option<&String> {
        self.config.get(key)
    }
    
    fn is_debug(&self) -> bool {
        self.get_config("debug")
            .map(|v| v == "true")
            .unwrap_or(false)
    }
}
```

## ❌ 错误处理

### 错误类型

```rust
use swa_plugin_sdk::PluginError;

#[derive(Debug, thiserror::Error)]
pub enum PluginError {
    #[error("插件初始化失败: {0}")]
    InitializationFailed(String),
    
    #[error("插件不支持的功能: {0}")]
    UnsupportedCapability(String),
    
    #[error("插件内部错误: {0}")]
    InternalError(String),
    
    #[error("参数验证失败: {0}")]
    ValidationError(String),
    
    #[error("数据库错误: {0}")]
    DatabaseError(String),
}
```

### 错误处理示例

```rust
fn api_dispatch(&self, app: &str, param: &RequestParameter) -> Option<Result<Value>> {
    match app {
        "myapp" => {
            // 参数验证
            let user_id = match param.parameter.get("user_id") {
                Some(v) => match v.as_u64() {
                    Some(id) if id > 0 => id,
                    _ => return Some(Err(tube::error!("无效的用户ID"))),
                },
                None => return Some(Err(tube::error!("缺少 user_id 参数"))),
            };
            
            // 业务逻辑处理
            match self.process_user(user_id).await {
                Ok(result) => Some(Ok(value!({
                    "ok": true,
                    "data": result
                }))),
                Err(e) => Some(Err(tube::error!("处理失败: {}", e))),
            }
        },
        _ => None
    }
}
```

## 💡 示例代码

### 完整的插件示例

```rust
use swa_plugin_sdk::{SwaPlugin, value, Value, Result, RequestParameter};
use std::collections::HashMap;

pub struct UserManagementPlugin {
    config: HashMap<String, String>,
    users: HashMap<u64, User>,
}

#[derive(Debug, Clone)]
struct User {
    id: u64,
    name: String,
    email: String,
    active: bool,
}

impl SwaPlugin for UserManagementPlugin {
    fn name(&self) -> &'static str {
        "user-management"
    }
    
    fn data_for(&self, path: &str) -> Option<Value> {
        match path {
            "users" => {
                let user_list: Vec<Value> = self.users.values()
                    .map(|user| value!({
                        "id": user.id,
                        "name": user.name.clone(),
                        "email": user.email.clone(),
                        "active": user.active
                    }))
                    .collect();
                
                Some(value!({
                    "title": "用户管理",
                    "users": user_list,
                    "total": self.users.len()
                }))
            },
            _ => None
        }
    }
    
    fn api_dispatch(&self, app: &str, param: &RequestParameter) -> Option<Result<Value>> {
        match app {
            "user" => self.handle_user_api(param),
            _ => None
        }
    }
}

impl UserManagementPlugin {
    pub fn new() -> Self {
        let mut config = HashMap::new();
        config.insert("max_users".to_string(), "1000".to_string());
        
        let mut users = HashMap::new();
        users.insert(1, User {
            id: 1,
            name: "Admin".to_string(),
            email: "admin@example.com".to_string(),
            active: true,
        });
        
        Self { config, users }
    }
    
    fn handle_user_api(&self, param: &RequestParameter) -> Option<Result<Value>> {
        match param.method.as_str() {
            "list" => self.list_users(),
            "get" => self.get_user(param),
            "create" => self.create_user(param),
            "update" => self.update_user(param),
            "delete" => self.delete_user(param),
            _ => Some(Err(tube::error!("不支持的方法: {}", param.method))),
        }
    }
    
    fn list_users(&self) -> Option<Result<Value>> {
        let user_list: Vec<Value> = self.users.values()
            .map(|user| value!({
                "id": user.id,
                "name": user.name.clone(),
                "email": user.email.clone(),
                "active": user.active
            }))
            .collect();
        
        Some(Ok(value!({
            "ok": true,
            "data": {
                "users": user_list,
                "total": self.users.len()
            }
        })))
    }
    
    fn get_user(&self, param: &RequestParameter) -> Option<Result<Value>> {
        let user_id = param.parameter.get("id")
            .and_then(|v| v.as_u64())
            .ok_or_else(|| tube::error!("缺少用户ID"))?;
        
        match self.users.get(&user_id) {
            Some(user) => Some(Ok(value!({
                "ok": true,
                "data": {
                    "id": user.id,
                    "name": user.name.clone(),
                    "email": user.email.clone(),
                    "active": user.active
                }
            }))),
            None => Some(Err(tube::error!("用户不存在"))),
        }
    }
    
    fn create_user(&self, param: &RequestParameter) -> Option<Result<Value>> {
        let name = param.parameter.get("name")
            .and_then(|v| v.as_str())
            .ok_or_else(|| tube::error!("缺少用户名"))?;
        
        let email = param.parameter.get("email")
            .and_then(|v| v.as_str())
            .ok_or_else(|| tube::error!("缺少邮箱"))?;
        
        let new_id = self.users.keys().max().unwrap_or(&0) + 1;
        let user = User {
            id: new_id,
            name: name.to_string(),
            email: email.to_string(),
            active: true,
        };
        
        // 在实际应用中，这里应该保存到数据库
        // self.save_user(&user).await?;
        
        Some(Ok(value!({
            "ok": true,
            "data": {
                "id": user.id,
                "name": user.name,
                "email": user.email,
                "active": user.active
            }
        })))
    }
    
    fn update_user(&self, param: &RequestParameter) -> Option<Result<Value>> {
        // 实现更新用户逻辑
        Some(Ok(value!({
            "ok": true,
            "data": {"message": "用户更新成功"}
        })))
    }
    
    fn delete_user(&self, param: &RequestParameter) -> Option<Result<Value>> {
        // 实现删除用户逻辑
        Some(Ok(value!({
            "ok": true,
            "data": {"message": "用户删除成功"}
        })))
    }
}

impl UserManagementPlugin {
    pub fn new() -> Self {
        Self::new()
    }
}

#[no_mangle]
pub extern "C" fn swa_plugin_create() -> *mut dyn SwaPlugin {
    Box::into_raw(Box::new(UserManagementPlugin::new()))
}
```

### 配置示例

```yaml
# conf/api_plugins.yml
user: "user-management"
admin: "user-management"
```

### API 调用示例

```bash
# 获取用户列表
curl -X POST "http://localhost:3000/api/user/user/list" \
  -H "Content-Type: application/json" \
  -d '{}'

# 获取特定用户
curl -X POST "http://localhost:3000/api/user/user/get" \
  -H "Content-Type: application/json" \
  -d '{"id": 1}'

# 创建新用户
curl -X POST "http://localhost:3000/api/user/user/create" \
  -H "Content-Type: application/json" \
  -d '{"name": "John Doe", "email": "john@example.com"}'
```

## 🔍 调试技巧

### 1. 日志输出

```rust
fn api_dispatch(&self, app: &str, param: &RequestParameter) -> Option<Result<Value>> {
    println!("[user-plugin] API请求: app={}, module={}, method={}", 
             app, param.module, param.method);
    println!("[user-plugin] 请求参数: {:?}", param.parameter);
    println!("[user-plugin] 查询参数: {:?}", param.query);
    
    // 处理逻辑...
    
    println!("[user-plugin] 处理完成");
    Some(Ok(value!({"ok": true})))
}
```

### 2. 错误追踪

```rust
fn process_data(&self, data: &Value) -> Result<Value> {
    println!("[user-plugin] 开始处理数据: {:?}", data);
    
    let result = match data.get("type") {
        Some(t) => {
            println!("[user-plugin] 数据类型: {:?}", t);
            self.handle_type(t)?
        },
        None => {
            println!("[user-plugin] 警告: 缺少type字段");
            return Err(tube::error!("缺少type字段"));
        }
    };
    
    println!("[user-plugin] 处理结果: {:?}", result);
    Ok(result)
}
```

### 3. 性能监控

```rust
use std::time::Instant;

fn api_dispatch(&self, app: &str, param: &RequestParameter) -> Option<Result<Value>> {
    let start_time = Instant::now();
    
    let result = match app {
        "user" => self.handle_user_api(param),
        _ => None,
    };
    
    let duration = start_time.elapsed();
    println!("[user-plugin] API处理耗时: {:?}", duration);
    
    result
}
```

---

这份 API 参考文档提供了完整的插件开发指南，包括所有核心接口、数据类型、配置系统和实际示例。开发者可以根据这份文档快速上手 SWA 插件开发。
