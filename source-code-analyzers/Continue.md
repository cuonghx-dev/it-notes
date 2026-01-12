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

# Core

The `core/` directory is the platform-agnostic brain of Continue. It contains all the business logic that powers the AI assistant, independent of any specific IDE.

## Core Directory Structure

```
core/
├── index.ts               # Main entry point & Core class
├── llm/                   # LLM provider system
├── context/               # Context gathering system
├── autocomplete/          # Tab autocomplete engine
├── indexing/              # Code indexing & embeddings
├── commands/              # Slash commands
├── config/                # Configuration management
├── protocol/              # Message protocol definitions
├── edit/                  # Code editing logic
├── promptFiles/           # Prompt template system
├── util/                  # Shared utilities
└── vendor/                # Vendored dependencies
```

---

## 1. LLM Module (`/core/llm`)

Handles all interactions with language models.

### Structure

```
llm/
├── index.ts               # BaseLLM abstract class
├── llms/                  # Provider implementations
│   ├── Anthropic.ts       # Claude models
│   ├── OpenAI.ts          # GPT models
│   ├── Ollama.ts          # Local Ollama models
│   ├── Gemini.ts          # Google Gemini
│   ├── Mistral.ts         # Mistral AI
│   ├── Azure.ts           # Azure OpenAI
│   ├── Bedrock.ts         # AWS Bedrock
│   ├── LMStudio.ts        # LM Studio local
│   ├── DeepSeek.ts        # DeepSeek models
│   └── ...                # Many more providers
├── templates/             # Prompt templates per model
├── autodetect.ts          # Auto-detect model capabilities
├── countTokens.ts         # Token counting utilities
└── stream.ts              # Streaming response handling
```

### Key Abstractions

```typescript
// core/llm/index.ts
abstract class BaseLLM {
  // Core properties
  model: string;
  contextLength: number;
  maxTokens: number;

  // Main methods
  abstract _streamChat(
    messages: ChatMessage[],
    options: LLMOptions
  ): AsyncGenerator<ChatMessage>;
  abstract _streamComplete(
    prompt: string,
    options: LLMOptions
  ): AsyncGenerator<string>;

  // Shared utilities
  countTokens(text: string): number;
  truncateMessages(messages: ChatMessage[]): ChatMessage[];
}
```

### Provider Example (Simplified)

```typescript
// core/llm/llms/Anthropic.ts
class Anthropic extends BaseLLM {
  static providerName = "anthropic";

  async *_streamChat(messages, options) {
    const response = await fetch("https://api.anthropic.com/v1/messages", {
      method: "POST",
      headers: { "x-api-key": this.apiKey },
      body: JSON.stringify({
        model: this.model,
        messages: this.convertMessages(messages),
        stream: true,
      }),
    });

    // Handle SSE streaming
    for await (const chunk of this.parseSSE(response)) {
      yield { role: "assistant", content: chunk.delta.text };
    }
  }
}
```

---

## 2. Context Module (`/core/context`)

Gathers relevant context to include in prompts.

### Structure

```
context/
├── index.ts               # IContextProvider interface
├── providers/             # Built-in context providers
│   ├── FileContextProvider.ts      # @file - specific files
│   ├── CodebaseContextProvider.ts  # @codebase - semantic search
│   ├── FolderContextProvider.ts    # @folder - directory contents
│   ├── DocsContextProvider.ts      # @docs - documentation
│   ├── WebContextProvider.ts       # @web - web search
│   ├── GitDiffContextProvider.ts   # @diff - git changes
│   ├── TerminalContextProvider.ts  # @terminal - terminal output
│   ├── ProblemsContextProvider.ts  # @problems - IDE diagnostics
│   ├── SearchContextProvider.ts    # @search - text search
│   └── ...
├── retrieval/             # Retrieval strategies
└── reranking/             # Result reranking
```

### Key Interface

```typescript
// core/context/index.ts
interface IContextProvider {
  // Metadata
  title: string;
  displayTitle: string;
  description: string;
  type: ContextProviderType; // "normal" | "query" | "submenu"

  // Core method - returns context items
  getContextItems(
    query: string,
    extras: ContextProviderExtras
  ): Promise<ContextItem[]>;

  // Optional: load indexed data
  loadSubmenuItems?(args: LoadSubmenuItemsArgs): Promise<ContextSubmenuItem[]>;
}

interface ContextItem {
  name: string;
  description: string;
  content: string; // The actual content to include
  uri?: string; // Source location
  editing?: boolean; // For edit context
}
```

### Provider Example

```typescript
// core/context/providers/FileContextProvider.ts
class FileContextProvider implements IContextProvider {
  title = "file";
  displayTitle = "Files";
  description = "Type to search for files";
  type = "submenu";

  async getContextItems(query: string, extras: ContextProviderExtras) {
    const content = await extras.ide.readFile(query);
    return [
      {
        name: query.split("/").pop(),
        description: query,
        content: `\`\`\`${query}\n${content}\n\`\`\``,
        uri: query,
      },
    ];
  }

  async loadSubmenuItems({ ide }) {
    const files = await ide.listWorkspaceContents();
    return files.map((file) => ({
      id: file,
      title: file.split("/").pop(),
      description: file,
    }));
  }
}
```

---

## 3. Autocomplete Module (`/core/autocomplete`)

Powers the tab completion feature.

### Structure

```
autocomplete/
├── completionProvider.ts  # Main completion logic
├── templates/             # Prompt templates (FIM format)
│   ├── index.ts
│   └── templates.ts       # Fill-in-middle templates
├── filtering/             # Filter/rank completions
│   ├── index.ts
│   └── filters.ts
├── charStreams.ts         # Character-level streaming
├── cache.ts               # Completion caching
├── ranking.ts             # Completion ranking
└── recentlyEdited.ts      # Track recent edits for context
```

### Core Flow

```typescript
// core/autocomplete/completionProvider.ts
async function getCompletions(
  document: AutocompleteInput,
  llm: BaseLLM,
  options: AutocompleteOptions
): Promise<AutocompleteOutcome> {
  // 1. Build context window
  const { prefix, suffix } = getContextAroundCursor(document);

  // 2. Add relevant snippets from recent files
  const snippets = await getRelevantSnippets(document, options);

  // 3. Format as Fill-in-Middle prompt
  const prompt = formatFIMPrompt(prefix, suffix, snippets, llm.model);

  // 4. Get completion from LLM
  const completion = await llm.complete(prompt, {
    maxTokens: options.maxTokens,
    stop: options.stopTokens,
  });

  // 5. Post-process and filter
  const filtered = filterCompletion(completion, document);

  return { completion: filtered };
}
```

### Fill-in-Middle (FIM) Template

```typescript
// Different models use different FIM formats
const templates = {
  starcoder: {
    prefix: "<fim_prefix>",
    suffix: "<fim_suffix>",
    middle: "<fim_middle>",
  },
  codellama: {
    prefix: "<PRE>",
    suffix: " <SUF>",
    middle: " <MID>",
  },
  deepseek: {
    prefix: "<｜fim▁begin｜>",
    suffix: "<｜fim▁hole｜>",
    middle: "<｜fim▁end｜>",
  },
};
```

---

## 4. Indexing Module (`/core/indexing`)

Manages code indexing for semantic search (@codebase).

### Structure

```
indexing/
├── CodebaseIndexer.ts     # Main indexer orchestration
├── chunk/                 # Code chunking strategies
│   ├── ChunkCodebaseIndex.ts
│   ├── chunk.ts           # Chunking algorithms
│   └── code.ts            # Language-aware chunking
├── embeddings/            # Embedding providers
│   ├── index.ts           # BaseEmbeddingsProvider
│   ├── OpenAIEmbeddings.ts
│   ├── OllamaEmbeddings.ts
│   ├── TransformersJsEmbeddings.ts  # Local embeddings
│   └── ...
├── FullTextSearchCodebaseIndex.ts  # Text search index
├── LanceDbIndex.ts        # Vector database
├── refreshIndex.ts        # Incremental updates
└── walkDir.ts             # Directory traversal
```

### Indexing Pipeline

```typescript
// Simplified indexing flow
class CodebaseIndexer {
  async index(workspaceDir: string) {
    // 1. Walk directory and collect files
    const files = await walkDir(workspaceDir, this.ignorePatterns);

    // 2. Chunk each file
    const chunks: Chunk[] = [];
    for (const file of files) {
      const content = await readFile(file);
      const fileChunks = chunkCode(content, file, {
        maxChunkSize: 512,
        overlap: 50,
      });
      chunks.push(...fileChunks);
    }

    // 3. Generate embeddings
    const embeddings = await this.embeddingsProvider.embed(
      chunks.map((c) => c.content)
    );

    // 4. Store in vector database
    await this.vectorDb.insert(
      chunks.map((chunk, i) => ({
        ...chunk,
        embedding: embeddings[i],
      }))
    );
  }

  async retrieve(query: string, topK: number = 10) {
    const queryEmbedding = await this.embeddingsProvider.embed([query]);
    return this.vectorDb.search(queryEmbedding[0], topK);
  }
}
```

### Chunking Strategy

```typescript
// core/indexing/chunk/code.ts
function chunkCode(
  content: string,
  filepath: string,
  options: ChunkOptions
): Chunk[] {
  const language = detectLanguage(filepath);

  // Use tree-sitter for language-aware chunking
  const tree = parser.parse(content);

  // Chunk by logical units: functions, classes, etc.
  const nodes = extractSemanticNodes(tree, language);

  return nodes.map((node) => ({
    content: node.text,
    startLine: node.startPosition.row,
    endLine: node.endPosition.row,
    filepath,
    language,
  }));
}
```

---

## 5. Commands Module (`/core/commands`)

Implements slash commands like `/edit`, `/comment`, `/share`.

### Structure

```
commands/
├── index.ts               # SlashCommand interface
├── slash/                 # Built-in slash commands
│   ├── edit.ts            # /edit - edit code
│   ├── comment.ts         # /comment - add comments
│   ├── share.ts           # /share - share conversation
│   ├── cmd.ts             # /cmd - run terminal command
│   ├── commit.ts          # /commit - generate commit msg
│   ├── review.ts          # /review - code review
│   └── ...
└── util.ts
```

### Interface

```typescript
// core/commands/index.ts
interface SlashCommand {
  name: string;
  description: string;

  run(context: SlashCommandContext): AsyncGenerator<string>;
}

interface SlashCommandContext {
  llm: BaseLLM;
  input: string; // User input after command
  history: ChatMessage[]; // Conversation history
  ide: IDE; // IDE access
  selectedCode: RangeInFile[]; // Currently selected code
  config: ContinueConfig;
}
```

### Command Example

```typescript
// core/commands/slash/comment.ts
const CommentCommand: SlashCommand = {
  name: "comment",
  description: "Add comments to the selected code",

  async *run(context) {
    const { llm, selectedCode, ide } = context;

    if (selectedCode.length === 0) {
      yield "Please select some code to comment.";
      return;
    }

    const code = await ide.readRangeInFile(selectedCode[0]);

    const prompt = `Add helpful comments to this code:\n\n${code}`;

    for await (const chunk of llm.streamChat([
      { role: "user", content: prompt },
    ])) {
      yield chunk.content;
    }
  },
};
```

---

## 6. Config Module (`/core/config`)

Handles loading and validating configuration.

### Structure

```
config/
├── index.ts               # Main config loading
├── types.ts               # TypeScript types for config
├── load.ts                # Load from file system
├── validation.ts          # Config validation
├── default.ts             # Default configuration
├── migrate.ts             # Config migration (versions)
└── profile/               # Profile management
    ├── index.ts
    └── doLoadConfig.ts
```

### Config Types

```typescript
// core/config/types.ts (simplified)
interface ContinueConfig {
  // Models
  models: ModelConfig[];
  tabAutocompleteModel?: ModelConfig;
  embeddingsProvider?: EmbeddingsProviderConfig;
  reranker?: RerankerConfig;

  // Features
  contextProviders?: ContextProviderConfig[];
  slashCommands?: SlashCommandConfig[];

  // Behavior
  systemMessage?: string;
  completionOptions?: CompletionOptions;
  requestOptions?: RequestOptions;

  // UI
  ui?: UIConfig;
}

interface ModelConfig {
  provider: string; // "anthropic", "openai", etc.
  model: string; // "claude-3-opus", "gpt-4", etc.
  apiKey?: string;
  apiBase?: string;
  contextLength?: number;
  // ...provider-specific options
}
```

---

## 7. Protocol Module (`/core/protocol`)

Defines message types for communication between components.

### Structure

```
protocol/
├── index.ts               # Main protocol exports
├── messenger/             # Messaging implementation
│   ├── index.ts
│   └── messageIde.ts
├── ideWebview.ts          # IDE <-> Webview messages
└── core.ts                # Core protocol types
```

### Message Types

```typescript
// core/protocol/ideWebview.ts (simplified)
type ToWebviewProtocol = {
  // Config updates
  configUpdate: [config: ContinueConfig];

  // Chat messages
  newSessionWithPrompt: [prompt: string];
  userInput: [input: string];

  // Streaming
  streamUpdate: [content: string];
  streamComplete: [];

  // Context
  highlightedCode: [code: RangeInFileWithContents];

  // State
  setInactive: [];
  indexProgress: [progress: IndexingProgress];
};

type FromWebviewProtocol = {
  // User actions
  newSession: [];
  input: [input: string];
  abort: [];

  // Settings
  selectModel: [model: string];
  toggleDevTools: [];

  // History
  loadSession: [sessionId: string];
  deleteSession: [sessionId: string];
};
```

---

## 8. Edit Module (`/core/edit`)

Handles code editing and diff application.

### Structure

```
edit/
├── index.ts               # Main edit logic
├── lazy/                  # Lazy diff application
│   ├── deterministic.ts   # Rule-based diff
│   └── model.ts           # LLM-based diff
├── streamDiff.ts          # Stream diff chunks
└── util.ts
```

---

## Module Interaction Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         Core Entry                              │
│                        (index.ts)                               │
└─────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│    Config     │    │   Protocol    │    │     Edit      │
│   (config/)   │    │  (protocol/)  │    │    (edit/)    │
└───────┬───────┘    └───────────────┘    └───────────────┘
        │
        │ Instantiates
        ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│      LLM      │◄───│   Commands    │    │  Autocomplete │
│    (llm/)     │    │  (commands/)  │    │(autocomplete/)│
└───────┬───────┘    └───────┬───────┘    └───────┬───────┘
        │                    │                    │
        │         ┌──────────┴──────────┐        │
        │         ▼                     ▼        │
        │  ┌───────────────┐    ┌───────────────┐│
        └─►│    Context    │    │   Indexing    │◄┘
           │  (context/)   │◄───│  (indexing/)  │
           └───────────────┘    └───────────────┘
```

---

## Key Entry Point: Core Class

```typescript
// core/index.ts (simplified)
class Core {
  config: ContinueConfig;
  llm: BaseLLM;
  contextProviders: Map<string, IContextProvider>;
  slashCommands: Map<string, SlashCommand>;
  codebaseIndexer: CodebaseIndexer;

  constructor(ide: IDE, config: ContinueConfig) {
    this.config = config;
    this.llm = this.initLLM(config.models[0]);
    this.contextProviders = this.initContextProviders(config);
    this.slashCommands = this.initSlashCommands(config);
    this.codebaseIndexer = new CodebaseIndexer(config, ide);
  }

  async *chat(messages: ChatMessage[], options: ChatOptions) {
    // 1. Gather context
    const context = await this.gatherContext(messages, options);

    // 2. Build system prompt
    const systemPrompt = this.buildSystemPrompt(context);

    // 3. Stream response
    for await (const chunk of this.llm.streamChat([
      { role: "system", content: systemPrompt },
      ...messages,
    ])) {
      yield chunk;
    }
  }

  async getCompletions(input: AutocompleteInput) {
    return getCompletions(input, this.llm, this.config);
  }
}
```

---
