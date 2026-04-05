# D2 Diagram Conversion & Build Verification

## Summary

Converted all ASCII art diagrams in the mdbook to D2 diagram language and verified successful rendering via `mdbook build`.

## Changes Made

### Configuration
- **`book.toml`**: Added `[preprocessor.d2]` with `layout = "elk"`, `inline = false`, `output-dir = "d2"`
- **`.github/workflows/docs.yml`**: Added D2 and mdbook-d2 installation steps
- **`theme/mathjax.js`**: Client-side MathJax 3 loader
- **`theme/diagrams.js`**: Makes D2 SVG images clickable

### Diagrams Converted (7 diagrams across 6 files)

| File | Diagram | SVG Output |
|---|---|---|
| `introduction.md` | Template lifecycle flow | `01.svg` |
| `architecture/overview.md` | Component topology | `1.1.svg` |
| `architecture/overview.md` | Entity hierarchy | `1.2.svg` |
| `architecture/overview.md` | Data flow | `1.3.svg` |
| `architecture/visual-canvas.md` | Layout algorithm | `2.1.svg` |
| `architecture/telemetry.md` | Telemetry architecture | `3.1.svg` |
| `architecture/grpc-service.md` | Frontend proxy pattern | `4.1.svg` |
| `templates/concepts.md` | Entity graph | `5.1.svg` |

### Intentionally Not Converted
- `architecture/overview.md`: `studio-data/` directory tree — kept as plain text since it's a file listing, not a relational diagram

## Build Verification

```
$ mdbook build docs/current
 INFO Book building has started
 INFO Running the html backend
 INFO HTML book written to docs/current/book
```

All 8 SVGs generated in `book/d2/`, all image references resolve correctly in HTML output.

## Tools Available via Nix

- `d2` v0.7.1
- `mdbook` v0.5.2
- `mdbook-d2` v0.3.8
