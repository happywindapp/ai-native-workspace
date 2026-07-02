# DOCX Export

Convert the generated Markdown documentation to `.docx` when the user requests a Word version.

## Preferred: pandoc

Check availability first:

```powershell
Get-Command pandoc -ErrorAction SilentlyContinue
```

If present:

```powershell
pandoc "README_ALL.md" -o "README_ALL.docx" --toc --toc-depth=3
```

Notes:
- `--toc` regenerates a native Word table of contents.
- Mermaid code blocks do NOT render as images in pandoc output — they appear as code text. To embed rendered diagrams, pre-render Mermaid to PNG/SVG (mermaid-cli `mmdc`) and reference the images in the Markdown before conversion, OR note in the doc that diagrams render in Markdown viewers.
- For a styled document, pass `--reference-doc=template.docx`.

## Mermaid pre-render (optional, for image-embedded DOCX)

```powershell
Get-Command mmdc -ErrorAction SilentlyContinue   # mermaid-cli
mmdc -i diagram.mmd -o diagram.png
```

## Fallback options

- If pandoc is unavailable: tell the user it is required (`choco install pandoc` / `winget install pandoc`) and offer to deliver Markdown only.
- Do NOT fabricate a `.docx` by renaming or hand-writing XML.
- The `media-processing` skill does not handle document conversion — pandoc is the tool.

## Output checklist

- [ ] `.md` validated and complete first
- [ ] pandoc availability confirmed
- [ ] TOC present in `.docx`
- [ ] Diagrams either pre-rendered as images or noted as Markdown-only
- [ ] File written next to the `.md` (or to the user-specified path)