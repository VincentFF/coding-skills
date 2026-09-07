# Confluence Storage Format (XHTML)

Load this reference only for the Storage XHTML path on Confluence Cloud.

## Core rules

- Produce well-formed XHTML: close every tag, quote attributes, and use `<br/>`.
- Escape bare `&` as `&amp;` and bare `<` as `&lt;` in text. Split `]]>` inside CDATA as `]]]]><![CDATA[>`.
- Keep the page title in page metadata; never include an H1 in the body.
- If a macro fails to render, retry it with `ac:schema-version="1"`.
- Use semantic HTML and the team's theme instead of hard-coded fonts.

```xml
<h2>Section</h2>
<h3>Subsection</h3>
<p>Text with <strong>emphasis</strong>, <em>detail</em>, and a<br/>line break.</p>
```

## Tables

Use native Confluence classes and the default layout unless the user explicitly requests a wider table or the unchanged page is already full width.

```xml
<table class="confluenceTable" data-layout="default" data-table-width="760">
  <tbody>
    <tr>
      <th class="confluenceTh" style="text-align: left; overflow-wrap: anywhere;">Column</th>
      <th class="confluenceTh" style="text-align: center; overflow-wrap: anywhere;">Status</th>
    </tr>
    <tr>
      <td class="confluenceTd" style="overflow-wrap: anywhere;">Long path or identifier</td>
      <td class="confluenceTd" style="text-align: center; overflow-wrap: anywhere;">Ready</td>
    </tr>
  </tbody>
</table>
```

Do not emulate layout with `style="width: 100%; table-layout: fixed;"`. Apply `overflow-wrap:anywhere;` to cells, not the table. Prefer multiple smaller tables, lists, or an expand macro for more than four columns or dense text.

## Attachments

Upload the file to the current page and verify it exists before inserting either reference.

```xml
<p style="text-align: center;">
  <ac:image ac:align="center" ac:width="700">
    <ri:attachment ri:filename="diagram.png" />
  </ac:image>
</p>
<p>Figure 1. Diagram caption.</p>

<ac:link>
  <ri:attachment ri:filename="report.pdf" />
  <ac:plain-text-link-body><![CDATA[Download the report (PDF)]]></ac:plain-text-link-body>
</ac:link>
```

Use `ri:url` for external images. The `ri:filename` value must exactly match the uploaded attachment name.

## Common macros

```xml
<ac:structured-macro ac:name="info" ac:schema-version="1">
  <ac:rich-text-body><p>Context or guidance.</p></ac:rich-text-body>
</ac:structured-macro>

<ac:structured-macro ac:name="warning" ac:schema-version="1">
  <ac:rich-text-body><p>Risk or required attention.</p></ac:rich-text-body>
</ac:structured-macro>

<ac:structured-macro ac:name="code" ac:schema-version="1">
  <ac:parameter ac:name="language">python</ac:parameter>
  <ac:plain-text-body><![CDATA[
def hello():
    print("Hello")
]]></ac:plain-text-body>
</ac:structured-macro>

<ac:structured-macro ac:name="expand" ac:schema-version="1">
  <ac:parameter ac:name="title">Details</ac:parameter>
  <ac:rich-text-body><p>Supplementary content.</p></ac:rich-text-body>
</ac:structured-macro>

<ac:structured-macro ac:name="toc" ac:schema-version="1" />
```
