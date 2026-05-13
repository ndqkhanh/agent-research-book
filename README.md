# Agent Research Book

[![Pages](https://img.shields.io/badge/read-online-blue)](https://ndqkhanh.github.io/agent-research-book/)
[![License: CC BY 4.0](https://img.shields.io/badge/license-CC%20BY%204.0-lightgrey)](LICENSE)

**321 deep-dives on AI-agent harness engineering** — from agent loops and skills to the May-2026 frontier of reasoning models, agentic RL, persistent memory, multi-agent coordination, scaling laws, agent security, and production operations.

- 📖 **Read online**: <https://ndqkhanh.github.io/agent-research-book/>
- 📚 **Interactive book** (single self-contained HTML): [`docs/index.html`](docs/index.html)
- 🗂️ **Full chapter index**: [`docs/README.md`](docs/README.md)
- 🆕 **What's new — May 2026**: chapters 100–321 cover memory systems (MemTier, Mnema, Mem0), skill discovery & self-evolution (AutoSkill, EvoSkill, SkillRL), multi-agent coordination & multi-hop reasoning, agent scaling laws (test-time compute, verifier/best-of-N, parallel scaling), production runtimes (LangGraph, AutoGen v0.4, Google ADK, OpenAI Agents SDK), agent security (prompt injection, supply chain, isolation), the Argus Omega skill-router design, deep research agent surveys, and 2026 synthesis capstones.

## Repository layout

```
agent-research-book/
├── docs/                     # 321 chapter markdown files + interactive book
│   ├── 01-agent-loop-architecture.md  ...  321-spec-driven-development-bmad-ai-2026-deep-synthesis.md
│   ├── index.html            # The Harness Engineering Book (build_book.py output)
│   ├── build_book.py         # Regenerates index.html from the markdown chapters
│   └── README.md             # Categorised chapter index
├── harness-engineering/      # Synthesis essays and cross-paper notes
└── index.md                  # GitHub Pages homepage (Jekyll)
```

## Reading the book

Three options, ordered by polish:

1. **Online (Pages site)** — navigable, searchable, themed: <https://ndqkhanh.github.io/agent-research-book/>.
2. **Interactive book** — open `docs/index.html` directly in a browser, or run `python3 -m http.server` from `docs/` and visit `http://localhost:8000`. Includes plain-English mode, glossary tooltips, Mermaid diagrams, line-by-line code walkthroughs, search, dark mode, bookmarks.
3. **Raw markdown** — read individual files in `docs/` on GitHub. Cross-references between chapters use relative links (`[chapter](14-reflexion.md)`).

## Building the interactive book locally

```bash
cd docs
python3 build_book.py
# → writes docs/index.html (~1.6 MB self-contained)
```

`build_book.py` requires only the Python standard library. It auto-discovers any markdown file matching `[0-9]*.md` in the same directory and assembles them into the `index.html` book using the part/category map at the top of the script.

## Contributing

This is a personal research notebook published as a public reference. Issues for fact-checking and broken links are welcome. PRs are not actively solicited but will be considered.

## Citations

Every chapter cites primary sources at the bottom (arXiv IDs, GitHub URLs, official blogs, retrieval dates). When a claim is from secondary reporting, that is called out explicitly. If you spot an inaccuracy, please open an issue.

## License

- **Prose** (markdown chapters, the interactive book, this README): [Creative Commons Attribution 4.0 International (CC BY 4.0)](LICENSE).
- **`docs/build_book.py`**: MIT (see header inside the file).
- **Cited papers and quotes**: remain under their original copyright; this work links to and summarises them; full text is not reproduced.

## Acknowledgements

Built on the shoulders of the open-source agent community — Anthropic, OpenAI, Microsoft Research, Google DeepMind, Hugging Face, the LangChain team, the All-Hands-AI team, the AutoGen community, and many others cited inside individual chapters.
