# SWA (Site Web Application) 项目

## 📖 项目简介

SWA 是一个基于 Rust 和 Actix-web 框架构建的现代化 Web 应用系统，采用插件化架构设计，支持动态模板渲染、智能路由匹配和中间件链处理。

## 🏗️ 系统架构

### 核心特性

- **🚀 高性能**: 基于 Rust 和 Actix-web 框架，提供卓越的性能表现
- **🔌 插件化**: 支持动态插件加载，业务逻辑模块化
- **🎨 模板系统**: 基于 Handlebars 的智能模板匹配和渲染
- **🛡️ 中间件链**: 7层中间件处理，包括日志、缓存、认证等
- **📱 响应式**: 支持多主题和响应式设计
- **🔧 可配置**: 灵活的配置系统，支持 YAML 和 JSON 配置

### 技术栈

- **后端框架**: Actix-web (Rust)
- **模板引擎**: Handlebars
- **数据库**: 支持多种数据库连接
- **缓存**: Redis 支持
- **插件系统**: 动态库加载 (`.dylib`, `.so`)
- **配置管理**: YAML/JSON 配置文件

## 📁 项目结构

```
swa/
├── src/                    # 主应用源码
│   ├── middleware/         # 中间件实现
│   ├── routing/           # 路由处理
│   ├── plugins/           # 插件管理
│   ├── template/          # 模板管理
│   └── config/            # 配置管理
├── plugins/               # 插件目录
│   ├── gov-site/          # 政府网站插件
│   ├── sdk/               # 插件开发SDK
│   └── docs/              # 插件开发文档
├── dist/                  # 构建输出目录
├── wwwroot/               # 静态资源
│   ├── templates/         # 模板文件
│   └── resources/         # 静态资源
└── conf/                  # 配置文件
```

## 🚀 快速开始

### 环境要求

- Rust 1.70+
- Cargo
- 数据库 (可选)
- Redis (可选，用于缓存)

### 安装和运行

1. **克隆项目**
   ```bash
   git clone <repository-url>
   cd swa
   ```

2. **安装依赖**
   ```bash
   cargo build
   ```

3. **配置应用**
   ```bash
   cp config.yaml.example config.yaml
   # 编辑 config.yaml 配置数据库和Redis连接
   ```

4. **运行应用**
   ```bash
   cargo run --bin swa -- --config config.yaml
   ```

5. **访问应用**
   ```
   http://localhost:8081
   ```

## 🔧 配置说明

### 主配置文件 (config.yaml)

```yaml
# 数据库配置
database:
  host: localhost
  port: 3306
  username: root
  password: password
  database: swa_db

# Redis配置
redis:
  host: localhost
  port: 6379
  password: ""

# 应用配置
app:
  port: 8081
  host: "0.0.0.0"
  debug: true

# IP访问控制
ip_access_control:
  enabled: true
```

### 路由配置 (dist/conf/routes.json)

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
        }
      ]
    }
  ]
}
```

## 🔌 插件开发

### 创建插件

1. **使用插件模板**
   ```bash
   cd swa/plugins
   cargo new my-plugin --lib
   ```

2. **实现插件接口**
   ```rust
   use swa_plugin_sdk::{SwaPlugin, RequestParameter, value};
   
   pub struct MyPlugin;
   
   impl SwaPlugin for MyPlugin {
       fn name(&self) -> &'static str {
           "my-plugin"
       }
       
       async fn data_for(&self, path: &str, param: &RequestParameter) -> Option<tube::Result<tube::Value>> {
           Some(Ok(value!({
               "title": "我的插件数据",
               "path": path
           })))
       }
   }
   ```

3. **编译插件**
   ```bash
   ./build-plugin-simple.sh macos my-plugin debug
   ```

### 插件开发文档

详细的插件开发指南请参考：
- [插件开发快速开始](swa/plugins/docs/QUICKSTART.md)
- [插件API参考](swa/plugins/docs/API_REFERENCE.md)
- [最佳实践](swa/plugins/docs/BEST_PRACTICES.md)

## 🛠️ 构建和部署

### 开发构建

```bash
# 构建主应用
cargo build

# 构建插件 (debug模式)
./build-plugin-simple.sh macos gov-site debug
```

### 生产构建

```bash
# 构建主应用 (release模式)
cargo build --release

# 构建插件 (release模式)
./build-plugin-simple.sh macos gov-site release
```

### 跨平台构建

```bash
# Linux musl
./build-plugin-simple.sh linux gov-site release

# macOS ARM64
./build-plugin-simple.sh macos-arm gov-site release
```

## 📊 中间件系统

SWA 采用7层中间件链设计：

1. **日志中间件**: 请求日志记录
2. **IP访问控制**: 访问权限控制
3. **缓存中间件**: 响应缓存处理
4. **主题选择器**: 动态主题切换
5. **路由匹配器**: 智能路由匹配
6. **上下文数据**: 全局数据注入
7. **认证中间件**: 用户认证处理

## 🎨 模板系统

### 智能模板匹配

系统支持多级模板查找：

1. **Public目录**: `public/{path}/index.html`
2. **主题目录**: `themes/{theme}/{template}.html`
3. **根目录**: `templates/{template}.html`
4. **404处理**: 返回错误页面

### 模板语法

基于 Handlebars 模板语法：

```html
<!DOCTYPE html>
<html>
<head>
    <title>{{meta.title}}</title>
</head>
<body>
    <h1>{{siteInfo.title}}</h1>
    {{#each navigationCategories}}
        <a href="{{url}}">{{name}}</a>
    {{/each}}
</body>
</html>
```

## 📈 性能优化

### 缓存策略

- **内存缓存**: 全局数据5分钟缓存
- **Redis缓存**: 支持分布式缓存
- **模板缓存**: 编译后模板缓存

### 插件优化

- **动态加载**: 按需加载插件
- **错误隔离**: 插件错误不影响主应用
- **资源管理**: 自动资源清理

## 🔍 调试和监控

### 日志系统

```bash
# 查看应用日志
tail -f logs/$(date +%Y%m%d)/output.log

# 查看查询日志
tail -f logs/$(date +%Y%m%d)/query_swa.log
```

### 调试模式

```bash
# 启用调试模式
cargo run --bin swa -- --config config.yaml --debug

# 构建调试版本插件
./build-plugin-simple.sh macos gov-site debug
```

## 📚 文档资源

- [架构概览](ARCHITECTURE_OVERVIEW.md) - 系统整体架构说明
- [请求流程文档](REQUEST_FLOW_DOCUMENTATION.md) - 详细的请求处理流程
- [插件开发文档](swa/plugins/docs/) - 插件开发完整指南
- [模板系统文档](swa/doc/) - 模板使用和开发指南

## 🤝 贡献指南

1. Fork 项目
2. 创建特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 打开 Pull Request

## 📄 许可证

本项目采用 MIT 许可证 - 查看 [LICENSE](LICENSE) 文件了解详情。

## 🆘 支持

如果您遇到问题或有任何疑问，请：

1. 查看 [FAQ](docs/FAQ.md)
2. 搜索 [Issues](https://github.com/your-repo/issues)
3. 创建新的 Issue
4. 联系维护团队

---

**SWA** - 现代化的 Rust Web 应用框架 🦀
