# SWA 应用请求处理流程文档

## 概述

本文档详细描述了 SWA (Site Web Application) 应用的请求处理流程，包括中间件链、路由处理、插件系统和模板渲染等核心组件的工作机制。

## 应用架构

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   HTTP 请求     │───▶│   中间件链        │───▶│   路由处理器     │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                                │                        │
                                ▼                        ▼
                       ┌──────────────────┐    ┌─────────────────┐
                       │  插件系统        │    │   模板引擎       │
                       └──────────────────┘    └─────────────────┘
```

## 详细请求流程图

```
HTTP 请求
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│                    Actix-web 框架                           │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│                    中间件链处理 (优化后顺序)                 │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────┐ │
│  │   日志      │ │  IP访问控制  │ │   缓存检查   │ │ 主题选择 │ │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────┘ │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────┐ │
│  │   模板匹配   │ │ 上下文数据   │ │   认证      │ │   ...   │ │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────┘ │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│                    统一模板处理                              │
│                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐      │
│  │  配置路由    │    │  智能匹配    │    │  404处理    │      │
│  │  匹配        │    │  成功        │    │  失败        │      │
│  └─────────────┘    └─────────────┘    └─────────────┘      │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│                   智能模板匹配 (在中间件中)                  │
│                                                             │
│  1. 配置路由匹配: routes.json 中的路由配置                  │
│  2. Public目录: public/{path}/index.html                   │
│  3. 主题目录: themes/{theme}/{template}.html               │
│  4. 根目录: templates/{template}.html                      │
│  5. 404处理: 返回错误页面                                   │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│                    插件数据处理                              │
│                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐      │
│  │  data_for   │    │ global_data │    │api_dispatch │      │
│  │  路径数据    │    │  全局数据    │    │  API分发    │      │
│  └─────────────┘    └─────────────┘    └─────────────┘      │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│                    模板渲染                                  │
│                                                             │
│  1. 确保模板已注册                                           │
│  2. 合并数据源 (插件 + 全局 + 上下文)                        │
│  3. Handlebars 渲染                                         │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│                    缓存和响应                                │
│                                                             │
│  1. 写入缓存 (如果启用)                                      │
│  2. 返回 HTTP 响应                                          │
└─────────────────────────────────────────────────────────────┘
```

## 请求处理流程

### 1. 请求接收阶段

当 HTTP 请求到达应用时，首先进入 Actix-web 框架：

```rust
// 主要入口点
HttpRequest → RouteHandler::handle_dynamic_route()
```

### 2. 中间件链处理

应用使用中间件链来处理请求，中间件按以下顺序执行：

#### 2.1 中间件执行顺序 (优化后)

1. **日志中间件** (`LoggingMiddleware`)
   - 记录请求信息（方法、路径、主题等）
   - 输出格式：`--> logging [GET] /path theme`

2. **IP访问控制中间件** (`IpAccessControlMiddleware`)
   - 安全检查优先，可配置启用/禁用
   - 检查客户端IP是否在允许列表中
   - 支持白名单和黑名单机制
   - 默认重定向到 `/warning` 页面

3. **缓存检查中间件** (`CacheMiddleware`)
   - 如果已缓存，直接返回响应
   - 缓存策略：`hybrid`（混合模式）
   - 缓存键格式：`{METHOD}:{PATH}:{THEME}:{USER_ID}`

4. **主题选择中间件** (`ThemeSelectionMiddleware`)
   - 根据域名或配置选择主题
   - 默认主题：`default`
   - 设置 `ctx.theme` 字段

5. **模板匹配中间件** (`RouteMatcherMiddleware`)
   - 统一处理配置路由和智能模板匹配
   - 首先尝试匹配 `routes.json` 中的路由配置
   - 配置路由失败时，进行智能模板查找
   - 智能查找优先级：Public目录 → 主题目录 → 根目录
   - 所有匹配失败时，标记为404

6. **上下文数据注入中间件** (`ContextDataMiddleware`)
   - 注入全局数据到模板上下文
   - 调用插件的 `global_data` 方法
   - 设置站点信息、导航数据等

7. **认证中间件** (`AuthMiddleware`)
   - 处理用户认证状态
   - 设置认证提示信息

#### 2.2 中间件上下文 (`MiddlewareContext`)

```rust
pub struct MiddlewareContext {
    pub theme: String,                    // 当前主题
    pub provider_id: i64,                 // 提供者ID
    pub req: Option<RequestParameter>,    // 请求参数
    pub data: HashMap<String, Value>,     // 上下文数据
    pub route_match: Option<RouteMatch>,  // 路由匹配结果
    // ... 其他字段
}
```

### 3. 统一模板处理阶段

#### 3.1 模板处理入口

```rust
RouteHandler::handle_configured_route() → 检查中间件设置的模板信息
```

#### 3.2 统一模板处理分支

根据中间件设置的模板信息，进入不同的处理分支：

##### 3.2.1 配置路由匹配
```rust
if match_type == "configured" {
    // 使用配置路由处理逻辑
    return self.handle_route_by_config(&route_config, template_config, ctx, plugins).await;
}
```

##### 3.2.2 智能匹配成功
```rust
if match_type == "smart" {
    // 使用智能匹配处理逻辑
    return self.handle_smart_match(&template_name, template_config, ctx, plugins).await;
}
```

##### 3.2.3 404处理
```rust
if match_type == "not_found" {
    // 返回404页面
    return self.handle_404(template_config).await;
}
```

### 4. 智能模板匹配机制 (在中间件中实现)

#### 4.1 模板匹配优先级

系统在路由匹配中间件中实现了智能模板匹配，按以下优先级处理：

1. **配置路由匹配**
   - 首先尝试匹配 `routes.json` 中的路由配置
   - 如果匹配成功，使用配置的模板和处理器

2. **智能模板查找** (配置路由失败时)
   - **Public目录**: `wwwroot/templates/public/{path}/index.html`
   - **主题目录**: `wwwroot/templates/themes/{theme}/{template_name}.html`
   - **根目录**: `wwwroot/templates/{template_name}.html`

3. **404处理**
   - 所有匹配失败时，返回404页面

#### 4.2 模板匹配实现

```rust
// 在 RouteMatcherMiddleware 中实现
fn find_smart_template(&self, path: &str, theme: &str) -> Option<String> {
    let template_candidates = self.extract_template_candidates(path);
    
    for candidate in &template_candidates {
        // 1. 检查Public目录
        if self.template_exists_in_public(candidate) {
            return Some(format!("public/{}/index", candidate));
        }
        
        // 2. 检查主题目录
        if self.template_exists_in_theme(candidate, theme) {
            return Some(candidate.clone());
        }
        
        // 3. 检查根目录
        if self.template_exists_in_root(candidate) {
            return Some(candidate.clone());
        }
    }
    
    None // 返回None表示404
}
```

### 5. 插件系统

#### 5.1 插件加载

应用启动时加载插件：

```rust
// 从 plugins/ 目录加载动态库
let plugin = load_plugin("libswa_plugin_bas_site.dylib");
```

#### 5.2 插件接口

插件实现 `SwaPlugin` trait：

```rust
pub trait SwaPlugin {
    fn name(&self) -> &'static str;
    fn init(&self, config: &PluginConfig) -> tube::Result<()>;
    async fn data_for(&self, path: &str, param: &RequestParameter) -> Option<tube::Result<tube::Value>>;
    async fn global_data(&self, param: &RequestParameter) -> Option<tube::Result<tube::Value>>;
    async fn api_dispatch(&self, app: &str, param: &RequestParameter) -> Option<tube::Result<tube::Value>>;
}
```

#### 5.3 插件数据处理

在路由处理中，系统会：

1. **调用插件的 `data_for` 方法**
   ```rust
   for plugin in plugins.iter() {
       if let Some(result) = plugin.data_for(path, param).await {
           // 合并插件数据到模板上下文
           merge_plugin_data(&mut template_data, result);
       }
   }
   ```

2. **检查插件指定的模板名称**
   ```rust
   if key == "_template_name" {
       // 插件可以指定使用哪个模板
       plugin_template_name = Some(template_name_str);
   }
   ```

3. **调用插件的 `global_data` 方法**
   ```rust
   // 在中间件中调用，注入全局数据
   if let Some(result) = plugin.global_data(param).await {
       // 合并全局数据
   }
   ```

### 6. 模板渲染

#### 6.1 模板注册

系统使用懒加载方式注册模板：

```rust
ensure_template_registered(template_configs, theme, template_name);
```

#### 6.2 模板渲染流程

1. **确保模板已注册**
2. **合并所有数据源**
   - 插件数据
   - 全局数据
   - 中间件上下文数据
3. **渲染模板**
   ```rust
   template_config.handlebars.render(&template_name, &data)
   ```

### 7. 缓存机制

#### 7.1 缓存策略

- **策略类型**：`hybrid`（混合模式）
- **TTL**：3600秒（1小时）
- **缓存键格式**：`{METHOD}:{PATH}:{THEME}:{USER_ID}`

#### 7.2 缓存处理

```rust
// 检查缓存
if let Some(cached_response) = cache.get(&cache_key) {
    return Ok(cached_response);
}

// 缓存未命中，处理请求
let response = process_request().await;

// 写入缓存
cache.set(&cache_key, &response, ttl);
```

### 8. 错误处理

#### 8.1 404错误处理

```rust
fn render_404_page(template_config: &TemplateConfig) -> HttpResponse {
    // 渲染404错误页面
    HttpResponse::NotFound()
        .content_type(ContentType::html())
        .body(rendered_content)
}
```

#### 8.2 模板错误处理

```rust
match template_config.handlebars.render(&template_name, &data) {
    Ok(body) => Ok(HttpResponse::Ok().body(body)),
    Err(e) => {
        err_log!("模板渲染错误: {}", e);
        Ok(render_404_page(&template_config))
    }
}
```

## 典型请求流程示例

### 示例1：访问首页 `/` (实际日志)

```
1. HTTP GET / → Actix-web
2. 中间件链处理：
   --> logging [GET] / default
   [debug] handle_dynamic_route called with plugins: 1 plugins loaded
   🔍 [路由处理] 中间件返回重定向响应，直接返回
3. 结果：HTTP 302 重定向到 /warning
```

**说明**：首页被重定向到警告页面，这是中间件中的安全检查机制。

### 示例2：访问警告页面 `/warning` (实际日志)

```
1. HTTP GET /warning → Actix-web
2. 中间件链处理：
   --> logging [GET] /warning default
   🔍 [缓存中间件] 检查缓存: key=GET:/warning:default, enabled=true, ttl=3600, strategy=hybrid
   ❌ [缓存未命中] hybrid模式: GET:/warning:default
   [debug] handle_dynamic_route called with plugins: 1 plugins loaded
3. 路由处理：
   ----> 兜底：直接从插件获取数据 path: /warning
   🔍 [智能模板查找] 开始查找: path=/warning, theme=default
   🔍 [智能模板查找] 候选模板: ["warning"]
   🔍 [智能模板查找] 使用默认模板: warning
   ----> 兜底：模板名称: warning
4. 插件数据处理：
   [route] 尝试从插件 'bas-site' 获取路径 '/warning' 的数据
   [route] 插件 'bas-site' 为路径 '/warning' 提供了数据
   [route] 插件指定模板名称: public/warning/index
   [route] 合并插件数据: 'content' = Text("...")
   [route] 合并插件数据: 'homeUrl' = Text("/")
   [route] 合并插件数据: 'id' = Int64(1)
   [route] 合并插件数据: 'meta' = Object(...)
   [route] 合并插件数据: 'name' = Text("公安信息安全保密"九严禁一必须"")
   [route] 合并插件数据: 'returnUrl' = Text("/")
5. 模板渲染：
   🔍 [路由处理] 准备注册模板: theme=default, template=public/warning/index
   🔍 [模板注册] 开始注册模板: theme=default, template=public/warning/index
   🔍 [模板注册] 模板不存在，需要注册: theme=default, template=public/warning/index
   🔍 [partial注册] 开始注册主题部分模板: theme=default, base_path=wwwroot/templates
   [template:init] 已注册公共挂件(主题:default): widgets/pagination -> wwwroot/templates/public/widgets/pagination.html
   🔍 [partial注册] 主题根目录: wwwroot/templates/themes/default
   🔍 [partial注册] 主题根目录路径存在: true
   🔍 [partial注册] 开始扫描主题目录: wwwroot/templates/themes/default
   🔍 [partial注册] 注册partial: __dialog -> wwwroot/templates/themes/default/__dialog.html
   🔍 [partial注册] 成功注册partial: __dialog
   🔍 [partial注册] 注册partial: __resources -> wwwroot/templates/themes/default/__resources.html
   🔍 [partial注册] 成功注册partial: __resources
   🔍 [partial注册] 注册partial: member/__header -> wwwroot/templates/themes/default/member/__header.html
   🔍 [partial注册] 成功注册partial: member/__header
   🔍 [partial注册] 注册partial: member/trends/__footer -> wwwroot/templates/themes/default/member/trends/__footer.html
   🔍 [partial注册] 成功注册partial: member/trends/__footer
   🔍 [partial注册] 注册partial: __header -> wwwroot/templates/themes/default/__header.html
   🔍 [partial注册] 成功注册partial: __header
   🔍 [partial注册] 注册partial: __footer -> wwwroot/templates/themes/default/__footer.html
   🔍 [partial注册] 成功注册partial: __footer
   🔍 [partial注册] 注册partial: __sidebar -> wwwroot/templates/themes/default/__sidebar.html
   🔍 [partial注册] 成功注册partial: __sidebar
   🔍 [模板路径] public模板: public/warning/index -> wwwroot/templates/public/warning/index.html
   🔍 [路由处理] 模板注册完成: theme=default, template=public/warning/index
6. 缓存和响应：
   💾 [缓存写入] 成功缓存页面: default::/warning::2
   05-10-2025 06:21:41::INFO: 127.0.0.1 "GET /warning HTTP/1.1" 200 6038 "-" "curl/8.7.1" 0.042305
7. 结果：HTTP 200 + 警告页面HTML内容 (6038字节)
```

### 示例3：访问不存在的页面 `/products` (实际日志)

```
1. HTTP GET /products → Actix-web
2. 中间件链处理：
   --> logging [GET] /products default
   [debug] handle_dynamic_route called with plugins: 1 plugins loaded
   🔍 [路由处理] 中间件返回重定向响应，直接返回
3. 结果：HTTP 302 重定向到 /warning
```

**说明**：所有未匹配的路径都被重定向到警告页面，这是中间件中的安全检查机制。

### 示例4：智能模板查找机制演示

假设我们有一个 `/products` 路径的模板文件：

```
1. HTTP GET /products → Actix-web
2. 中间件链处理：正常处理（假设没有重定向）
3. 路由处理：handle_legacy_route("/products", ...)
4. 智能模板查找：
   🔍 [智能模板查找] 开始查找: path=/products, theme=default
   🔍 [智能模板查找] 候选模板: ["products"]
   🔍 [模板存在检查] 主题模板存在: wwwroot/templates/themes/default/products.html
   🔍 [智能模板查找] 找到主题模板: default/products
5. 插件数据处理：调用插件的 data_for("/products", ...)
6. 模板渲染：使用 "products" 模板
7. 返回：HTTP 200 + 产品页面内容
```

## 配置说明

### 中间件配置

中间件配置在 `CoreMiddlewareChain::new()` 中设置，采用优化后的顺序：

```rust
let chain = CoreMiddlewareChain::new()
    .add(LoggingMiddleware::new())                    // 1. 日志记录
    .add(IpAccessControlMiddleware::new())            // 2. IP访问控制
    .add(CacheMiddleware::new())                      // 3. 缓存检查
    .add(ThemeSelectionMiddleware::new())             // 4. 主题选择
    .add(RouteMatcherMiddleware::new(registry))       // 5. 模板匹配
    .add(ContextDataMiddleware::new())                // 6. 上下文数据
    .add(AuthMiddleware::new());                      // 7. 认证
```

#### IP访问控制配置

IP访问控制可以通过配置文件控制：

```yaml
# config.yaml
ip_access_control:
  enabled: true  # 启用IP访问控制（默认）
  # enabled: false  # 禁用IP访问控制（用于开发测试）
```

### 插件配置

插件配置通过 `PluginConfig` 传递：

```rust
pub struct PluginConfig {
    pub plugin_data: tube::Value,    // 插件特定配置
    pub app_config: tube::Value,     // 应用完整配置
    pub config_path: Option<String>, // 配置文件路径
}
```

### 模板配置

模板配置支持多主题：

```rust
pub struct TemplateConfig {
    pub handlebars: Handlebars<'static>,
    pub theme: String,
    pub top_level_vars: HashMap<String, Vec<String>>,
}
```

## 性能优化

### 1. 缓存策略

- 使用混合缓存策略，平衡性能和实时性
- 缓存键包含用户标识，支持个性化缓存
- TTL设置为1小时，适合大多数场景

### 2. 懒加载

- 模板采用懒加载方式，首次访问时才注册
- 插件动态加载，按需初始化

### 3. 异步处理

- 所有插件方法都是异步的
- 中间件链支持异步处理
- 模板渲染异步执行

## 扩展性

### 1. 插件扩展

- 插件可以实现自定义的数据处理逻辑
- 支持指定自定义模板名称
- 可以处理API请求分发

### 2. 中间件扩展

- 中间件链可以轻松添加新的中间件
- 支持中间件的顺序调整
- 每个中间件都可以短路请求处理

### 3. 模板扩展

- 支持多主题系统
- 智能模板查找机制
- 支持嵌套模板和部分模板

## 调试和监控

### 1. 日志输出

应用提供详细的日志输出：

```
--> logging [GET] /path theme
🔍 [智能模板查找] 开始查找: path=/path, theme=default
🔍 [模板存在检查] 主题模板存在: /path/to/template.html
[route] 插件 'bas-site' 为路径 '/path' 提供了数据
💾 [缓存写入] 成功缓存页面: key
```

### 2. 性能监控

- 请求处理时间记录
- 缓存命中率统计
- 模板渲染时间监控

## 当前应用特性总结

### 核心特性

1. **智能模板查找机制**
   - 自动根据路径和主题查找最合适的模板文件
   - 支持主题目录和根目录的模板查找
   - 完全移除硬编码的路径映射

2. **插件驱动的数据处理**
   - 插件可以指定自定义模板名称
   - 支持路径数据、全局数据和API分发
   - 插件数据自动合并到模板上下文

3. **中间件链架构**
   - 7个核心中间件按顺序处理请求
   - 支持中间件短路机制
   - 每个中间件都可以修改请求上下文

4. **缓存优化**
   - 混合缓存策略（hybrid）
   - 支持个性化缓存（包含用户ID）
   - TTL设置为1小时

5. **安全检查机制**
   - 所有请求默认重定向到警告页面
   - 需要用户确认后才能访问其他页面
   - 支持IP访问控制

### 技术亮点

- **异步处理**：所有插件方法和中间件都支持异步
- **懒加载**：模板采用懒加载方式，首次访问时才注册
- **动态插件**：支持运行时加载动态库插件
- **多主题支持**：支持多主题系统，主题间完全隔离
- **详细日志**：提供完整的请求处理日志，便于调试

### 扩展性

- **插件扩展**：可以轻松添加新的插件来处理特定路径
- **中间件扩展**：可以添加新的中间件来增强功能
- **模板扩展**：支持嵌套模板和部分模板
- **配置扩展**：支持灵活的配置管理

## 总结

SWA 应用采用了现代化的架构设计，通过中间件链、插件系统和智能模板查找机制，实现了高度可扩展和可维护的Web应用框架。整个请求处理流程清晰明确，支持灵活的配置和扩展，能够满足复杂Web应用的需求。

当前应用特别适合基础网站等需要安全检查和内容管理的场景，通过插件系统可以轻松扩展各种业务功能。
