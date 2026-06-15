# ChemicalBook 结构式查找 Skill 🧪

> Hermes Agent 专用 Skill — 根据化学名称/CAS号/ChemicalBook链接，自动查找并下载化学结构式图片

## 功能

输入一个**化学名称**（中文/英文）、**CAS号**、或 **ChemicalBook 产品页面链接**，自动在 ChemicalBook (www.chemicalbook.com) 上找到该化合物，下载结构式图片并展示。

## 安装

将本仓库克隆或下载到 Hermes Agent 的 skills 目录：

```bash
# 进入你的 Hermes skills 目录
cd ~/AppData/Local/hermes/skills

# 克隆本仓库
git clone https://github.com/janewas/chemicalbook-structure-skill.git research/chemicalbook-structure
```

或者手动复制 `SKILL.md` 和 `references/` 文件夹到 `skills/research/chemicalbook-structure/` 下。

## 使用方法

在 Hermes Agent 中直接说：

> **"帮我查一下 水杨酸 的结构式"**
> **"查一下 1-溴代萘 的结构式"**
> **"查 CAS 69-72-7 的结构式"**

支持三种输入方式：

| 输入类型 | 示例 |
|---------|------|
| 🔗 ChemicalBook 链接 | `https://www.chemicalbook.com/ProductChemicalPropertiesCB5197277.htm` |
| 🔢 CAS 号 | `90-11-9` |
| 🧪 化学名称 | `水杨酸`、`Salicylic acid`、`苯酚` |

## 工作原理

1. 根据输入定位 ChemicalBook 产品页面
2. 从页面 `<meta property="og:image">` 标签提取结构式图片 URL
3. 下载 GIF 格式结构式图片
4. 验证并展示给用户

结构式图片 URL 模式：`https://www.chemicalbook.com/CAS/GIF/{CAS}.gif`

## 注意事项

- **Windows 用户**：在 WebUI 中展示图片时需使用完整盘符路径（如 `C:/Users/.../Temp/file.gif`），不要用 `/tmp/` 前缀
- 详情见 `references/webui-win-path-pitfall.md`

## 许可

MIT
