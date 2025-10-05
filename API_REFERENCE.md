# SWA API 参考文档

## 📖 概述

本文档详细介绍了 SWA 应用的 API 接口，包括路由配置、插件接口、中间件接口和模板系统接口。

## 🛣️ 路由系统

### 路由配置格式

```json
{
  "domains": [
    {
      "domain": "localhost",
      "theme": "default",
      "routes": [
        {
          "path": "/",
          "method": "GET",
          "handler": "index",
          "template": "index",
          "description": "首页",
          "enabled": true
        },
        {
          "path": "/content/{id}",
          "method": "GET",
          "handler": "content",
          "template": "content",
          "description": "内容详情页",
          "enabled": true
        }
      ]
    }
  ],
  "default_theme": "default",
  "static_routes": [
    {
      "path": "/health",
      "method": "GET",
      "handler": "health_check",
      "template": "json",
      "description": "健康检查",
      "enabled": true
    }
  ]
}
```

### 路由参数

| 参数 | 类型 | 必需 | 描述 |
|------|------|------|------|
| `path` | string | ✅ | 路由路径，支持参数占位符 `{id}` |
| `method` | string | ✅ | HTTP 方法 (GET, POST, PUT, DELETE) |
| `handler` | string | ✅ | 处理器名称，对应插件中的处理逻辑 |
| `template` | string | ✅ | 模板名称，用于渲染页面 |
| `description` | string | ❌ | 路由描述 |
| `enabled` | boolean | ❌ | 是否启用，默认 true |

### 路径参数

路径中的 `{param}` 格式会被解析为路径参数：

```json
{
  "path": "/content/{id}",
  "handler": "content"
}
```

在插件中可以通过 `param.url_path` 获取完整路径，并解析参数：

```rust
let content_id = param.url_path
    .split('/')
    .last()
    .and_then(|s| s.parse::<i64>().ok())
    .unwrap_or(0);
```

## 🔌 插件接口

### SwaPlugin Trait

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

### RequestParameter 结构

```rust
pub struct RequestParameter {
    pub app: String,           // 应用名称
    pub module: String,        // 模块名称
    pub method: String,        // 方法名称
    pub path: String,          // 请求路径
    pub url_path: String,      // 完整URL路径
    pub parameter: Parameter,  // 请求参数
    pub value: Value,          // 请求数据
    pub authorizer: Option<Authorizer>, // 授权信息
}
```

### Parameter 结构

```rust
pub struct Parameter {
    pub headers: HashMap<String, String>,    // 请求头
    pub client_ip: String,                   // 客户端IP
    pub content_type: String,                // 内容类型
    pub cookie: String,                      // Cookie信息
    pub version: String,                     // 版本信息
    pub authorization: String,               // 授权信息
    pub timestamp: u64,                      // 时间戳
    pub app_id: String,                      // 应用ID
    pub sign: String,                        // 签名
    pub terminal: String,                    // 终端信息
    pub app_platform: AppPlatform,          // 应用平台
    pub os: String,                          // 操作系统
    pub api_manage: bool,                    // 是否管理端接口
    pub inner_request: bool,                 // 是否内部调用
    pub ext_data: HashMap<String, String>,   // 扩展数据
    pub query: HashMap<String, String>,      // 查询参数
    pub provider: (i8, u64),                // 数据提供者
    pub rent: (i8, u64),                    // 租户信息
}
```

### PluginConfig 结构

```rust
pub struct PluginConfig {
    pub plugin_data: tube::Value,    // 插件特定配置
    pub app_config: tube::Value,     // 应用完整配置
    pub config_path: Option<String>, // 配置文件路径
}
```

## 🛡️ 中间件接口

### IMiddleware Trait

```rust
pub trait IMiddleware {
    /// 处理请求
    async fn process(&self, req: &MiddlewareRequest, ctx: &mut MiddlewareContext) -> Result<Option<HttpResponse>, actix_web::Error>;
}
```

### MiddlewareRequest 结构

```rust
pub struct MiddlewareRequest {
    pub path: String,
    pub method: String,
    pub headers: HeaderMap,
    pub query: Query<HashMap<String, String>>,
    pub client_ip: String,
}
```

### MiddlewareContext 结构

```rust
pub struct MiddlewareContext {
    pub theme: String,                    // 当前主题
    pub route_config: Option<RouteConfig>, // 路由配置
    pub template_info: Option<TemplateInfo>, // 模板信息
    pub plugins: Option<Arc<PluginRegistry>>, // 插件注册表
    pub req: Option<RequestParameter>,    // 请求参数
    pub data: HashMap<String, Value>,     // 上下文数据
}
```

### 中间件执行顺序

1. **LoggingMiddleware** - 日志记录
2. **IpAccessControlMiddleware** - IP访问控制
3. **CacheMiddleware** - 缓存检查
4. **ThemeSelectorMiddleware** - 主题选择
5. **RouteMatcherMiddleware** - 路由匹配
6. **ContextDataMiddleware** - 上下文数据注入
7. **AuthMiddleware** - 认证处理

## 🎨 模板系统

### 模板查找顺序

1. **Public目录**: `wwwroot/templates/public/{path}/index.html`
2. **主题目录**: `wwwroot/templates/themes/{theme}/{template}.html`
3. **根目录**: `wwwroot/templates/{template}.html`
4. **404处理**: 返回错误页面

### 模板数据注入

模板渲染时会自动注入以下数据：

```rust
let template_data = value!({
    // 插件数据
    "pluginData": plugin_data,
    
    // 全局数据
    "globalData": global_data,
    
    // 上下文数据
    "contextData": context_data,
    
    // 系统数据
    "theme": ctx.theme,
    "timestamp": chrono::Utc::now().timestamp(),
    "version": env!("CARGO_PKG_VERSION"),
});
```

### Handlebars 助手函数

```rust
// 内置助手函数
{{#if condition}}...{{/if}}           // 条件判断
{{#each items}}...{{/each}}           // 循环遍历
{{#with object}}...{{/with}}          // 对象上下文
{{format_date timestamp}}             // 日期格式化
{{format_number number}}              // 数字格式化
{{truncate text length}}              // 文本截断
{{url_encode text}}                   // URL编码
{{html_escape text}}                  // HTML转义
```

## 🔧 配置系统

### 主配置文件 (config.yaml)

```yaml
# 数据库配置
database:
  host: localhost
  port: 3306
  username: root
  password: password
  database: swa_db
  pool_size: 10
  timeout: 30

# Redis配置
redis:
  host: localhost
  port: 6379
  password: ""
  db: 0
  pool_size: 10

# 应用配置
app:
  port: 8081
  host: "0.0.0.0"
  debug: true
  workers: 4

# 日志配置
logging:
  level: "info"
  file: "logs/app.log"
  max_size: "100MB"
  max_files: 10

# 安全配置
security:
  secret_key: "your-secret-key"
  session_timeout: 3600
  rate_limit: 1000

# IP访问控制
ip_access_control:
  enabled: true

# 缓存配置
cache:
  enabled: true
  ttl: 300
  max_size: "1GB"
```

### 插件配置文件 (plugins/config.json)

```json
{
  "plugins": [
    {
      "name": "bas-site",
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

## 📊 响应格式

### 成功响应

```json
{
  "status": "success",
  "data": {
    "title": "页面标题",
    "content": "页面内容",
    "timestamp": 1640995200
  },
  "message": "操作成功"
}
```

### 错误响应

```json
{
  "status": "error",
  "error": {
    "code": 40098,
    "message": "请求提供的参数错误",
    "details": "具体错误信息"
  },
  "timestamp": 1640995200
}
```

### 分页响应

```json
{
  "status": "success",
  "data": {
    "items": [...],
    "pagination": {
      "page": 1,
      "size": 20,
      "total": 100,
      "pages": 5
    }
  }
}
```

## 🔍 调试接口

### 健康检查

```http
GET /health
```

响应：
```json
{
  "status": "ok",
  "timestamp": 1640995200,
  "version": "1.0.0",
  "services": {
    "database": "ok",
    "redis": "ok",
    "plugins": "ok"
  }
}
```

### 系统信息

```http
GET /api/system/info
```

响应：
```json
{
  "app": {
    "name": "SWA",
    "version": "1.0.0",
    "environment": "production"
  },
  "system": {
    "os": "Linux",
    "arch": "x86_64",
    "memory": "8GB",
    "cpu": "4 cores"
  },
  "plugins": [
    {
      "name": "bas-site",
      "version": "1.0.0",
      "enabled": true
    }
  ]
}
```

## 🚨 错误代码

| 错误代码 | 描述 | 解决方案 |
|----------|------|----------|
| 40001 | 参数错误 | 检查请求参数格式 |
| 40002 | 认证失败 | 检查认证信息 |
| 40003 | 权限不足 | 检查用户权限 |
| 40098 | 请求参数错误 | 检查必填参数 |
| 50001 | 内部服务器错误 | 查看服务器日志 |
| 50002 | 数据库连接失败 | 检查数据库配置 |
| 50003 | 插件加载失败 | 检查插件文件 |

## 📚 示例代码

### 插件数据提供示例

```rust
async fn data_for(&self, path: &str, param: &RequestParameter) -> Option<tube::Result<tube::Value>> {
    match path {
        "content" => {
            let content_id = param.url_path
                .split('/')
                .last()
                .and_then(|s| s.parse::<i64>().ok())
                .unwrap_or(0);
            
            let content = Content::new(param).get_detail(content_id)?;
            
            Some(Ok(value!({
                "meta": {
                    "title": content.get_string("title"),
                    "keywords": content.get_string("keywords"),
                    "description": content.get_string("description")
                },
                "content": content,
                "timestamp": chrono::Utc::now().timestamp()
            })))
        }
        _ => None
    }
}
```

### 全局数据提供示例

```rust
async fn global_data(&self, param: &RequestParameter) -> Option<tube::Result<tube::Value>> {
    let site_info = Site::new(param).get_field(&value!{"id": 1, "field": "title,logo"})?;
    let navigation = Links::new(param).get_category_links("navigation")?;
    
    Some(Ok(value!({
        "siteInfo": site_info,
        "navigation": navigation,
        "timestamp": chrono::Utc::now().timestamp()
    })))
}
```

### API分发示例

```rust
async fn api_dispatch(&self, app: &str, param: &RequestParameter) -> Option<tube::Result<tube::Value>> {
    match app {
        "cms" => {
            let action = param.parameter.query.get("action").unwrap_or(&"list".to_string());
            
            match action.as_str() {
                "list" => {
                    let contents = Content::new(param).get_contents(10, 0)?;
                    Some(Ok(value!({
                        "status": "success",
                        "data": contents
                    })))
                }
                "detail" => {
                    let id = param.parameter.query.get("id")
                        .and_then(|s| s.parse::<i64>().ok())
                        .unwrap_or(0);
                    let content = Content::new(param).get_detail(id)?;
                    Some(Ok(value!({
                        "status": "success",
                        "data": content
                    })))
                }
                _ => Some(Ok(value!({
                    "status": "error",
                    "message": "未知的操作"
                })))
            }
        }
        _ => None
    }
}
```

## 📖 参考资源

- [Actix-web 文档](https://actix.rs/docs/)
- [Handlebars 模板语法](https://handlebarsjs.com/guide/)
- [Rust 异步编程](https://rust-lang.github.io/async-book/)
- [插件开发指南](PLUGIN_DEVELOPMENT_GUIDE.md)

---

**API Reference Complete!** 📚
