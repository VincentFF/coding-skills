# Confluence Storage Format (XHTML) 参考

仅当走**路径 B**（`content_format="storage"`，整页 XHTML）时加载本文件。目标环境：Confluence Cloud。

## 通用规则（违反即 400 或渲染破损）

- **Well-formed**：所有标签必须正确闭合（`<br/>` 而非 `<br>`），属性值加引号。
- **转义**：body 文本中的裸 `&` 写为 `&amp;`，裸 `<` 写为 `&lt;`。
- **CDATA**：代码块内容含 `]]>` 时，拆分为 `]]]]><![CDATA[>`。
- **宏容错**：宏渲染失败时，给 `ac:structured-macro` 补充 `ac:schema-version="1"` 再试。
- **页面标题**：Confluence page title 是元数据，不写在 body 中；正文不要放 `<h1>`。page title 的对齐和字体由 Confluence 主题控制。

## 标题与段落

不硬编码字体，保持团队默认主题；用基础标签控制样式。

```xml
<h2>正文一级小节</h2>
<h3>正文二级小节</h3>
<h2 style="text-align: center;">需要居中的正文小节标题</h2>
<p>标准段落，<strong>加粗</strong>、<em>斜体</em>，行内换行用<br/>而非连续空段落。</p>
```

## 表格

必须带 Confluence 原生 class，才能激活默认边框与背景色。默认表格 layout 必须显式写为 Confluence default，并允许长文本在单元格内换行。

```xml
<table class="confluenceTable" data-layout="default" data-table-width="760">
  <tbody>
    <tr>
      <th class="confluenceTh" style="text-align: left; overflow-wrap: anywhere;">列头 1</th>
      <th class="confluenceTh" style="text-align: center; overflow-wrap: anywhere;">列头 2</th>
    </tr>
    <tr>
      <td class="confluenceTd" style="overflow-wrap: anywhere;">很长的数据或路径 1</td>
      <td class="confluenceTd" style="text-align: center; overflow-wrap: anywhere;">数据 2</td>
    </tr>
  </tbody>
</table>
```

表格规则：

- 默认使用 `data-layout="default" data-table-width="760"`；不要用 `style="width: 100%; table-layout: fixed;"` 代替 layout。
- 长路径、长英文标识符和代码文本使用单元格级 `overflow-wrap:anywhere;`，不要为换行修改 table 的 `data-layout`。
- 只有用户明确要求宽表/全宽，或目标页面已是 full-width 且本次任务不改版式时，才使用 `data-layout="wide"` / `data-layout="full-width"`。
- 不使用固定像素宽度，除非用户明确要求。
- 超过 4 列或单元格包含大段文字时，优先拆成多个小表格、改成小节 + 列表，或使用 expand 宏。

## 附件图片

用 `ac:width` 防止图片超出屏幕宽度；引用的附件必须已上传到该页面。

附件图片（居中限宽，推荐）：

```xml
<p style="text-align: center;">
  <ac:image ac:align="center" ac:width="700">
    <ri:attachment ri:filename="你的图片名称.png" />
  </ac:image>
</p>
<p>图 1 图片说明</p>
```

外链图片：

```xml
<p style="text-align: center;">
  <ac:image ac:align="center" ac:width="700">
    <ri:url ri:value="https://example.com/image.png" />
  </ac:image>
</p>
```

## 附件文件链接

JSON、XLSX、TMPL、PDF、ZIP 等非图片附件，先上传到当前页面，再使用 `ac:link + ri:attachment`：

```xml
<ac:link>
  <ri:attachment ri:filename="alicloud-alert-hk-rules.xlsx" />
  <ac:plain-text-link-body><![CDATA[阿里云香港告警规则汇总 xlsx]]></ac:plain-text-link-body>
</ac:link>
```

规则：

- `ri:filename` 必须与实际上传到当前页面的文件名一致。
- 不使用仓库相对路径作为 Confluence 链接。
- 引用前先通过 `confluence_get_attachments` 确认附件存在。

## 常用宏

信息面板（蓝色，提示说明）：

```xml
<ac:structured-macro ac:name="info" ac:schema-version="1">
  <ac:rich-text-body>
    <p>这里是提示说明文字。</p>
  </ac:rich-text-body>
</ac:structured-macro>
```

警告面板（黄/红色，风险警告）：

```xml
<ac:structured-macro ac:name="warning" ac:schema-version="1">
  <ac:rich-text-body>
    <p>注意：这是一个高风险操作。</p>
  </ac:rich-text-body>
</ac:structured-macro>
```

代码块（保留缩进与语法高亮，内容包在 CDATA 中）：

```xml
<ac:structured-macro ac:name="code" ac:schema-version="1">
  <ac:parameter ac:name="language">python</ac:parameter>
  <ac:parameter ac:name="theme">Confluence</ac:parameter>
  <ac:plain-text-body><![CDATA[
def hello_world():
    print("Hello, Confluence!")
  ]]></ac:plain-text-body>
</ac:structured-macro>
```

折叠块（隐藏长日志或补充信息）：

```xml
<ac:structured-macro ac:name="expand" ac:schema-version="1">
  <ac:parameter ac:name="title">点击展开查看详细日志</ac:parameter>
  <ac:rich-text-body>
    <p>这里是隐藏的详细内容...</p>
  </ac:rich-text-body>
</ac:structured-macro>
```

目录（自动生成页面标题索引）：

```xml
<ac:structured-macro ac:name="toc" ac:schema-version="1" />
```
