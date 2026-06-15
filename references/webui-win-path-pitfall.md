# WebUI Windows 本地文件路径处理机制

## 背景

Hermes WebUI 的 `MarkdownRenderer.vue` 组件处理 Markdown 中的图片/文件路径时，会将本地绝对路径自动转换为 `/api/hermes/download` 的下载 URL。

## 代码路径

- `packages/client/src/components/hermes/chat/MarkdownRenderer.vue`（第170-175行）
- `packages/client/src/api/hermes/download.ts`
- `packages/server/src/routes/hermes/download.ts`

## 处理流程

1. **Markdown 渲染**：`![alt](/tmp/file.gif)` → `<img src="/tmp/file.gif">`
2. **路径检测**（`isLocalFilePath`）：
   - 以 `/` 开头 → 是本地路径
   - 匹配 `[a-zA-Z]:[\\/]` (如 `C:/...`) → 是本地路径
3. **路径归一化**（`normalizeLocalFilePath`）：将反斜杠 `\` 转成正斜杠 `/`
4. **生成下载 URL**（`getDownloadUrl`）：`{base}/api/hermes/download?path={path}&token={token}&profile={profile}`
5. **服务器处理**：`download.ts` 路由读取文件并返回

## Windows 关键坑

### Node.js 的 `path.isAbsolute()` 行为

| 路径 | `isAbsolute()` 结果 |
|------|-------------------|
| `C:/Users/...` | ✅ `true` |
| `C:\Users\...` | ✅ `true`（被 `normalizePlatformPath` 处理） |
| `/tmp/file.gif` | ❌ `false`（Unix 风格，Windows 不含盘符） |
| `file.gif` | ❌ `false` |

### 后果

MSYS/Git Bash 的 `/tmp` 映射到 `C:/Users/{user}/AppData/Local/Temp/`，但这只在 MSYS 内部有效。当 `/tmp/file.gif` 被传给 WebUI：

1. `isAbsolute('/tmp/file.gif')` → `false`（Windows Node.js）
2. `resolveHermesPath('/tmp/file.gif', profile)` 被调用
3. 路径被解析到 Hermes 家目录（如 `~/.hermes/tmp/file.gif`）
4. 文件不存在 → 404

### 正确做法

**始终使用完整的 Windows 盘符路径**：

```markdown
![描述](<C:/Users/用户名/AppData/Local/Temp/file.gif>)
```

而不是：

```markdown
![描述](/tmp/file.gif)  <!-- ❌ 在 Windows WebUI 下不可用 -->
```

### 为什么尖括号是必需的

Windows 路径中的冒号 `:` 会被 Markdown 解析器误解（比如认为是 URL 协议部分）。用 `<>` 包裹路径可避免：
```markdown
![描述](<C:/Users/用户名/AppData/Local/Temp/file.gif>)
```

## 验证方式

在 WebUI 中发送测试消息，检查浏览器的开发者工具中：
1. `<img>` 标签的 `src` 属性是否被替换为 `/api/hermes/download?path=...`
2. 该 URL 的请求是否返回 200

## 相关文件

- `file-provider.ts`: `validatePath()`, `normalizePlatformPath()`, `resolveHermesPath()`
- `download.ts`: `getDownloadUrl()`, `downloadFile()`, `fetchFileText()`
