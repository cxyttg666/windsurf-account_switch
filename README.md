# XG-Windsurf

XG-Windsurf 激活管理工具 - VS Code 扩展

> 🤖 **声明**：本项目包含大量 AI 辅助生成的代码，主要用于学习和研究目的。代码质量和架构设计可能存在不足之处，欢迎各位大佬提出改进建议，请轻喷 🙏

## 功能特性

- ✨ **账户激活管理** - 激活码验证和管理
- 🔄 **无感换号** - 实现账号无缝切换，无需手动重新登录
- 🖥️ **多设备支持** - 支持多台设备使用同一激活码
- 🚀 **自动登录** - 通过动态补丁实现自动注入认证信息
- 📂 **路径管理** - 自动检测和管理 Windsurf 安装路径
- 📢 **公告获取** - 获取服务器端公告信息

## ⚡ 无感换号实现原理

### 核心思想

通过**动态修补（Runtime Patching）** Windsurf 的核心扩展文件，注入自定义认证命令，实现账号的无缝切换。

### 技术实现流程

#### 1. **补丁检测与应用** (`WindsurfPatchService`)

项目会在运行时检查 Windsurf 的核心扩展文件（`extension.js`）是否已应用补丁：

```
检查补丁特征关键字：
- "windsurf.provideAuthTokenToAuthProviderWithShit"
- "handleAuthTokenWithShit"
```

如果未检测到补丁，系统会自动：

1. **定位原始函数** - 查找 `handleAuthToken` 函数（Windsurf 原生的认证处理函数）
2. **注入新函数** - 在原函数后插入 `handleAuthTokenWithShit` 函数
3. **注册自定义命令** - 添加 `windsurf.provideAuthTokenToAuthProviderWithShit` 命令
4. **写入文件** - 将修改后的内容写回 `extension.js`

#### 2. **关键区别：绕过服务器验证**

| 对比项 | 原生 `handleAuthToken` | 补丁 `handleAuthTokenWithShit` |
|--------|----------------------|------------------------------|
| **认证流程** | 调用 `registerUser(token)` 向服务器验证 | **直接接受传入的认证信息** |
| **网络请求** | 需要连接官方服务器验证 | **无需额外网络请求** |
| **适用场景** | 官方正常登录流程 | 自定义激活码系统 |

**核心代码差异：**

```javascript
// 原生函数 - 需要服务器验证
async handleAuthToken(A) {
    const e = await (0, Q.registerUser)(A);  // ← 这里会请求官方服务器
    const { apiKey: t, name: i } = e;
    // ...
}

// 补丁函数 - 直接使用传入的凭据
async handleAuthTokenWithShit(A) {
    const { apiKey: t, name: i } = A;  // ← 直接使用传入参数，无需验证
    // ...
}
```

#### 3. **自动登录流程** (`WindsurfAutoLoginService`)

当用户点击"刷新"或激活账号时：

```
1. 检查补丁状态
   ↓
2. 如需要则应用补丁（并重启 Windsurf）
   ↓
3. 执行登出命令（清理旧会话）
   ↓
4. 调用自定义命令注入新会话
   vscode.commands.executeCommand(
       "windsurf.provideAuthTokenToAuthProviderWithShit",
       {
           apiKey: "your-api-key",
           name: "user@example.com",
           apiServerUrl: "https://api.example.com"
       }
   )
   ↓
5. 会话注入到 VSCode Secrets
   ↓
6. 触发会话变更事件，更新 UI
```

#### 4. **文件修改位置**

补丁会修改以下文件（具体路径因操作系统而异）：

- **Windows**: `%LOCALAPPDATA%\Programs\Windsurf\resources\app\extensions\codeium.windsurf-...\dist\extension.js`
- **macOS**: `/Applications/Windsurf.app/Contents/Resources/app/extensions/codeium.windsurf-.../dist/extension.js`
- **Linux**: `~/.windsurf/resources/app/extensions/codeium.windsurf-.../dist/extension.js`

#### 5. **权限处理**

由于需要修改系统文件，项目包含权限检测机制：

- **Windows**: 检测文件是否可写，提示以管理员身份运行 Windsurf
- **macOS/Linux**: 检测文件权限，提示使用 `chmod` 修改权限

### 安全性说明

⚠️ **重要提示**：
- 补丁仅修改本地文件，不涉及网络攻击或破解
- 补丁函数与原生函数逻辑几乎完全相同，仅跳过服务器验证步骤
- 每次 Windsurf 更新后，补丁可能需要重新应用
- 使用自定义激活系统替代官方认证服务器

### 为什么叫"无感换号"？

用户只需点击账号列表中的其他账号，系统会：
1. 自动登出当前账号
2. 自动注入新账号的认证信息
3. 自动刷新 Cascade 界面

整个过程**无需手动操作**，体验如同在多个官方账号间切换。

---

## 安装

1. 克隆本仓库
```bash
git clone https://github.com/your-username/xg-windsurf.git
cd xg-windsurf
```

2. 安装依赖
```bash
npm install
```

3. 编译项目
```bash
npm run build
```

4. 打包扩展
```bash
npm run package
```

## 配置

### API 服务器配置

在开始使用前，你需要配置API服务器地址。编辑 `src/config/api.ts` 文件：

```typescript
export const API_CONFIG = {
    BASE_URL: 'http://your-api-server.com',  // 替换为你的API服务器地址
    TIMEOUT: 30000,
} as const;
```

### API 端点

项目使用以下API端点（请确保你的服务器实现这些端点）：

#### 激活相关
- `POST /api/account/validate-key` - 激活码验证
  ```json
  请求: { "key_code": "xxx", "device_id": "xxx" }
  响应: { "code": 200, "msg": "success", "data": {...} }
  ```

- `POST /api/account/activate-refresh` - 刷新激活状态
  ```json
  请求: { "key_code": "xxx", "device_id": "xxx" }
  响应: { 
    "code": 200, 
    "data": {
      "account": "{...}",  // JSON字符串，包含 apiKey, name, apiServerUrl 等
      "metadata": "...",
      "timestamp": "..."
    }
  }
  ```

#### 其他功能
- `POST /quota/api/key-usage` - 获取配额使用状态
- `GET /api/announcements` - 获取公告列表
- `POST /api/key/convert` - Cursor密钥转换为Windsurf密钥

## 开发指南

### 监听模式（开发）

```bash
npm run watch
```

这会同时启动：
- Extension 监听模式（后端）
- Webview 开发服务器（前端）

### 构建生产版本

```bash
npm run build
```

构建后的文件位于 `dist/` 目录。

### 打包扩展

```bash
npm run package
```

生成 `.vsix` 文件，可在 VS Code / Windsurf 中手动安装。

### 项目结构

```
xg-windsurf/
├── src/
│   ├── config/              # 配置文件
│   │   └── api.ts          # API服务器配置
│   ├── services/            # 核心服务
│   │   ├── apiService.ts           # API请求封装
│   │   ├── storageService.ts       # 本地存储
│   │   ├── windsurfPatchService.ts # 补丁注入（核心）
│   │   ├── windsurfAutoLoginService.ts # 自动登录
│   │   └── windsurfPathService.ts  # 路径检测
│   ├── types/               # TypeScript 类型定义
│   ├── webview/             # 前端界面（React）
│   │   ├── components/     # React 组件
│   │   ├── provider.ts     # Webview Provider
│   │   └── App.tsx         # 主应用
│   └── extension.ts         # 扩展入口
├── webpack/                 # Webpack 配置
├── package.json            # 项目配置
└── README.md               # 本文件
```

## 核心技术栈

- **后端（Extension）**
  - TypeScript
  - VS Code Extension API
  - Node.js fs/path 模块

- **前端（Webview）**
  - React 19
  - TypeScript
  - Webpack 5
  - CSS 样式

- **构建工具**
  - Webpack
  - ts-loader
  - @vscode/vsce

## 许可证

查看 [LICENSE](LICENSE) 文件了解详情。

## 贡献指南

欢迎提交 Pull Request 和 Issue！

**关于代码质量**：
- 本项目大部分代码由 AI 辅助生成，可能存在不够优雅或不符合最佳实践的地方
- 非常欢迎各位大佬指出问题并提供改进建议
- 如果你有更好的实现方案，欢迎直接提交 PR

在提交 PR 前，请确保：
1. 代码通过 ESLint 检查 (`npm run lint`)
2. 所有 TypeScript 类型正确
3. 测试通过（如有）
4. 提交信息清晰明了

## 常见问题

### Q: 补丁应用失败？
**A**: 检查以下几点：
- Windsurf 是否以管理员/root权限运行？
- Windsurf 版本是否兼容？（补丁基于特定版本开发）
- `extension.js` 文件是否被防病毒软件锁定？

### Q: 切换账号后 Cascade 没反应？
**A**: 
- 确保补丁已成功应用（检查日志）
- 尝试重启 Windsurf
- 检查账号的 `apiKey` 和 `apiServerUrl` 是否正确

### Q: 每次更新 Windsurf 都要重新打补丁？
**A**: 是的。Windsurf 更新会覆盖 `extension.js` 文件，补丁需要重新应用。

### Q: 支持哪些 Windsurf 版本？
**A**: 理论上支持所有包含 `handleAuthToken` 函数的版本，但建议使用最新稳定版。

## 注意事项

⚠️ **重要提示**：
- 本项目需要配合后端API服务器使用
- **后端API应返回明文数据**（本开源版本已移除加密功能）
- 首次使用前必须在 `src/config/api.ts` 中配置API服务器地址
- 补丁会修改 Windsurf 的系统文件，请确保有备份
- 仅供学习交流使用，请勿用于非法用途



欢迎添加 QQ 交流项目使用问题、提出建议或分享你的使用经验！

## 致谢

- 🤖 感谢 AI 编程助手（Cascade/Cursor/Claude 等）在开发过程中提供的巨大帮助
- 💻 感谢 Windsurf 团队开发了如此优秀的 AI 编程工具
- 🌟 感谢所有为开源社区做出贡献的开发者！
- 🙏 感谢每一位使用本项目并提出宝贵意见的朋友
