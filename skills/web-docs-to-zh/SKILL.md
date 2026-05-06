---
name: web-docs-to-zh
description: 将在线文档翻译成中文 Markdown 文件。提供通用 HTML 清洗管道，处理现代文档站（Docusaurus / Mintlify / VitePress / Sphinx 等）的 DOM 噪音（theme-admonition、复制按钮、锚点 #、活跃标记、侧边栏导航等）。
author: kang99
version: 1.0.0
license: MIT
metadata:
  hermes:
    tags: [documentation, translation, chinese, markdown, docusaurus, mintlify, vitepress, html-cleaning]
    related_skills: []
---

# web-docs-to-zh — Web 文档翻译

将在线文档站网页内容翻译成中文并保存为 Markdown 文件。提供一套通用 HTML 清洗管道，专门处理现代文档站的 DOM 噪音。

## 何时使用 (When to Use)

用户提供文档 URL 列表（通常一行一个，用空白行分隔），要求获取并翻译成中文 `.md` 文件保存到指定目录。

### 中文触发短语
- 「获取网页文档转换成中文md保存在xxx」
- 「把这几个文档站翻译成中文」

### English Trigger Phrases
- "Translate this documentation to Chinese"
- "Get the content in Chinese markdown"
- "Translate docs to zh-cn"
- "Convert this doc to Chinese"

## 工作流

### 场景 A：文档有 GitHub 源码仓库（首选路径）

直接用 `curl` 获取 raw markdown 并直接翻译（见下方「步骤」）。

### 场景 B：无 raw URL，需要使用 HTML 清洗管道

当无法获取 raw URL 时，通过 `browser_navigate` + `browser_console` 提取 HTML DOM，然后运行 **HTML 清洗管道**（见下方「HTML 清洗管道」章节）。

## 步骤

### 1. 创建目标目录

如果目标目录不存在，先创建：
```bash
mkdir -p "/path/to/target/dir"
```

### 2. 解析 URL

从用户请求中解析 URL —— 通常一行一个，用空白行分隔。

### 3. 内容获取（每个 URL 单独处理，不要并行）

#### 3a. 首选：curl 获取 raw markdown（有 GitHub 仓库时）

从文档页 URL 推导 raw markdown URL。常见模式：

- `/docs/path/to/page` → `raw.githubusercontent.com/<owner>/<repo>/main/website/docs/path/to/page.md`
- `/docs/user-guide/configuration` → 通过 `/docs/intro`（About 区域）或 footer 中的 GitHub 链接查找仓库 owner
- 不确定 owner/repo？通过 `/docs/intro` 或 footer 中的 GitHub 链接查找仓库 owner
- 验证：`curl -sL <raw-url>` + `head -c 200` 确认返回 `---\nsidebar_position:` （有效的 markdown frontmatter 前缀）

直接 curl，不要截断：
```bash
curl -sL <raw-url> -o /tmp/docs_page.md
read_file limit=600 offset=1 path=/tmp/docs_page.md
read_file limit=2000 offset=601 path=/tmp/docs_page.md
```

#### 3b. 备选：web_extract 快速检查

```python
from hermes_tools import web_extract
result = web_extract(urls=["https://target/page"])
# 检查返回内容长度；如果 < 500 字符 → fallback 用 browser
```

仅用于短页面。`web_extract` 限制约 5000 字符且可能总结长页面。

#### 3c. 最后手段：浏览器 DOM 提取（HTML 清洗管道的输入）

当 raw URL 不可用时，通过浏览器提取 HTML：

1. 首选 `browser_console` 提取 `innerHTML`（包含完整 DOM，含 `theme-admonition`、类名等，供清洗管道使用）：
   ```
   browser_console(expression="document.querySelector('article, main, .theme-doc-markdown, .docs-content')?.innerHTML")
   ```
2. 备选 `innerText`（纯文本，跳过清洗管道直接翻译，但会丢失结构信息）。
3. 最后手段：`browser_snapshot(full=true)` —— 慢且超过 8000 字符时会截断。

永远不要用 `web_extract` 处理长文档 → 限制约 5000 字符且会总结长页面。

### 4. 翻译

- **curl/raw markdown 场景**：直接翻译成中文，保留所有技术细节、代码块、YAML 配置、表格、链接和格式。
- **HTML 清洗场景**：先用清洗管道转为 Markdown，再翻译（见下方「HTML 清洗管道」章节）。

翻译通用原则：
- 格式锁定（标题、列表、代码块、表格、行内代码、链接保持原样不变）
- 代码块内的内容不翻译
- 保留技术专有名词不直译
- 不做主观添加
- 代码块保持原样 —— YAML 配置、CLI 命令、JSON 示例不应翻译

### 5. 文件命名

**命名规范**：从页面标题或 URL slug 使用 kebab-case（短横线分隔小写）。
- "Persistent Memory" → `persistent-memory.md`
- "Context Files" → `context-files.md`

### 6. 写入并确认

用 `write_file` 写入 `target_dir/filename.md`，然后 `ls` 列出所有新建的文件确认完成。

## HTML 清洗管道

当通过 `browser_console(expression="...innerHTML")` 提取到 HTML 后，在 `execute_code` 中运行以下管道。按顺序执行：

### Step 0：提取主内容区域

```python
from bs4 import BeautifulSoup, NavigableString, Tag
import re

soup = BeautifulSoup(html_content, 'html.parser')

# 优先提取正文区域
for sel in ['article', 'main', '.theme-doc-markdown', '.docs-content']:
    el = soup.find(sel)
    if el and len(str(el)) > 200:
        main = el
        break
else:
    body = soup.find('body')
    main = body if body else soup
```

### Step 1：转换 admonition（提示框）为 GFM callout

```python
def convert_admonitions(node):
    """将 theme-admonition div 转为 GFM callout，跳过 SVG/icon 标签。"""
    callout_map = {
        'note': 'NOTE', 'tip': 'TIP', 'warning': 'WARNING',
        'caution': 'CAUTION', 'danger': 'DANGER', 'info': 'INFO'
    }
    for div in node.find_all(class_=re.compile(r'theme-admonition-(note|tip|warning|caution|danger|info)')):
        ad_type = None
        for c in div.get('class', []):
            for k in callout_map:
                if k in c:
                    ad_type = k
                    break
            if ad_type:
                break
        if not ad_type:
            continue
        # 提取文本（排除 SVG 和 icon 标签）
        text_parts = []
        for child in div.children:
            if isinstance(child, Tag) and child.name == 'svg':
                continue
            t = child.get_text(strip=True)
            if t and t.lower() not in [k for k in callout_map.values()]:
                text_parts.append(t)
        content = ' '.join(text_parts).strip()
        if content:
            # 插入 callout 标记
            marker = f'\n> [!{callout_map[ad_type]}]\n> {content}\n'
            div.insert_before(marker)
            div.extract()
    return node

main = convert_admonitions(main)
```

### Step 2：移除 DOM 噪音

```python
# 移除不需要的标签
tags_to_remove = ['script', 'style', 'nav', 'aside', 'footer', 'header', 'button', 'iframe']
for tag_name in tags_to_remove:
    for el in main.find_all(tag_name):
        el.decompose()

# 移除噪音类名的元素
cls_filters = ['admonitionIcon', 'copyButton', 'admonitionHeading',
               'admonitionContent']
for cls_name in cls_filters:
    for el in main.find_all(class_=re.compile(cls_name)):
        el.decompose()

# 移除所有 data-* 属性
for el in main.find_all(attrs=re.compile(r'^data-')):
    del_attrs = [a for a in el.attrs if a.startswith('data-')]
    for a in del_attrs:
        del el[a]

# 用 word boundary 避免误杀 'reactive'、'disconnect' 等类名
for el in main.find_all(class_=re.compile(r'\bactive\b')):
    el.decompose()
for el in main.find_all(class_=re.compile(r'activeListItem')):
    el.decompose()
```

### Step 3：HTML → Markdown 递归转换

```python
def to_markdown(node):
    """递归将 BS4 节点转为 Markdown 字符串。"""
    if isinstance(node, NavigableString):
        return node.get_text()
    if not isinstance(node, Tag):
        return ''

    tag = node.name

    # 代码块 - 直接返回原始内容（不翻译）
    if tag == 'pre':
        code_el = node.find('code')
        code_text = code_el.get_text() if code_el else node.get_text()
        return f'\n```\n{code_text}\n```\n'

    if tag == 'code':
        parent = node.parent
        if parent and parent.name == 'pre':
            return f'```{node.get_text()}```'
        lang = node.get('class', [None])[0]
        if lang and 'language-' in str(lang):
            lang = str(lang).replace('language-', '')
        return f'`{node.get_text()}`'

    if tag in ('h1', 'h2', 'h3', 'h4', 'h5', 'h6'):
        level = int(tag[1])
        text = ''.join(to_markdown(n) for n in node.children).strip()
        return f'\n{"#" * level} {text}\n'

    if tag == 'a':
        href = node.get('href', '#')
        text = ''.join(to_markdown(n) for n in node.children).strip()
        if not text:
            text = href
        return f'[{text}]({href})'

    if tag == 'strong':
        text = ''.join(to_markdown(n) for n in node.children)
        return f'**{text}**'

    if tag == 'em':
        text = ''.join(to_markdown(n) for n in node.children)
        return f'*{text}*'

    if tag == 'blockquote':
        text = ''.join(to_markdown(n) for n in node.children).strip()
        return '\n> ' + text.replace('\n', '\n> ') + '\n'

    if tag in ('ul', 'ol'):
        items = []
        for li in node.find_all('li'):
            text = to_markdown(li).strip()
            if text:
                items.append(f'- {text}' if tag == 'ul' else f'1. {text}')
        return '\n' + '\n'.join(items) + '\n'

    if tag == 'table':
        rows = []
        for tr in node.find_all('tr'):
            cells = [td.get_text(strip=True) for td in tr.find_all(('td', 'th'))]
            if cells:
                rows.append(cells)
        if not rows:
            return ''
        header = '| ' + ' | '.join(rows[0]) + ' |'
        sep = '| ' + ' | '.join(['---'] * len(rows[0])) + ' |'
        data = '\n'.join('| ' + ' | '.join(r) + ' |' for r in rows[1:])
        return '\n' + header + '\n' + sep + '\n' + data + '\n'

    if tag == 'br':
        return '\n'

    if tag == 'p':
        text = ''.join(to_markdown(n) for n in node.children).strip()
        return '\n' + text + '\n'

    # 默认递归
    parts = []
    for child in node.children:
        parts.append(to_markdown(child))
    return ''.join(parts)

markdown_text = to_markdown(main)
```

## HTML 清洗清单

翻译前必须确认以下内容已处理：

| 噪音 | 特征 | 处理方式 |
|------|-|-|
| `theme-admonition` | `<div class="theme-admonition ...">` + SVG icon | 转为 `> [!NOTE]` / `> [!TIP]` / `> [!WARNING]` |
| 复制按钮 | `<button class="...copyButton...">` | 移除 |
| 侧边栏/导航 | `<nav>`, `<aside>`, `.navbar`, `.sidebar` | 移除 |
| 活跃标记 | class 含 `active` | 移除元素或属性 |
| 数据属性 | `data-expanded`, `data-min-heading-level` | 移除所有 `data-*` |
| 代码块 class | `<code class="language-yaml">` | 保留语言标识，移除 class |
| `href="#"` 锚点占位符 | href 为 # 的链接 | 转为 `[text](#heading)`，保留 #heading URL |

## 推荐获取优先级

```
curl raw URL ──(成功)──→ 直接翻译 ──→ write_file
        │
        └──(失败/无raw)──→ web_extract ──(内容>500)──→ 翻译 ──→ write_file
                                │
                                └──(失败/短页面)──→ browser_navigate + innerHTML
                                                       │
                                                       └── execute_code 清洗管道 ──→ 翻译 ──→ write_file
```

## 翻译后 QA 清洗（重要）

将翻译后的 `.md` 文件写入本地前，必须运行以下清洗步骤——这些字段是文档站（Docusaurus）渲染用，本地阅读不需要：

### QA Step 1：移除文档站专用 frontmatter 字段（含中文检测）

**核心规则**：只要 frontmatter 中含有 `sidebar_position`，就必须处理。**是否移除整个 frontmatter 块取决于其中是否包含中文字符。**

#### 检测逻辑（必做）

```python
import re

fm_match = re.match(r'^---\n(.*?)\n---', content, re.DOTALL)
if fm_match:
    fm_text = fm_match.group(1)
    has_chinese = bool(re.search(r'[\u4e00-\u9fff]', fm_text))
    has_sidebar = 'sidebar_position' in fm_text

    if has_chinese and has_sidebar:
        # 模式 A：frontmatter 是翻译产物（含中文）→ 移除整个 frontmatter 块
        content = re.sub(r'^---\n.*?\n---\n?', '', content, count=1, flags=re.DOTALL)
    elif has_sidebar and not has_chinese:
        # 模式 B：title/description 为英文原文 → 仅移除 sidebar_position
        content = re.sub(r'^sidebar_position:.*\n', '', content, count=1, flags=re.MULTILINE)
    else:
        # 无 sidebar_position → frontmatter 保留（可能是作者原始 frontmatter）
```

#### 三类场景速查

| 场景 | frontmatter 特征 | 处理方式 | 占比 |
|------|----------------|--------|------|
| **模式 A** | 含中文字符 + sidebar_position | 移除整个 `---...---` 块 | 最常见（27/33） |
| **模式 B** | 无中文（title/description 为英文）+ sidebar_position | 仅删除 `sidebar_position:` 行 | 较少（5/33） |
| **无 sidebar_position** | 无 sidebar_position | 保留原始 frontmatter | 无需处理 |

**翻译后生成 frontmatter 时绝不要添加 `sidebar_position`、`title`、`description`、`sidebar_label`** —— 这些是文档站渲染用字段，本地阅读不需要。

### QA Step 2：移除 HTML admonition 残渣

Docusaurus 翻译时可能残留 DOM 层级的 HTML 标签：

```html
<div class="theme-admonition theme-admonition-tip admonition_xJq3 alert alert--success">
  <div class="admonitionHeading_Gvgb">
    <span class="admonitionIcon_Rf37">
      <svg viewBox="0 0 12 16">...</svg>
    </span>开始使用
  </div>
  <div class="admonitionContent_BuS1">
    <p>内容...</p>
  </div>
</div>
```

处理方式：
- 检测文件中是否存在 `theme-admonition`、`admonitionIcon`、`admonitionHeading`、`alert alert--success` 等特征
- 有则删除整行（通常是文件开头第 3 行左右的长 `<div>` 标签）
- 如果之前清洗管道已将 admonition 转为 `> [!NOTE]` 格式则无需处理

### QA Step 3：验证 frontmatter 中文检测逻辑生效

翻译后写入文件前，用 Python 验证所有文件的 frontmatter 已按正确规则处理：

```python
import re

def check_fm_clean(content):
    fm_match = re.match(r'^---\n(.*?)\n---', content, re.DOTALL)
    if not fm_match:
        return True  # 无 frontmatter → 通过
    fm_text = fm_match.group(1)
    if 'sidebar_position' in fm_text:
        if re.search(r'[\u4e00-\u9fff]', fm_text):
            return False  # 有中文 + sidebar_position = 未处理（应移除整个块）
        return True     # 仅 sidebar_position 行（已用 模式 B 处理）
    return True  # sidebar_position 不存在 → 通过

assert check_fm_clean(content), "frontmatter 中仍有未处理的 sidebar_position + 中文"
```

### QA Step 4：完整性验证

- 文件以 `# ` 标题开头（不是 `---` frontmatter）— 模式 A 的文件应已移除整个 frontmatter
- 不含 `<div class=`、`theme-admonition`、`admonitionIcon` 等 HTML 标签
- 无 `sidebar_position` 字段
- 中文内容完整、无截断

## 注意事项

- **`web_extract` 对长文档永远不适用** —— 限制约 5000 字符且会总结页面。有 GitHub 仓库时，始终先用 `curl` 拉 raw markdown。
- **每个 URL 单独处理** —— 不要并行化 fetch+translate，避免上下文问题并确保每个翻译完整。
- **对长文档的读取策略**：`curl` + `read_file` 优先于浏览器。`read_file` 通过 `offset`/`limit` 分页读取大文件（如 10KB+ 的文档分块读取）。
- **`browser_console` 大量输出会写入临时文件** —— 不要假设它直接返回。用 `read_file` 在临时文件上分页读取。
- **批量拉取场景**：如果有 5+ 个文档需要翻译，用 `execute_code` 批量 `requests.get()` 拉取所有 raw URL 保存到临时文件，再逐个打开翻译。这比逐个 curl 快得多。
- **验证内容完整性** ：如果文档 > 15KB 或分多个 chunk 读取，在翻译前检查每个 chunk 末尾是否完整（没有截断）。如果中间文档（chunk 2+）内容不完整，重新拉取。
- **HTML 清洗后验证** —— 用 `read_file` 读取生成的中间 Markdown 文件，确认结构完整（标题层级正确、代码块完整、表格格式正常）。
- **侧边栏标题与文件名不匹配**：有些侧边栏菜单标题（如 "Weixin (WeChat)"）不匹配文件路径 slug（如 `weixin.md`）。应该用 URL 路径而非菜单标题来生成 raw URL。
- **侧边栏计数确认**：浏览器快照中的侧边栏计数（如 "Guides & Tutorials (19)"）可能与 API 返回的文件数不一致（API 可能只返回直接子目录文件）。始终以浏览器侧边栏计数为准，但用 API 获取文件列表。
- **文档计数可能变化**：文档网站的侧边栏计数会随更新变化。不要硬编码计数，每次用 `browser_snapshot` + 解析侧边栏获取实时列表。


