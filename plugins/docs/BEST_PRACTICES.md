# SWA 插件开发最佳实践

## 📋 目录

- [代码组织](#代码组织)
- [错误处理](#错误处理)
- [性能优化](#性能优化)
- [安全性](#安全性)
- [测试策略](#测试策略)
- [部署和监控](#部署和监控)

## 🏗️ 代码组织

### 1. 项目结构

```
my-plugin/
├── Cargo.toml              # 项目配置
├── src/
│   ├── lib.rs              # 插件主入口
│   ├── config.rs           # 配置管理
│   ├── handlers/           # 请求处理器
│   │   ├── mod.rs
│   │   ├── api.rs          # API 处理
│   │   └── template.rs     # 模板数据处理
│   ├── models/             # 数据模型
│   │   ├── mod.rs
│   │   └── user.rs
│   ├── services/           # 业务逻辑
│   │   ├── mod.rs
│   │   └── user_service.rs
│   └── utils/              # 工具函数
│       ├── mod.rs
│       └── validation.rs
├── tests/                  # 集成测试
│   └── integration_tests.rs
└── README.md               # 插件文档
```

### 2. 模块化设计

```rust
// src/lib.rs
use swa_plugin_sdk::{SwaPlugin, value, Value, Result, RequestParameter};

mod config;
mod handlers;
mod models;
mod services;
mod utils;

use config::PluginConfig;
use handlers::{ApiHandler, TemplateHandler};
use services::UserService;

pub struct MyPlugin {
    config: PluginConfig,
    user_service: UserService,
}

impl SwaPlugin for MyPlugin {
    fn name(&self) -> &'static str {
        "my-plugin"
    }
    
    fn data_for(&self, path: &str) -> Option<Value> {
        TemplateHandler::handle(path, &self.config)
    }
    
    fn api_dispatch(&self, app: &str, param: &RequestParameter) -> Option<Result<Value>> {
        ApiHandler::handle(app, param, &self.user_service)
    }
}
```

### 3. 配置管理

```rust
// src/config.rs
use std::collections::HashMap;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PluginConfig {
    pub database_url: String,
    pub api_timeout: u64,
    pub max_connections: u32,
    pub debug_mode: bool,
    pub features: HashMap<String, bool>,
}

impl PluginConfig {
    pub fn from_env() -> Result<Self, Box<dyn std::error::Error>> {
        Ok(PluginConfig {
            database_url: std::env::var("DATABASE_URL")
                .unwrap_or_else(|_| "sqlite://:memory:".to_string()),
            api_timeout: std::env::var("API_TIMEOUT")
                .unwrap_or_else(|_| "30".to_string())
                .parse()?,
            max_connections: std::env::var("MAX_CONNECTIONS")
                .unwrap_or_else(|_| "10".to_string())
                .parse()?,
            debug_mode: std::env::var("DEBUG_MODE")
                .unwrap_or_else(|_| "false".to_string())
                .parse()?,
            features: Self::load_features(),
        })
    }
    
    fn load_features() -> HashMap<String, bool> {
        let mut features = HashMap::new();
        features.insert("user_management".to_string(), true);
        features.insert("analytics".to_string(), false);
        features
    }
    
    pub fn is_feature_enabled(&self, feature: &str) -> bool {
        self.features.get(feature).copied().unwrap_or(false)
    }
}
```

## ❌ 错误处理

### 1. 自定义错误类型

```rust
// src/errors.rs
use thiserror::Error;

#[derive(Error, Debug)]
pub enum PluginError {
    #[error("配置错误: {0}")]
    ConfigError(String),
    
    #[error("数据库错误: {0}")]
    DatabaseError(#[from] sqlx::Error),
    
    #[error("验证错误: {0}")]
    ValidationError(String),
    
    #[error("业务逻辑错误: {0}")]
    BusinessError(String),
    
    #[error("外部服务错误: {0}")]
    ExternalServiceError(String),
}

pub type PluginResult<T> = Result<T, PluginError>;
```

### 2. 错误处理策略

```rust
// src/handlers/api.rs
use crate::errors::{PluginError, PluginResult};

pub struct ApiHandler;

impl ApiHandler {
    pub fn handle(
        app: &str, 
        param: &RequestParameter, 
        user_service: &UserService
    ) -> Option<Result<Value>> {
        match app {
            "user" => Self::handle_user_api(param, user_service),
            _ => None,
        }
    }
    
    fn handle_user_api(
        param: &RequestParameter, 
        user_service: &UserService
    ) -> Option<Result<Value>> {
        let result = async_std::task::block_on(async {
            match param.method.as_str() {
                "create" => Self::create_user(param, user_service).await,
                "update" => Self::update_user(param, user_service).await,
                "delete" => Self::delete_user(param, user_service).await,
                _ => Err(PluginError::ValidationError(
                    format!("不支持的方法: {}", param.method)
                )),
            }
        });
        
        Some(result.map_err(|e| tube::error!("{}", e)))
    }
    
    async fn create_user(
        param: &RequestParameter, 
        user_service: &UserService
    ) -> PluginResult<Value> {
        // 参数验证
        let user_data = Self::validate_user_data(&param.parameter)?;
        
        // 业务逻辑
        let user = user_service.create_user(user_data).await?;
        
        Ok(value!({
            "ok": true,
            "data": {
                "id": user.id,
                "name": user.name,
                "email": user.email
            }
        }))
    }
    
    fn validate_user_data(data: &Value) -> PluginResult<UserData> {
        let name = data.get("name")
            .and_then(|v| v.as_str())
            .ok_or_else(|| PluginError::ValidationError("缺少用户名".to_string()))?;
        
        let email = data.get("email")
            .and_then(|v| v.as_str())
            .ok_or_else(|| PluginError::ValidationError("缺少邮箱".to_string()))?;
        
        if name.is_empty() {
            return Err(PluginError::ValidationError("用户名不能为空".to_string()));
        }
        
        if !email.contains('@') {
            return Err(PluginError::ValidationError("邮箱格式不正确".to_string()));
        }
        
        Ok(UserData {
            name: name.to_string(),
            email: email.to_string(),
        })
    }
}
```

## ⚡ 性能优化

### 1. 连接池管理

```rust
// src/services/database.rs
use sqlx::{Pool, Postgres, PgPool};
use std::sync::Arc;

pub struct DatabaseService {
    pool: Arc<PgPool>,
}

impl DatabaseService {
    pub async fn new(database_url: &str) -> Result<Self, sqlx::Error> {
        let pool = PgPool::connect(database_url).await?;
        
        // 配置连接池
        sqlx::query("SET statement_timeout = '30s'")
            .execute(&pool)
            .await?;
        
        Ok(Self {
            pool: Arc::new(pool),
        })
    }
    
    pub fn get_pool(&self) -> &PgPool {
        &self.pool
    }
}
```

### 2. 缓存策略

```rust
// src/services/cache.rs
use std::collections::HashMap;
use std::sync::RwLock;
use std::time::{Duration, Instant};

#[derive(Debug, Clone)]
struct CacheEntry<T> {
    value: T,
    expires_at: Instant,
}

pub struct CacheService<T> {
    cache: RwLock<HashMap<String, CacheEntry<T>>>,
    default_ttl: Duration,
}

impl<T: Clone> CacheService<T> {
    pub fn new(default_ttl: Duration) -> Self {
        Self {
            cache: RwLock::new(HashMap::new()),
            default_ttl,
        }
    }
    
    pub fn get(&self, key: &str) -> Option<T> {
        let cache = self.cache.read().ok()?;
        let entry = cache.get(key)?;
        
        if entry.expires_at < Instant::now() {
            drop(cache);
            self.remove(key);
            return None;
        }
        
        Some(entry.value.clone())
    }
    
    pub fn set(&self, key: String, value: T) {
        let entry = CacheEntry {
            value,
            expires_at: Instant::now() + self.default_ttl,
        };
        
        if let Ok(mut cache) = self.cache.write() {
            cache.insert(key, entry);
        }
    }
    
    pub fn remove(&self, key: &str) {
        if let Ok(mut cache) = self.cache.write() {
            cache.remove(key);
        }
    }
}
```

### 3. 异步处理

```rust
// src/services/async_processor.rs
use std::sync::Arc;
use tokio::sync::mpsc;
use tokio::task::JoinHandle;

pub struct AsyncProcessor {
    sender: mpsc::UnboundedSender<ProcessingTask>,
    handle: JoinHandle<()>,
}

pub enum ProcessingTask {
    ProcessUser(u64),
    SendEmail(String, String),
    GenerateReport(String),
}

impl AsyncProcessor {
    pub fn new() -> Self {
        let (sender, mut receiver) = mpsc::unbounded_channel();
        
        let handle = tokio::spawn(async move {
            while let Some(task) = receiver.recv().await {
                match task {
                    ProcessingTask::ProcessUser(user_id) => {
                        Self::process_user_async(user_id).await;
                    },
                    ProcessingTask::SendEmail(to, content) => {
                        Self::send_email_async(to, content).await;
                    },
                    ProcessingTask::GenerateReport(report_type) => {
                        Self::generate_report_async(report_type).await;
                    },
                }
            }
        });
        
        Self { sender, handle }
    }
    
    pub fn process_user(&self, user_id: u64) {
        let _ = self.sender.send(ProcessingTask::ProcessUser(user_id));
    }
    
    async fn process_user_async(user_id: u64) {
        // 异步处理用户数据
        println!("Processing user: {}", user_id);
        tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
    }
}
```

## 🔒 安全性

### 1. 输入验证

```rust
// src/utils/validation.rs
use regex::Regex;
use swa_plugin_sdk::Value;

pub struct Validator;

impl Validator {
    pub fn validate_email(email: &str) -> bool {
        let email_regex = Regex::new(r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$")
            .expect("Invalid email regex");
        email_regex.is_match(email)
    }
    
    pub fn validate_password(password: &str) -> bool {
        password.len() >= 8 && 
        password.chars().any(|c| c.is_ascii_lowercase()) &&
        password.chars().any(|c| c.is_ascii_uppercase()) &&
        password.chars().any(|c| c.is_ascii_digit())
    }
    
    pub fn sanitize_string(input: &str) -> String {
        input
            .chars()
            .filter(|c| c.is_alphanumeric() || c.is_whitespace() || *c == '@' || *c == '.')
            .collect()
    }
    
    pub fn validate_json_structure(data: &Value, required_fields: &[&str]) -> Result<(), String> {
        for field in required_fields {
            if data.get(field).is_none() {
                return Err(format!("缺少必需字段: {}", field));
            }
        }
        Ok(())
    }
}
```

### 2. 权限控制

```rust
// src/services/auth.rs
use std::collections::HashSet;

pub struct AuthService {
    permissions: HashSet<String>,
}

impl AuthService {
    pub fn new() -> Self {
        let mut permissions = HashSet::new();
        permissions.insert("user:read".to_string());
        permissions.insert("user:write".to_string());
        permissions.insert("admin:all".to_string());
        
        Self { permissions }
    }
    
    pub fn has_permission(&self, permission: &str) -> bool {
        self.permissions.contains(permission)
    }
    
    pub fn check_user_permission(&self, user_id: u64, action: &str) -> bool {
        // 根据用户ID和操作检查权限
        match action {
            "read" => self.has_permission("user:read"),
            "write" => self.has_permission("user:write"),
            "admin" => self.has_permission("admin:all"),
            _ => false,
        }
    }
}
```

## 🧪 测试策略

### 1. 单元测试

```rust
// src/tests/unit_tests.rs
#[cfg(test)]
mod tests {
    use super::*;
    use swa_plugin_sdk::{value, RequestParameter};

    #[test]
    fn test_plugin_name() {
        let plugin = MyPlugin::new();
        assert_eq!(plugin.name(), "my-plugin");
    }
    
    #[test]
    fn test_data_for_index() {
        let plugin = MyPlugin::new();
        let data = plugin.data_for("index");
        
        assert!(data.is_some());
        let value = data.unwrap();
        assert!(value.get("title").is_some());
    }
    
    #[test]
    fn test_api_dispatch() {
        let plugin = MyPlugin::new();
        let param = create_test_parameter();
        
        let result = plugin.api_dispatch("user", &param);
        assert!(result.is_some());
        
        let response = result.unwrap();
        assert!(response.is_ok());
    }
    
    fn create_test_parameter() -> RequestParameter {
        RequestParameter {
            app: "user".to_string(),
            module: "user".to_string(),
            method: "create".to_string(),
            parameter: value!({
                "name": "Test User",
                "email": "test@example.com"
            }),
            authorizer: None,
            headers: std::collections::HashMap::new(),
            query: std::collections::HashMap::new(),
            error: tube_web::ErrorInfo::default(),
        }
    }
}
```

### 2. 集成测试

```rust
// tests/integration_tests.rs
use swa_plugin_sdk::{SwaPlugin, value};

#[test]
fn test_plugin_loading() {
    // 测试插件加载
    let plugin = create_plugin();
    assert_eq!(plugin.name(), "my-plugin");
}

#[test]
fn test_api_integration() {
    // 测试 API 集成
    let plugin = create_plugin();
    let param = create_api_parameter();
    
    let result = plugin.api_dispatch("user", &param);
    assert!(result.is_some());
    
    let response = result.unwrap();
    assert!(response.is_ok());
    
    let data = response.unwrap();
    assert!(data.get("ok").and_then(|v| v.as_bool()).unwrap_or(false));
}

fn create_plugin() -> Box<dyn SwaPlugin> {
    // 创建插件实例
    Box::new(MyPlugin::new())
}

fn create_api_parameter() -> RequestParameter {
    // 创建测试参数
    RequestParameter {
        app: "user".to_string(),
        module: "user".to_string(),
        method: "create".to_string(),
        parameter: value!({
            "name": "Integration Test User",
            "email": "integration@example.com"
        }),
        authorizer: None,
        headers: std::collections::HashMap::new(),
        query: std::collections::HashMap::new(),
        error: tube_web::ErrorInfo::default(),
    }
}
```

## 🚀 部署和监控

### 1. 构建脚本

```bash
#!/bin/bash
# build.sh

set -e

echo "Building SWA plugin..."

# 清理之前的构建
cargo clean

# 编译插件
cargo build --release

# 复制到插件目录
cp target/release/libswa_plugin_*.dylib ../../plugins/

echo "Plugin built and deployed successfully!"

# 运行测试
cargo test

echo "All tests passed!"
```

### 2. 监控和日志

```rust
// src/utils/logging.rs
use std::time::Instant;

pub struct PerformanceLogger {
    start_time: Instant,
    operation: String,
}

impl PerformanceLogger {
    pub fn new(operation: &str) -> Self {
        Self {
            start_time: Instant::now(),
            operation: operation.to_string(),
        }
    }
    
    pub fn log_duration(&self) {
        let duration = self.start_time.elapsed();
        println!("[PERF] {} completed in {:?}", self.operation, duration);
    }
}

impl Drop for PerformanceLogger {
    fn drop(&mut self) {
        self.log_duration();
    }
}

// 使用示例
fn api_dispatch(&self, app: &str, param: &RequestParameter) -> Option<Result<Value>> {
    let _perf_logger = PerformanceLogger::new(&format!("api_dispatch_{}", app));
    
    // 处理逻辑...
    
    Some(Ok(value!({"ok": true})))
}
```

### 3. 健康检查

```rust
// src/services/health.rs
pub struct HealthService {
    database_healthy: bool,
    external_services_healthy: bool,
}

impl HealthService {
    pub fn new() -> Self {
        Self {
            database_healthy: true,
            external_services_healthy: true,
        }
    }
    
    pub async fn check_health(&self) -> Value {
        let mut status = "healthy";
        let mut issues = Vec::new();
        
        if !self.database_healthy {
            status = "unhealthy";
            issues.push("Database connection failed");
        }
        
        if !self.external_services_healthy {
            status = "degraded";
            issues.push("External service unavailable");
        }
        
        value!({
            "status": status,
            "timestamp": chrono::Utc::now().timestamp(),
            "issues": issues,
            "version": env!("CARGO_PKG_VERSION")
        })
    }
}
```

## 📋 总结

遵循这些最佳实践可以确保你的 SWA 插件：

1. **可维护**：清晰的代码结构和模块化设计
2. **可靠**：完善的错误处理和测试覆盖
3. **高性能**：优化的数据库连接和缓存策略
4. **安全**：输入验证和权限控制
5. **可监控**：日志记录和健康检查

记住：好的插件不仅仅是功能实现，更是工程实践的体现！
