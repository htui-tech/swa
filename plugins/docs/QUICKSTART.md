# SWA 插件快速入门

## 🚀 5分钟创建你的第一个插件

### 步骤 1: 创建插件项目

```bash
# 进入插件目录
cd plugins

# 创建新插件
mkdir hello-world
cd hello-world
```

### 步骤 2: 配置项目

创建 `Cargo.toml`：

```toml
[package]
name = "swa-plugin-hello-world"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
swa-plugin-sdk = { path = "../sdk" }
```

### 步骤 3: 编写插件代码

创建 `src/lib.rs`：

```rust
use swa_plugin_sdk::{SwaPlugin, value, Value, Result, RequestParameter};

pub struct HelloWorldPlugin;

impl SwaPlugin for HelloWorldPlugin {
    fn name(&self) -> &'static str {
        "hello-world"
    }
    
    fn data_for(&self, path: &str) -> Option<Value> {
        match path {
            "index" => Some(value!({
                "title": "Hello World Plugin",
                "message": "欢迎使用 SWA 插件系统！",
                "features": [
                    "动态加载",
                    "模板数据注入",
                    "API 处理"
                ]
            })),
            _ => None
        }
    }
    
    fn api_dispatch(&self, app: &str, param: &RequestParameter) -> Option<Result<Value>> {
        match app {
            "hello" => {
                let name = param.parameter.get("name")
                    .and_then(|v| v.as_str())
                    .unwrap_or("World");
                
                Some(Ok(value!({
                    "ok": true,
                    "data": {
                        "message": format!("Hello, {}!", name),
                        "timestamp": chrono::Utc::now().timestamp()
                    }
                })))
            },
            _ => None
        }
    }
}

impl HelloWorldPlugin {
    pub fn new() -> Self {
        Self
    }
}

#[no_mangle]
pub extern "C" fn swa_plugin_create() -> *mut dyn SwaPlugin {
    Box::into_raw(Box::new(HelloWorldPlugin::new()))
}
```

### 步骤 4: 添加依赖

更新 `Cargo.toml`：

```toml
[dependencies]
swa-plugin-sdk = { path = "../sdk" }
chrono = "0.4"
```

### 步骤 5: 编译插件

```bash
# 编译插件
cargo build --release

# 复制到插件目录
cp target/release/libswa_plugin_hello_world.dylib ../../plugins/
```

### 步骤 6: 配置 API 路由

创建或编辑 `conf/api_plugins.yml`：

```yaml
hello: "hello-world"
```

### 步骤 7: 启动 SWA 服务器

```bash
# 回到项目根目录
cd ../..

# 启动服务器
./target/release/swa --port 3000 --auto-routes --template-root templates --data-root data
```

### 步骤 8: 测试插件

**测试模板数据：**
```bash
curl http://localhost:3000/
```

**测试 API：**
```bash
curl -X POST "http://localhost:3000/api/hello/greet/say" \
  -H "Content-Type: application/json" \
  -d '{"name": "Alice"}'
```

**预期响应：**
```json
{
  "code": 200,
  "result": {
    "ok": true,
    "data": {
      "message": "Hello, Alice!",
      "timestamp": 1759528666
    }
  },
  "message": ""
}
```

## 🎉 完成！

恭喜！你已经成功创建了第一个 SWA 插件。现在你可以：

1. 修改插件代码来添加更多功能
2. 重新编译和部署插件
3. 查看 SWA 日志来调试插件
4. 参考完整文档来开发更复杂的插件

## 📚 下一步

- 阅读 [完整开发指南](README.md)
- 查看 [API 参考文档](API_REFERENCE.md)
- 研究示例插件代码
- 开发你自己的业务插件

## 🔧 常用命令

```bash
# 编译插件
cargo build --release

# 部署插件
cp target/release/libswa_plugin_*.dylib ../../plugins/

# 启动服务器
./target/release/swa --port 3000 --auto-routes

# 测试 API
curl -X POST "http://localhost:3000/api/{app}/{module}/{method}" \
  -H "Content-Type: application/json" \
  -d '{"param": "value"}'
```

---

**Happy Plugin Development! 🎉**
