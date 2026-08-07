---
name: confluence-pages
description: 通过 mcp-atlassian 读写 Confluence 页面的规范——Markdown 整页同步、标题映射、附件上传、表格排版、Storage 双路径和写后验证。仅手动触发。
disable-model-invocation: true
---

# Confluence Pages（mcp-atlassian 专用）

前提：环境中的 `confluence_*` 工具来自开源 mcp-atlassian server。`confluence_create_page` / `confluence_update_page` 支持 `content_format` 参数（`markdown` 默认 / `wiki` / `storage`）。

## 安全边界

- 只操作用户在指令中给出 URL 或 Page ID 的页面。页面位置不明确时，向用户索要链接。
- 页面写操作只用：`confluence_create_page`、`confluence_update_page`、`confluence_add_comment`、`confluence_add_label`。
- 附件写操作只用：`confluence_upload_attachment`、`confluence_upload_attachments`；附件读取使用 `confluence_get_attachments`。
- 删除页面、移动层级、修改权限、删除附件：提示用户在 Web 端手动完成，本 skill 不调用对应工具（包括 `confluence_delete_page`）。
- 本 skill 不涉及 `jira_*` 工具。

## Markdown 整页同步规则

当用户要求“将某个 Markdown 文档覆盖/同步到 Confluence 页面”时，必须先执行文档级预处理，不能把 Markdown 原样写入。

### 页面标题

- 文档第一个非空块如果是唯一的 `# 一级标题`，将其作为 Confluence page title，并从正文中移除。
- Confluence 正文中不允许出现一级标题；正文应从 `##` 或更低级别开始。
- 如果正文中还有其他 `# 一级标题`，先向用户确认；用户已明确要求整页同步时，可将后续一级标题降级为 `##`。
- 部分更新页面时保留当前 page title；整页覆盖且用户明确要求用文档覆盖时，使用文档一级标题更新 page title。
- page title 是纯文本元数据，无法通过 mcp-atlassian 设置居中、字体或颜色。禁止为了模拟标题居中而在正文中保留或伪造 H1。

### 本地引用

- 写入前必须扫描 Markdown 中的相对链接、HTML `<img>`、`<figure>` 和其它本地资源引用。
- 本地文件不能直接以仓库路径形式作为 Confluence 可点击链接。
- 图片、JSON、XLSX、TMPL、PDF、ZIP 等被正文直接引用的文件必须上传到当前页面，并替换为 Confluence 附件引用。
- 本地 Markdown 文件默认不上传，渲染为代码样式的仓库路径；如果用户提供目标 Confluence 页面映射，再转换为页面链接。
- 外部 HTTP/HTTPS 链接保留。
- 无法解析或不存在的相对链接，写入前必须向用户报告，不能静默保留成无效链接。

### Raw HTML 与图示

- 不把 Markdown 中的任意 raw HTML 原样写入 Confluence。
- `<figure><img src="./..."></figure>` 转换为上传附件 + `ac:image` + 独立图注段落。
- `<template>` 中的 AI Context、Mermaid 源等非页面展示内容，默认不进入正文；用户明确要求保留时，转换为 code macro。
- Mermaid 图优先使用源文档已生成的 SVG/PNG 附件；没有生成物时，保留为 code macro 或请求用户确认是否渲染。

## 格式决策：两条排版路径

**单次 create/update 调用的整个 body 只能是一种格式，不可混排。**

- **路径 A — Markdown（默认）**：`content_format` 显式传 `"markdown"`。覆盖标题、列表、普通表格、代码块（```lang）、外部链接、外部图片。与其它项目 Markdown 文档风格统一。
- **路径 B — Storage XHTML**：当页面需要 Confluence 专有宏、页面附件引用、图片宽度/居中控制或更明确的表格控制时，**整页**改用 `content_format="storage"`，先 read 本 skill 目录下的 `references/storage-format.md`。

以下情况必须使用路径 B：

- 正文需要引用本地附件，尤其是 JSON、XLSX、TMPL、PDF 等非图片附件；
- 需要控制图片宽度、居中或引用页面附件；
- 需要 Confluence 宏（info / warning / expand / toc 等）；
- 需要比 Markdown 更明确地控制表格宽度、单元格换行或复杂排版；
- Markdown 中包含 `<figure>`、`<template>`、Mermaid 上下文块等不能原样进入 Confluence 的 HTML 结构，需要先转换。

决策规则：

- 新建纯 Markdown 页面 → 路径 A。
- 新建页面包含本地附件、页面图片、宏或复杂表格 → 路径 B。
- 更新已有页面 → 先 `confluence_get_page`：返回内容中存在 Markdown 无法表达的结构（宏、页面附件、嵌套布局、固定宽度图片等）→ 路径 B，保持页面原格式；否则 → 路径 A。
- 路径 A 下用户要求宏或本地附件引用 → 告知“该页需整页切换为 storage 格式”，确认后再执行。

## 表格排版

### Markdown 路径

- `table_layout` 和 `page_width` 的默认值与例外见「调用参数纪律 > 默认参数值」。
- 不为了修复表格而擅自修改 `page_width`。

### Storage 路径

- 普通表格使用 `class="confluenceTable"`。
- 默认表格 layout 必须显式写为 Confluence 默认布局：`<table class="confluenceTable" data-layout="default" data-table-width="760">`。
- 禁止用 `style="width: 100%; table-layout: fixed;"` 代替 Confluence layout；这会绕过页面编辑器的表格布局状态，导致页面显示为非 default layout。
- 长路径、长英文标识符和代码文本允许换行，必要时只给 `th`/`td` 单元格增加 `overflow-wrap:anywhere;`，不要改 table 的 `data-layout`。
- 只有用户明确要求宽表/全宽，或目标页面已是 full-width 且本次任务不改版式时，才将 `data-layout` 改为 `wide` / `full-width`，并按需设置对应 `data-table-width`。
- 不使用固定像素宽度，除非用户明确要求。
- 超过 4 列或单元格包含大段文字的表格，优先考虑拆成多个小表格、改成小节 + 列表，或将详细内容放入 expand 宏。

## 调用参数纪律

### 默认参数值

以下默认值适用于 `confluence_create_page` / `confluence_update_page`，除非用户明确要求，否则不得偏离：

| 参数 | 默认值 | 例外 |
|------|--------|------|
| `page_width` | `"default"` | 用户明确要求全宽，或目标页面已是 full-width 且本次任务不改版式时，才用 `"full-width"` |
| `table_layout` | `"default"` | 用户明确要求宽表，或页面已设为 full-width 时，才用 `"wide"` / `"full-width"` |
| `content_format` | `"markdown"` | 命中「格式决策」路径 B 条件时用 `"storage"` |
| `include_content` | 不传（false） | 写后验证统一走 `confluence_get_page` 读回，不依赖响应带回正文 |
| `is_minor_edit` | 不传（false） | 仅用户明确说明是小改动、不希望通知订阅者时传 `true` |
| `version_comment` | 不传 | 仅整页同步等需要可追溯来源时填写（如 `synced from <repo 路径>`）；禁止无意义占位注释 |
| `enable_heading_anchors` | 不传（false） | 页面需要被按标题锚点跳转（目录页、FAQ、长规范文档）时开启 |
| `emoji` | 不传 | 用户明确要求，或同空间同级页面已有统一 emoji 规范时设置 |
| `parent_id` | 无默认，禁止猜测 | 只能来自用户明确给出的页面，或列出页面树由用户选择 |

注意：`table_layout` 参数只作用于 Markdown 路径中由 mcp-atlassian 生成的表格；Storage 路径必须在每个 `<table>` 上显式设置 `data-layout="default"`（以及默认 `data-table-width="760"`），不能依赖调用参数修正 Storage 正文内的表格布局。

正文传参方式：正文超过约 20KB 时使用 `content_file`（按容器挂载约定写入 `/app/uploads/...` 路径），否则使用 `content` inline 传入。

- **硬禁令：禁止在页面正文、评论、图片引用或 `data:` URI 中写入 base64 编码内容**。正文只需纯文本 `content`（或 `content_file`）；读取阶段拿到的 base64 图片内容仅用于查看，不得进入正文写操作的任何参数。
- **容器路径不可读时优先使用 staging**：当 MCP server 只能读取容器内 `/app/uploads` 时，先把待上传文件真实复制到宿主机挂载目录，再用容器路径调用附件上传工具；不要优先使用 base64。
- **base64 仅作最后兜底**：只有无法使用 `/app/uploads` staging 时，`confluence_upload_attachment` 才允许用 `content_base64 + filename` 代替 `file_path`。

## 容器 `/app/uploads` 文件桥接

当前 MCP server 的宿主机挂载约定为：

```text
宿主机：/Users/v1fanchao/data/mcp-atlassian
容器内：/app/uploads
```

当附件文件位于仓库中、MCP server 无法直接读取仓库路径时，使用以下流程：

1. 为当前页面创建独立 staging 目录：`/Users/v1fanchao/data/mcp-atlassian/<page-id>/`。
2. 将正文明确引用且用户授权读取的本地文件**真实复制**到该目录；禁止使用软链接代替复制。
3. 调用附件上传工具时使用容器路径：`/app/uploads/<page-id>/<filename>`。
4. 附件文件名默认保持源文件 basename；不要为了避免冲突随意改名，除非正文引用同步修改。
5. 上传成功后可以清理该页面的 staging 文件；如果后续还要重试或排查，可先保留并在最终汇报中说明。
6. 多文件上传可传逗号分隔的容器路径；失败时逐个用 `confluence_upload_attachment` 定位问题。

禁止把宿主机绝对路径（例如 `/Users/.../docs/...`）直接传给附件上传工具；MCP server 只认容器内路径。

## 写入纪律

`confluence_update_page` 是**全量替换 body**——mcp-atlassian 没有局部更新工具。任何局部修改必须走完整流程：

1. `confluence_get_page` 拉取当前全量内容。
2. 本地合并出完整的新 body：只改目标区域，其余部分原样保留；写入前确认目标内容尚不存在（幂等，重复执行不得产生重复段落）。
3. 用户已明确要求“整页覆盖/用某文档覆盖”时，不再因覆盖比例重复确认；否则若预计覆盖/删除超过原内容约 50%，先向用户展示变更摘要，确认后再写。
4. 整体写回。遇到 409/版本冲突：重新 get_page → 重新合并 → 重试一次；仍失败则向用户报错（可能有人在 Web 端同时编辑）。
5. 大页面：分段读取，合并时只动目标区域。

## 图片与附件

### 附件处理顺序

更新已有页面：

1. 解析源文档中的所有本地资源引用。
2. 调用 `confluence_get_attachments` 获取页面已有附件。
3. 如果 MCP server 无法直接读取仓库文件，先按“容器 `/app/uploads` 文件桥接”复制待上传文件。
4. 对缺失或可能更新的附件调用 `confluence_upload_attachment` / `confluence_upload_attachments`。
5. 附件上传成功后，再更新正文。
6. 如果附件上传失败，不得在正文中写入指向该附件的引用；应向用户报告并选择中止或降级为仓库路径文本。

新建页面：

1. 先创建页面以获得 page id。
2. 需要 staging 时，将附件复制到 `/Users/v1fanchao/data/mcp-atlassian/<page-id>/`。
3. 使用 `/app/uploads/<page-id>/<filename>` 上传附件。
4. 再次更新正文，写入附件引用。

### 附件引用规则

- 图片附件使用 `ac:image + ri:attachment`。
- 非图片附件使用 `ac:link + ri:attachment`。
- 引用前先确认附件已经存在于当前页面。
- 附件文件名默认使用源文件 basename。
- 页面已有语义相同但文件名不同的旧附件时，默认仍上传源文件 basename 对应的新附件，并保留旧附件不删除。
- 重复执行同步时避免无脑重复上传产生大量附件版本：文件名和大小一致且内容未确认变化时可复用；无法判断时向用户说明选择。

### 禁止行为

- 禁止把 `./xxx.json`、`./xxx.xlsx` 这类仓库相对路径当成 Confluence 可点击链接。
- 禁止引用尚未上传到当前页面的附件，否则渲染为裂图或死链。
- 禁止在正文中使用 `data:` URI 或 base64 图片。
- 禁止把宿主机绝对路径传给附件上传工具；容器部署时必须使用 `/app/uploads/...` 路径。
- 禁止用软链接构造 staging 文件，必须真实复制。

## 写入前检查

真正调用 create/update 前，必须完成以下检查：

- [ ] page title 已确定，正文无 H1；
- [ ] 所有本地附件已上传或明确降级；
- [ ] staging 文件位于 `/Users/v1fanchao/data/mcp-atlassian/<page-id>/`，上传参数使用 `/app/uploads/<page-id>/...`；
- [ ] 不存在未处理的仓库相对链接；
- [ ] 不存在 base64/data URI；
- [ ] 表格采用合适的宽度策略；
- [ ] raw HTML、figure、template、Mermaid 已转换；
- [ ] Storage XML well-formed；
- [ ] 附件引用文件名与实际上传文件名一致。

## 写后验证与汇报

完成标准（未满足不算完成）：

1. 写入后 `confluence_get_page` 读回，确认 page metadata title 等于预期标题。
2. 确认正文不存在一级标题或重复页面标题。
3. 调用 `confluence_get_attachments`，确认所有本地资源已出现在附件列表中。
4. 确认正文中的每个 `ri:attachment` 都能匹配到当前页面附件。
5. 确认表格包含 `confluenceTable`；默认页面下每个表格都有 `data-layout="default"`，且没有 `style="width: 100%; table-layout: fixed;"` 或硬编码超宽尺寸。
6. 确认图片使用页面附件或有效外链，而不是无效相对路径。
7. 确认标题层级、代码块、宏、图注等关键结构确实存在于返回内容中。
8. 确认正文中没有残留仓库相对链接、raw HTML、`data:` URI 或 base64。

向用户汇报必须包含：

- 页面 URL；
- page title 是否更新；
- 正文结构变更摘要；
- 上传/复用了哪些附件；
- 哪些本地链接没有上传及原因；
- 读回验证结果。

## 评论

`confluence_add_comment` 遵循同一路径决策，默认 Markdown。
