---
name: web-docs-to-zh
description: 使用当用户提供在线文档 URL（文档站、帮助文档、API 文档等），要求翻译成中文、获取中文内容、保存为中文 Markdown 文件。自动处理 raw markdown 获取、HTML DOM 清洗（Docusaurus/Mintlify/VitePress/Sphinx）。
author: kang99
version: 1.2.0
license: MIT
metadata:
  hermes:
    tags: [documentation, translation, chinese, markdown, docusaurus, mintlify, vitepress, html-cleaning, mkdocs]
    related_skills: []
---

# web-docs-to-zh — Web 文档翻译

将在线文档站网页内容翻译成中文并保存为 Markdown 文件。提供一套通用清洗管道，专门处理文档站的特殊语法和 DOM 噪音。

## 何时使用 (When to Use)

用户提供文档 URL 列表（通常一行一个，用空白行分隔），要求获取并翻译成中文 `.md` 文件保存到指定目录。

### 中文触发短语

**基础句式：**
- 「把这些文档转成中文markdown存到xxx」
- 「帮忙把这几个网页文档翻译」
- 「把这段网址的文档翻译成中文」
- 「把这个文档站的中文版搞定」
- 「把这些doc翻译一下保存」
- 「把在线文档获取一下，翻译为中文」
- 「帮我把这个doc网页转中文md」
- 「抓取这些链接的文档，译成中文」

**变体句式：**
- 「把这些网页文档搞成中文版」
- 「帮我把这个文档站译成中文md」
- 「把这几个链接的文档拉下来翻译」
- 「把xxx文档站的中文md版本搞一份」
- 「这些文档帮我翻译成中文markdown」
- 「把这篇/这些在线文档转中文md」

**关键词组合触发：**
- 「xxx文档 / doc / docs」+「翻译」/「中文版」/「中文md」/「中文化」
- 「xxx wiki」+「整理」/「翻译」/「md」
- 「xxx guide」+「翻译」/「中文」/「markdown」
- 「xxx help / 帮助文档」+「翻译」/「md」

**英文触发短语：**
- "Translate this documentation to Chinese"
- "Get the content in Chinese markdown"
- "Translate docs to zh-cn"
- "Convert this doc to Chinese"
- "Save this doc as Chinese MD"
- "Download this documentation in Chinese"

## 工作流

### 场景 A：文档有 GitHub 源码仓库（首选路径）

直接用 `curl` 获取 raw markdown，运行 **Markdown 预处理** 清洗特殊语法，然后翻译。

### 场景 B：无 raw URL，需要使用 HTML 清洗管道

当无法获取 raw URL 时，通过 `browser_navigate` + `browser_console` 提取 HTML DOM，然后运行 **HTML 清洗管道**。

## 步骤

### 1. 创建目标目录

```bash
mkdir -p "/path/to/target/dir"
```

### 2. 解析 URL

从用户请求中解析 URL —— 通常一行一个，用空白行分隔。

### 3. 内容获取（每个 URL 单独处理，不要并行）

#### 3a. 首选：curl 获取 raw markdown（有 GitHub 仓库时）

从文档页 URL 推导 raw markdown URL。常见模式：

- `/docs/path/to/page` → `raw.githubusercontent.com/<owner>/<repo>/main/website/docs/path/to/page.md`
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

#### 3c. 最后手段：浏览器 DOM 提取

当 raw URL 不可用时，通过浏览器提取 HTML：

1. 首选 `browser_console` 提取 `innerHTML`：
   ```
   browser_console(expression="document.querySelector('article, main, .theme-doc-markdown, .docs-content')?.innerHTML")
   ```
2. 备选 `innerText`（纯文本，跳过清洗管道直接翻译，但会丢失结构信息）。
3. 最后手段：`browser_snapshot(full=true)` —— 慢且超过 8000 字符时会截断。

#### 3d. Markdown 预处理（针对 raw markdown 源文件）

某些文档站（如 MkDocs Material）使用非标准 Markdown 语法（CommonMark 不支持）。获取 raw URL 后、翻译前，**必须**执行以下清洗步骤：

**步骤 1**：选项卡 (`===`) → 子标题 `###`

MkDocs Material 用 `=== "Tab Name"` 创建内容分组选项卡，本地阅读应改为子标题：

```python
text = re.sub(r'^\s*==={3,}\s*"(.+?)"\s*$', r'### \1', text, flags=re.MULTILINE)
```

**步骤 2**：折叠块语法 → 普通内容

处理以下三种折叠语法：

1. `?`` Title` 语法 — 直接转为子标题：
   ```python
   text = re.sub(r'^\s*\?\?\?\s+(.+?)(?:\n\n|\Z)', lambda m: f'#### {m.group(1)}\n', text, flags=re.MULTILINE)
   ```

2. 以 `` ```语言名 `` 开头的折叠语法（MkDocs 折叠而非代码块） — 提取标题并保留块内内容：
   ```python
   def strip_code_folds(text):
       """将 ```python 等折叠语法转为 #### python，保留块内内容"""
       lines = text.split('\n')
       result = []
       skip_until_close = False
       title = None
       for line in lines:
           stripped = line.lstrip()
           if not skip_until_close and stripped.startswith('```'):
               lang = stripped[3:].strip()
               title = f'#### {lang}' if lang else ''
               skip_until_close = True
               continue
           if skip_until_close and stripped == '```':
               skip_until_close = False
               result.append(title if title else '')
               title = None
               continue
           if skip_until_close:
               result.append(line)
               continue
           result.append(line)
       return '\n'.join(result)
   ```

3. `<details><summary>` HTML 折叠块 — 提取 summary 内容作为 `#### Title`，移除包裹标签：
   ```python
   text = re.sub(
       r'<details[^>]*>\s*<summary[^>]*>\s*(.+?)\s*</summary>\s*<[^>]*>(.*?)</details>',
       lambda m: f'#### {m.group(1).strip()}\n{m.group(2).strip()}',
       text, flags=re.IGNORECASE | re.DOTALL
   )
   # 无包裹标签的简单 <details> 块直接移除
   text = re.sub(r'<details[^>]*>.*?</details>', '', text, flags=re.IGNORECASE | re.DOTALL)
   ```

**步骤 3**：锚点交叉引用（`[text][ref]`） → 普通文本

MkDocs Material 用 `[LLM][vllm.LLM]` 语法引用锚点，这些引用定义在文档末尾。本地阅读时引用链会断裂，应移除第二个引用括号：

```python
# 正确匹配 [text][ref] 结构：两个 [^]]+ 都不可包含 ]
text = re.sub(r'\[([^\]]+)\]\[([^\]]+)\]', r'\1', text)
```

⚠️ **注意**：不要写成 `\[( [^\]])+\\]`（漏掉第一个 `[^\]` 中的 `]` 转义），正确写法是 `[^\]]+`，匹配"非 `]` 字符"。

**步骤 4**：检测 MkDocs 特征（用于判断是否需要清洗）

```python
import re

def has_mkdocs_features(text):
    features = []
    if re.search(r'==={3}', text):
        features.append('tabs (===)')
    if re.search(r'^\s*\?\?\?\s+', text, re.MULTILINE):
        features.append('folds (???)')
    if re.search(r'\[[^\]]+\]\[[^\]]+\]', text):
        features.append('cross-ref links')
    if re.search(r'<details>', text, re.IGNORECASE):
        features.append('HTML details')
    return features
```

**执行顺序**：按步骤 1→2→3 依次执行，先处理 `===` 和折叠块，再处理交叉引用。

### 4. 翻译

- **curl/raw markdown 场景**：直接翻译成中文，保留所有技术细节、代码块、YAML 配置、表格、链接和格式。
- **HTML 清洗场景**：先用清洗管道转为 Markdown，再翻译（见下方「HTML 清洗管道」章节）。

翻译通用原则：
- 格式锁定（标题、列表、代码块、表格、行内代码、链接保持原样不变）
- 代码块内的内容不翻译
- 保留技术专有名词不直译
- 不做主观添加
- 代码块保持原样

### 5. 文件命名

从页面标题或 URL slug 使用 kebab-case（短横线分隔小写）。
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
        text_parts = []
        for child in div.children:
            if isinstance(child, Tag) and child.name == 'svg':
                continue
            t = child.get_text(strip=True)
            if t and t.lower() not in [k for k in callout_map.values()]:
                text_parts.append(t)
        content = ' '.join(text_parts).strip()
        if content:
            marker = f'\n> [!{callout_map[ad_type]}]\n> {content}\n'
            div.insert_before(marker)
            div.extract()
    return node

main = convert_admonitions(main)
```

### Step 2：移除 DOM 噪音

```python
tags_to_remove = ['script', 'style', 'nav', 'aside', 'footer', 'header', 'button', 'iframe']
for tag_name in tags_to_remove:
    for el in main.find_all(tag_name):
        el.decompose()

cls_filters = ['admonitionIcon', 'copyButton', 'admonitionHeading', 'admonitionContent']
for cls_name in cls_filters:
    for el in main.find_all(class_=re.compile(cls_name)):
        el.decompose()

for el in main.find_all(attrs=re.compile(r'^data-')):
    del_attrs = [a for a in el.attrs if a.startswith('data-')]
    for a in del_attrs:
        del el[a]

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

    parts = []
    for child in node.children:
        parts.append(to_markdown(child))
    return ''.join(parts)

markdown_text = to_markdown(main)
```

## HTML 清洗清单

翻译前必须确认以下内容已处理：

| 噪音 | 特征 | 处理方式 |
|------|------|----------|
| `theme-admonition` | `<div class="theme-admonition ...">` + SVG icon | 转为 `> [!NOTE]` / `> [!TIP]` / `> [!WARNING]` |
| 复制按钮 | `<button class="...copyButton...">` | 移除 |
| 侧边栏/导航 | `<nav>`, `<aside>`, `.navbar`, `.sidebar` | 移除 |
| 活跃标记 | class 含 `active` | 移除元素或属性 |
| 数据属性 | `data-expanded`, `data-min-heading-level` | 移除所有 `data-*` |
| 代码块 class | `<code class="language-yaml">` | 保留语言标识，移除 class |
| `href="#"` 锚点占位符 | href 为 # 的链接 | 转为 `[text](#heading)`，保留 #heading URL |

## Markdown 预处理清单（MkDocs Material 文档的 raw source）

翻译前必须确认以下内容已处理：

| 噪音 | 特征 | 处理方式 |
|------|------|----------|
| MkDocs tab 选项卡 | `=== "Tab Name"` | 转为 `### Tab Name` 子标题 |
| MkDocs ?? 折叠块 | `??? Title` 行首 | 转为 `#### Title` 子标题 |
| MkDocs 代码折叠 | ```python 行首（无缩进） | 提取标题为 `#### python`，保留内容 |
| `<details><summary>` | HTML 折叠块标签 | 提取 summary 为 `#### TITLE`，移除包裹 |
| 锚点交叉引用 | `[text][ref]` | 移除第二层括号，变为 `text` |

## 推荐获取优先级

```
curl raw URL ──(成功)──→ Markdown 预处理 ──→ 翻译 ──→ write_file
        │
        └──(失败/无raw)──→ web_extract ──(内容>500)──→ 翻译 ──→ write_file
                                │
                                └──(失败/短页面)──→ browser_navigate + innerHTML
                                                       │
                                                       └── execute_code HTML 清洗管道 ──→ 翻译 ──→ write_file
```

## 翻译后 QA 清洗（重要）

将翻译后的 `.md` 文件写入本地前，必须运行以下清洗步骤——这些字段是文档站（Docusaurus/MkDocs）渲染用，本地阅读不需要。

### QA Step 1：移除文档站专用 frontmatter 字段

**核心规则：只要 frontmatter 中含有 `sidebar_position`，就必须处理。是否移除整个 frontmatter 块取决于其中是否包含中文字符。**

```python
import re

fm_match = re.match(r'^---\n(.*?)\n---', content, re.DOTALL)
if fm_match:
    fm_text = fm_match.group(1)
    has_chinese = bool(re.search(r'[\u4e00-\u9fff]', fm_text))
    has_sidebar = 'sidebar_position' in fm_text

    if has_chinese and has_sidebar:
        # 模式 A：frontmatter 含中文 → 移除整个 frontmatter 块
        content = re.sub(r'^---\n.*?\n---\n?', '', content, count=1, flags=re.DOTALL)
    elif has_sidebar and not has_chinese:
        # 模式 B：title/description 为英文 → 仅移除 sidebar_position
        content = re.sub(r'^sidebar_position:.*\n', '', content, count=1, flags=re.MULTILINE)
    else:
        # 无 sidebar_position → frontmatter 保留
        pass
```

#### 三类场景速查

| 场景 | frontmatter 特征 | 处理方式 |
|------|----------|--------|
| **模式 A** | 含中文字符 + sidebar_position | 移除整个 `---...---` 块 |
| **模式 B** | 无中文 + sidebar_position | 仅删除 `sidebar_position:` 行 |
| **无 sidebar_position** | 无 sidebar_position | 保留原始 frontmatter |

翻译后生成 frontmatter 时绝不要添加 `sidebar_position`、`title`、`description`、`sidebar_label` —— 这些是文档站渲染用字段，本地阅读不需要。

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
- 有则删除整行
- 如果之前清洗管道已将 admonition 转为 `> [!NOTE]` 格式则无需处理

### QA Step 3：验证 frontmatter 处理逻辑生效

```python
import re

def check_fm_clean(content):
    fm_match = re.match(r'^---\n(.*?)\n---', content, re.DOTALL)
    if not fm_match:
        return True
    fm_text = fm_match.group(1)
    if 'sidebar_position' in fm_text:
        if re.search(r'[\u4e00-\u9fff]', fm_text):
            return False  # 有中文 + sidebar_position = 未处理
        return True     # 仅 sidebar_position 行（已用模式 B 处理）
    return True

assert check_fm_clean(content), "frontmatter 中仍有未处理的 sidebar_position + 中文"
```

### QA Step 4：MkDocs Material 语法清洗验证

```python
def check_mkdocs_clean(content):
    issues = []
    if re.search(r'^\s*==={3,}\s*"', content, re.MULTILINE):
        issues.append('未清洗的 MkDocs 选项卡 (===)')
    if re.search(r'^\s*\?\?\?\s+', content, re.MULTILINE):
        issues.append('未清洗的 MkDocs 折叠块 (???)')
    if re.search(r'\[[^\]]+\]\[[^\]]+\]', content):
        issues.append('未清理的锚点交叉引用 ([text][ref])')
    if re.search(r'\s*</details>\s*$', content, re.MULTILINE):
        issues.append('未清洗的 HTML </details>')
    return issues
```

### QA Step 5：完整性验证

- 文件以 `# ` 标题开头（不是一串 `---` frontmatter）
- MkDocs 文档不含 `=== "Tab Name"`、`???`、`[text][ref]` 等特殊语法
- 不含 `<div class= theme-admonition`、`admonitionIcon` 等 HTML 标签
- code block 边界完整（``` 成对出现）
- 中文内容完整、无截断

## 注意事项

- **`web_extract` 对长文档永远不适用** —— 限制约 5000 字符且会总结页面。有 GitHub 仓库时，始终先用 `curl` 拉 raw markdown。
- **每个 URL 单独处理** —— 不要并行化 fetch+translate，避免上下文问题并确保每个翻译完整。
- **对长文档的读取策略**：`curl` + `read_file` 优先于浏览器。`read_file` 通过 `offset`/`limit` 分页读取大文件。
- **`browser_console` 大量输出会写入临时文件** —— 不要假设它直接返回。用 `read_file` 在临时文件上分页读取。
- **批量拉取场景**：如果有 5+ 个文档需要翻译，用 `execute_code` 批量 `requests.get()` 拉取所有 raw URL 保存到临时文件，再逐个打开翻译。
- **验证内容完整性**：如果文档 > 15KB 或分多个 chunk 读取，在翻译前检查每个 chunk 末尾是否完整（没有截断）。
- **HTML 清洗后验证** —— 用 `read_file` 读取生成的中间 Markdown 文件，确认结构完整。
- **侧边栏标题与文件名不匹配**：有些侧边栏菜单标题不匹配文件路径 slug。应该用 URL 路径而非菜单标题来生成 raw URL。
- **MkDocs Material 预处理必做**：MkDocs Material 文档的 raw source 使用了大量非标准 Markdown（`===` 选项卡、`???` 折叠、`[text][ref]` 锚点引用），翻译前**必须**清洗，否则本地 Markdown 阅读器无法正确显示。
- **文档计数可能变化**：文档网站的侧边栏计数会随更新变化。不要硬编码计数，每次用 `browser_snapshot` + 解析侧边栏获取实时列表。
