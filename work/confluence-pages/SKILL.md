---
name: confluence-pages
description: "Confluence pages: use mcp-atlassian when a user provides a page URL or ID and asks to create, edit, reorganize, format, sync Markdown, comment, label, or manage page attachments. Preserve content safely and validate every write."
---

# Confluence Pages (mcp-atlassian)

Use the `confluence_*` tools supplied by the open-source mcp-atlassian server. `confluence_create_page` and `confluence_update_page` support `markdown` (default), `wiki`, and `storage` content formats.

## Scope and allowed operations

- Act only on a page identified by a URL or page ID in the user's request. Ask for the link when the target is ambiguous.
- Write pages only with `confluence_create_page`, `confluence_update_page`, `confluence_add_comment`, and `confluence_add_label`.
- Read attachments with `confluence_get_attachments`; write them only with `confluence_upload_attachment` or `confluence_upload_attachments`.
- Direct users to the Confluence web UI for deleting pages or attachments, moving pages, or changing permissions. Do not use deletion tools.
- Do not use `jira_*` tools in this skill.

## Choose one body format

A single create or update request must use exactly one format.

- **Markdown** — Use for normal headings, lists, simple tables, code blocks, and external links or images. Pass `content_format: "markdown"` explicitly.
- **Storage XHTML** — Use for Confluence macros, page attachments, attachment images, image sizing or alignment, complex tables, or source HTML/figures/Mermaid that require conversion. Before writing storage, read `references/storage-format.md`.

For an existing page, always call `confluence_get_page` first. Keep the Storage path if the content contains macros, page attachments, layouts, or fixed-size images that Markdown cannot preserve. If converting a Markdown page to Storage is necessary, explain that the entire body must change format and obtain confirmation before doing so.

## Prepare source content

### Titles and headings

- For a full Markdown sync, use a single first non-empty `# Heading` as the page title and remove it from the body.
- Do not put an H1 or a duplicate page title in the body. Body content starts at H2.
- Keep the existing title for partial updates. For an explicitly requested full-document replacement, update the title from the source H1.
- If the source has additional H1 headings, ask before changing them; for an explicitly authorized full sync, demote them to H2.

### Links, images, and source HTML

- Inspect every relative link and local resource reference before writing.
- Keep HTTP(S) links unchanged.
- Upload locally referenced images and non-Markdown files (for example JSON, XLSX, TMPL, PDF, and ZIP) to the target page, then replace their references with page-attachment references.
- Render a local Markdown link as a code-styled repository path unless the user provides a mapping to a Confluence page.
- Report unresolved local references before writing; never leave them as clickable repository paths.
- Convert raw HTML rather than writing it as-is. Convert `figure` image content into an uploaded attachment image and a separate caption. Omit non-display `template` content by default; use a code macro only when the user asks to retain it. Prefer an existing SVG or PNG for Mermaid; otherwise retain its source in a code macro or ask whether to render it.
- Never put base64 content or `data:` URIs in page bodies, comments, or image references.

## Update and attachment workflow

`confluence_update_page` replaces the full body.

1. Read the full current page and merge the requested change locally, preserving unrelated content. Ensure the target content is not already present.
2. Unless the user explicitly requests a full replacement, show a change summary and obtain confirmation when the change would replace or remove roughly half of the current body.
3. For local resources, list existing attachments, upload missing or changed files, then update the body. Do not create an attachment reference until the upload succeeds.
4. For a new page, create the page first, upload its local resources, then update the body with their references.
5. On a version conflict, read again, re-merge, and retry once. Report a second conflict without further retries.

Reuse an existing attachment only when its name and size match and its content is not expected to change. Otherwise upload the source basename and keep older attachments; this skill does not delete attachments.

## Attachment staging

When the MCP server cannot read the source file directly:

1. Copy each authorized file into `/Users/v1fanchao/data/mcp-atlassian/<page-id>/`. Use real copies, not symlinks.
2. Upload it with `/app/uploads/<page-id>/<filename>`.
3. Keep its source basename unless the body reference is changed accordingly.

Use `content_base64` only when this staging bridge cannot be used. Never pass a host path such as `/Users/...` to an attachment tool.

## Layout and request defaults

Use semantic headings, short paragraphs, lists, code blocks, and tables only when they improve scanning.

- Default page width: `"default"`. Use full width only when the user explicitly asks, or when the existing page is full width and the task does not alter its layout.
- Default table layout: `"default"`. Do not widen a page solely to repair a table.
- In Storage, use `class="confluenceTable"`, `data-layout="default"`, and `data-table-width="760"`. Let long content wrap at the cell level. Use wide or full-width tables only under the same explicit conditions as page width.
- Prefer smaller tables, sections, lists, or expand macros when a table has more than four columns or dense prose.
- Leave `include_content`, `is_minor_edit`, `version_comment`, `enable_heading_anchors`, `emoji`, and `parent_id` unset unless the user gives a reason. Use `is_minor_edit: true` only when they explicitly request no subscriber notification. Set `version_comment` only for traceable full syncs. Never guess a parent.
- Pass inline `content` up to about 20 KB; use `content_file` under `/app/uploads/...` for larger bodies.

## Validate and report

After every write:

1. Read the page back and confirm its metadata title, the intended structure, and the absence of H1 or duplicate title text in the body.
2. List attachments and verify every `ri:attachment` reference resolves to an attachment on this page.
3. Verify that no unresolved local paths, raw source HTML, base64, or `data:` URIs remain.
4. For Storage tables, verify `confluenceTable`, default layout attributes, and no `width: 100%; table-layout: fixed;` workaround.
5. Report the page URL, title change, layout/content changes, uploaded or reused attachments, any resources not uploaded and why, and the read-back result.

Use the same format-selection rules for `confluence_add_comment`; Markdown is the default.
