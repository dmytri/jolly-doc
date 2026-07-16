# Diagram Format Comparison

How to view each format in this terminal (xterm-256color, no root).

| Format | File | View now | To view richly |
|---|---|---|---|
| **Svgbob** | `svgbob/jolly-context.bob` | `cat svgbob/jolly-context.bob` | Renders perfectly as-is — pure ASCII |
| **Unicode boxes** | `unicode/jolly-context.md` | `glow unicode/jolly-context.md` | Renders well; glow may wrap long lines |
| **Graphviz DOT** | `dot/jolly-context.dot` | `cat dot/jolly-context.dot` | `sudo apt install graphviz && dot -Tascii dot/jolly-context.dot` |
| **D2** | `d2/jolly-context.d2` | `cat d2/jolly-context.d2` | `npm i -g @d2lang/core && d2 ... && sudo apt install chafa && chafa out.svg` |
| **Nomnoml** | `nomnoml/jolly-context.nomnoml` | `npx nomnoml jolly-context.nomnoml` | `npx nomnoml file.nomnoml out.svg && chafa out.svg` |

## What each viewer gives you

| Viewer | ASCII | Unicode | Images (SVG/PNG) | Mermaid |
|---|---|---|---|---|
| `cat` | ✓ | ✓ | ✗ | ✗ (raw source) |
| `glow` | ✓ | ✓ (may wrap) | ✗ | ✗ (no render) |
| `chafa` | ✗ | ✗ | ✓ (terminal blocks) | ✗ |
| `leaf` | ✓ | ✓ | ✓ (if Sixel/Kitty) | ✓ (client-side JS) |
| `flow` | ✓ | ✓ | partial | ✓ (chartbrew API) |

## My opinions

**Best pure-text terminal diagram:** `svgbob` — readable as-is with `cat`, no tools, no wrap issues. Autolayout is manual though.

**Best auto-layout + terminal rendering combo:** `graphviz dot -Tascii` — write once, auto-layout, commit the ASCII output. Best of both worlds. Needs `apt install graphviz`.

**Best colour rendering:** `d2` + `chafa` — but you need a terminal with Sixel/Kitty support for good results.

**Svgbob** is the clear winner for "view in any terminal with zero tooling." If you want auto-layout, **Graphviz DOT** + committed `-Tascii` output is the strongest combo: auto-layout text that renders everywhere.
