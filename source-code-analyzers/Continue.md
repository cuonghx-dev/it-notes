# Continue

Continue is an open-source AI coding assistant that brings LLM capabilities directly into your IDE. Let me walk you through its architecture and structure.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        IDE Layer                            │
│  ┌──────────────────┐    ┌──────────────────────────────┐  │
│  │  VS Code Extension │    │  JetBrains Plugin (Kotlin)  │  │
│  └────────┬─────────┘    └─────────────┬────────────────┘  │
└───────────┼────────────────────────────┼────────────────────┘
            │                            │
            ▼                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      Core Engine                            │
│  ┌─────────────┐ ┌──────────────┐ ┌───────────────────────┐│
│  │ LLM Providers│ │Context System│ │ Indexing & Embeddings ││
│  └─────────────┘ └──────────────┘ └───────────────────────┘│
└─────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│                        GUI Layer                            │
│              React-based Chat & Settings UI                 │
└─────────────────────────────────────────────────────────────┘
```

## Source Code Structure

The repository is a **monorepo** organized as follows:

```
continue/
├── core/                  # The brain - shared business logic
├── extensions/
│   └── vscode/            # VS Code extension
├── plugins/
│   └── intellij/          # JetBrains plugin (Kotlin)
├── gui/                   # React frontend (webview UI)
├── binary/                # Standalone binary for the core
├── packages/              # Shared utilities and configs
└── docs/                  # Documentation site
```

---

