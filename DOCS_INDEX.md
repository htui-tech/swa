# SWA 文档索引

## 📚 文档概览

欢迎使用 SWA (Site Web Application) 文档系统！本文档索引将帮助您快速找到所需的文档资源。

## 🚀 快速开始

### 新用户必读
1. **[README.md](README.md)** - 项目概述、快速开始和基本使用
2. **[架构概览](ARCHITECTURE_OVERVIEW.md)** - 系统整体架构和核心组件
3. **[请求流程文档](REQUEST_FLOW_DOCUMENTATION.md)** - 详细的请求处理流程

## 🔧 开发指南

### 插件开发
- **[插件开发指南](PLUGIN_DEVELOPMENT_GUIDE.md)** - 完整的插件开发教程
- **[插件API参考](swa/plugins/docs/API_REFERENCE.md)** - 插件接口详细说明
- **[插件最佳实践](swa/plugins/docs/BEST_PRACTICES.md)** - 插件开发最佳实践
- **[插件快速开始](swa/plugins/docs/QUICKSTART.md)** - 插件开发快速入门

### 模板系统
- **[Handlebars 快速参考](swa/doc/HANDLEBARS_QUICK_REFERENCE.md)** - 模板语法快速参考
- **[Handlebars 助手函数](swa/doc/HANDLEBARS_HELPERS.md)** - 内置助手函数说明
- **[Handlebars 示例](swa/doc/HANDLEBARS_EXAMPLES.md)** - 模板使用示例
- **[调试助手](swa/doc/DEBUG_HELPERS.md)** - 模板调试工具

## 🚀 部署和运维

### 部署指南
- **[部署指南](DEPLOYMENT_GUIDE.md)** - 生产环境部署完整指南
- **[构建文档](BUILD.md)** - 应用构建和编译说明

### 配置管理
- **[配置参考](API_REFERENCE.md#配置系统)** - 完整的配置选项说明
- **[路由配置](dist/conf/README.md)** - 路由配置详细说明

## 📖 API 参考

### 核心API
- **[API参考](API_REFERENCE.md)** - 完整的API接口文档
- **[插件接口](API_REFERENCE.md#插件接口)** - 插件开发接口
- **[中间件接口](API_REFERENCE.md#中间件接口)** - 中间件开发接口
- **[模板系统](API_REFERENCE.md#模板系统)** - 模板系统接口

### 服务API
- **[CMS服务](cms/README.md)** - CMS系统API文档
- **[认证服务](swa/src/middleware/auth.rs)** - 认证中间件API

## 🏗️ 架构和设计

### 系统架构
- **[架构概览](ARCHITECTURE_OVERVIEW.md)** - 系统整体架构
- **[请求流程](REQUEST_FLOW_DOCUMENTATION.md)** - 请求处理流程
- **[中间件系统](swa/src/middleware/)** - 中间件实现

### 核心组件
- **[路由系统](swa/src/routing/)** - 路由处理实现
- **[插件系统](swa/plugins/)** - 插件管理实现
- **[模板系统](swa/src/template/)** - 模板管理实现
- **[配置系统](swa/src/config/)** - 配置管理实现

## 🛠️ 工具和脚本

### 构建工具
- **[插件编译脚本](build-plugin-simple.sh)** - 跨平台插件编译
- **[主应用构建](build.sh)** - 主应用构建脚本
- **[构建配置](build-config.json)** - 构建配置文件

### 开发工具
- **[Makefile](Makefile)** - 常用命令快捷方式
- **[Git脚本](git.sh)** - Git操作辅助脚本

## 📊 示例和教程

### 插件示例
- **[基础网站插件](swa/plugins/bas-site/)** - 完整的插件实现示例
- **[插件SDK](swa/plugins/sdk/)** - 插件开发SDK

### 模板示例
- **[默认模板](wwwroot/templates/themes/default/)** - 默认主题模板
- **[公共模板](wwwroot/templates/public/)** - 公共页面模板

## 🔍 故障排除

### 常见问题
- **[部署问题](DEPLOYMENT_GUIDE.md#故障排除)** - 部署常见问题解决
- **[插件问题](PLUGIN_DEVELOPMENT_GUIDE.md#调试插件)** - 插件开发问题解决
- **[模板问题](swa/doc/DEBUG_HELPERS.md)** - 模板渲染问题解决

### 调试工具
- **[日志系统](API_REFERENCE.md#调试接口)** - 日志查看和分析
- **[健康检查](API_REFERENCE.md#健康检查)** - 系统状态检查
- **[性能监控](DEPLOYMENT_GUIDE.md#监控和日志)** - 性能监控配置

## 📋 开发规范

### 代码规范
- **[Rust代码规范](https://doc.rust-lang.org/book/)** - Rust官方编程规范
- **[插件开发规范](swa/plugins/docs/BEST_PRACTICES.md)** - 插件开发最佳实践

### 文档规范
- **[Markdown规范](https://www.markdownguide.org/)** - Markdown文档编写规范
- **[API文档规范](API_REFERENCE.md)** - API文档编写示例

## 🔗 外部资源

### 技术文档
- **[Rust官方文档](https://doc.rust-lang.org/)** - Rust编程语言文档
- **[Actix-web文档](https://actix.rs/docs/)** - Web框架文档
- **[Handlebars文档](https://handlebarsjs.com/)** - 模板引擎文档

### 社区资源
- **[Rust社区](https://users.rust-lang.org/)** - Rust用户社区
- **[GitHub仓库](https://github.com/your-repo)** - 项目源码仓库
- **[问题反馈](https://github.com/your-repo/issues)** - 问题报告和功能请求

## 📞 支持和联系

### 获取帮助
1. **查看文档**: 首先查看相关文档
2. **搜索问题**: 在GitHub Issues中搜索类似问题
3. **创建Issue**: 如果问题未解决，创建新的Issue
4. **联系维护者**: 通过GitHub联系项目维护者

### 贡献代码
1. **Fork项目**: Fork项目到您的GitHub账户
2. **创建分支**: 创建特性分支进行开发
3. **提交代码**: 提交代码并创建Pull Request
4. **代码审查**: 等待代码审查和合并

---

## 📝 文档更新日志

- **2024-10-05**: 创建完整的文档体系
- **2024-10-05**: 添加插件开发指南
- **2024-10-05**: 添加部署指南
- **2024-10-05**: 添加API参考文档
- **2024-10-05**: 更新架构概览文档

---

**Happy Reading!** 📚

如果您发现文档有任何问题或需要补充，请随时创建Issue或提交Pull Request。
