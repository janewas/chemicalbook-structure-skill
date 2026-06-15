---
name: chemicalbook-structure
description: 从 ChemicalBook (www.chemicalbook.com) 查找化学结构式并下载。用户给化学名/CAS号/链接→找到结构式图片→下载发给用户。
platforms: [windows, linux, macos]
---

# ChemicalBook 化学结构式查找

## 功能

用户提供一个**化学名称**（中文或英文）、**CAS号**、或 **ChemicalBook 产品页面链接**，自动找到并下载该化合物的**结构式图片**（GIF 格式）。

## 工作原理

ChemicalBook 的每个产品详情页（`ProductChemicalPropertiesCB{ID}.htm`）的 `<meta property="og:image">` 标签中嵌有结构式图片的完整 URL。

典型结构式图片 URL：
- `https://www.chemicalbook.com/CAS/GIF/{CAS}.gif`（无日期路径）
- `https://www.chemicalbook.com/CAS/{date}/GIF/{CAS}.gif`（带日期路径）

## 工作流程

⚠️ **输入处理须知**：用户通常只给**化学名称**（中文），不知道 CAS 号。切勿让用户自己去查 CAS 号！

**从化学名获取 CAS 号的步骤**：
1. 浏览器打开 `https://www.chemicalbook.com/` → 搜化学名
2. 在搜索结果或产品页的基础信息表中找到 CAS 号
3. 也可以用 web 搜索 `{化学名} CAS` 快速定位 CAS 号
4. 拿到 CAS 号后走快速路径（最快）

### 快速路径（首选：已知 CAS 号时）

如果已知 CAS 号，**直接尝试结构式图片的固定 URL**，无需打开浏览器或访问产品页：

```bash
# 尝试直接下载（大部分化学物质都能走通）
curl -s -o "/tmp/{CAS}.gif" -w "%{http_code}" \
  -A "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
  "https://www.chemicalbook.com/CAS/GIF/{CAS}.gif"
# → HTTP 200（有图）或 HTTP 404（无此路径，需回退）
```

经验：**约 90% 的化学物质**图片在 `CAS/GIF/{CAS}.gif` 路径（无日期子目录），少数在 `CAS/{date}/GIF/{CAS}.gif`。HTTP 200 且 `file` 验证为 GIF 即可确认。

### 标准流程（需定位产品页）

当 CAS 号未知或直接 URL 返回 404 时，才需要定位产品页：

#### 情况 A: 用户给了 ChemicalBook 产品页面链接
直接使用该 URL 进入 Step 2。

#### 情况 B: 用户给了 CAS 号
1. 先尝试快速路径（见上）
2. 如果快速路径失败，访问 `https://www.chemicalbook.com/CAS_{CAS}.htm` 从上下游产品列表中找到 `ProductChemicalPropertiesCB{ID}.htm` 链接
3. 进入产品页提取 `og:image`

#### 情况 C: 用户给了化学名称（中文/英文）
1. 用浏览器打开 `https://www.chemicalbook.com/`
2. 在搜索框中输入化学名称
3. 点击搜索按钮
4. 在搜索结果中找到对应的产品链接（通常以 `ProductChemicalPropertiesCB` 开头）
5. 点击进入产品详情页

> **注意**：ChemicalBook 的搜索页面有 WAF 保护，curl 直接请求会返回"服务不可用"。必须用浏览器交互方式搜索。

### 批量处理

当需要一次查询多个化学物质时（适合有 CAS 号列表的场景）：

```bash
# 批量尝试直接 URL（并行下载）
for cas in "71-43-2" "108-88-3" "64-17-5"; do
  curl -s -o "/tmp/${cas}.gif" \
    -A "Mozilla/5.0" \
    "https://www.chemicalbook.com/CAS/GIF/${cas}.gif"
done

# 验证全部为 GIF
file /tmp/${CAS1}.gif /tmp/${CAS2}.gif /tmp/${CAS3}.gif
```

注意：批量时无法用 `&` 后台执行（terminal 报错）。逐个串行 curl 即可，每个请求 < 1 秒。

### Step 2: 提取结构式图片 URL

从产品详情页的 HTML 中提取 `og:image`：

```bash
curl -s -L -A "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
  "https://www.chemicalbook.com/ProductChemicalPropertiesCB{ID}.htm" \
  | grep -i "og:image" | grep -oP 'content="[^"]*"' | head -1
```

或者从页面的 `结构式` 行提取图片：
```html
<tr><th scope="row" class="w130">结构式</th>
  <td><img src="CAS/20180808/GIF/90-11-9.gif" alt="结构式"/></td>
</tr>
```

也可以直接从浏览器截图判断结构式是否存在。

### Step 3: 下载图片

```bash
# 下载到 Windows 临时目录（MSYS `/tmp` 映射到 C:/Users/{user}/AppData/Local/Temp/）
curl -s -o "/tmp/{CAS}.gif" \
  -A "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
  "https://www.chemicalbook.com/{图片路径}"
```

### Step 4: 验证并展示

1. 用 `file` 命令验证下载的文件是 GIF 图片
2. 用 `vision_analyze` 查看图片内容确认是目标结构式
3. 用 Markdown 格式展示给用户（Windows 用完整盘符路径，详见下方"输出规范"和"路径问题"）

## 常见问题

### 搜索被 WAF 屏蔽
- 搜索页（`Search.aspx`）的 curl 请求会返回"服务不可用"
- 使用**浏览器**进行搜索操作（browser_type + browser_click）
- 或者直接使用 CAS 号通过 `CAS_{编号}.htm` 页面找产品链接

### 多搜索结果
- 搜索结果可能包含多个同名化合物（不同纯度/规格）
- 选择第一个最匹配的结果，即 `ProductChemicalProperties` 开头的最相关链接

### 图片提取失败
- 如果 `og:image` 不存在，检查页面中"结构式"行是否有 `<img>` 标签
- 如果页面完全加载后仍然没有结构式，说明该化合物可能没有结构式图片

### 路径问题：Windows 下用完整路径，不要用 `/tmp/`

**快速查 Windows 实际路径**：
```bash
# MSYS `/tmp` 映射到的真实 Windows 路径
cd /tmp && pwd -W   # 输出如 C:/Users/Hasee/AppData/Local/Temp
```
WebUI 的 MarkdownRenderer（`MarkdownRenderer.vue`）会把 Markdown 图片/文件中的本地路径自动转换为 `/api/hermes/download?path=...` 下载 URL 来加载。但在 Windows 上：
- `/tmp/{file}.gif` 在 MSYS 终端中可用，但 Node.js 的 `path.isAbsolute('/tmp/...')` 返回 `false`
- 下载路由会调用 `resolveHermesPath()` 把路径解析到 Hermes 家目录下，找不到文件
- **必须使用完整的 Windows 盘符路径**：`C:/Users/{用户名}/AppData/Local/Temp/{file}.gif`

详见：`references/webui-win-path-pitfall.md`

## 输出规范

图片展示格式（Windows — 用完整盘符路径）：
```markdown
![{化学名} 结构式](<C:/Users/用户名/AppData/Local/Temp/{CAS}.gif>)
```

示例：
```markdown
![1-溴代萘结构式](<C:/Users/Hasee/AppData/Local/Temp/90-11-9.gif>)
```
