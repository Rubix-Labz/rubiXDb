# Building the RUBIXDB book

This directory contains the source-controlled technical monograph:

- `RUBIXDB_BOOK.md` — complete manuscript.
- `TECHNICAL_INVENTORY.md` — source/test/configuration evidence map.
- `BOOK_SPEC.md` — editorial structure and status rules.
- `diagrams/*.mmd` — maintainable Mermaid-compatible diagram sources.
- `build_book.py` — deterministic local HTML/PDF builder.
- `review_report.md` — generated QA record after a review run.

## Build

From the repository root:

```powershell
python docs/book/build_book.py
```

The script writes:

```text
docs/book/out/RUBIXDB_BOOK.html
docs/book/out/RUBIXDB_BOOK.pdf
docs/book/out/manifest.json
docs/book/review_report.md
```

The HTML edition embeds a restrained stylesheet and leaves Mermaid diagrams as maintainable `<pre class="mermaid">` blocks. A publishing site can render those blocks with its approved Mermaid runtime. The local environment used for the first edition did not contain Pandoc, Typst, TeX/Tectonic, Mermaid CLI, Graphviz, Chromium, or wkhtmltopdf. ReportLab was available, so the builder creates a real paginated PDF with a cover, table of contents, running headers/footers, headings, code blocks, tables, and diagram-source panels. The PDF does not pretend to contain rendered Mermaid artwork when no Mermaid renderer is installed; the editable `.mmd` sources remain authoritative.

## Update procedure

1. Confirm the new Git revision and inspect changed source/configuration.
2. Update `TECHNICAL_INVENTORY.md` with evidence before changing prose.
3. Update source references and status labels in `RUBIXDB_BOOK.md`.
4. Add or revise `.mmd` diagrams only when the source architecture changes.
5. Run `python docs/book/build_book.py`.
6. Inspect the generated HTML and PDF, especially the cover, contents, code-heavy pages, tables, diagram panels, and final appendices.
7. Record test/build limitations in `review_report.md` and update the edition revision/date.

Do not use this book output to replace existing project documentation. It is a dedicated publication artifact and is intentionally versioned against a specific source revision.
