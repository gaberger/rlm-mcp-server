# RLM MCP Server

An MCP (Model Context Protocol) server implementing **Recursive Language Model (RLM)** primitives for processing inputs that exceed practical context limits.

Based on the technique from: [Recursive Language Models](https://arxiv.org/html/2512.24601v1)

## Overview

RLM enables LLMs to process inputs far beyond their context windows through:
1. **Decomposition** - Breaking large inputs into manageable chunks
2. **Recursive Analysis** - Processing each chunk independently
3. **Aggregation** - Combining results into coherent output

This MCP server provides the control primitives to implement this pattern.

## Installation

```bash
cd rlm-mcp-server
npm install
npm run build
```

## Configuration

### Claude Code

Add to your Claude Code MCP settings (`~/.claude/settings.json` or project `.claude/settings.json`):

```json
{
  "mcpServers": {
    "rlm": {
      "command": "node",
      "args": ["/path/to/rlm-mcp-server/dist/index.js"]
    }
  }
}
```

### Claude Desktop

Add to `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "rlm": {
      "command": "node",
      "args": ["/path/to/rlm-mcp-server/dist/index.js"]
    }
  }
}
```

## Tools

### `rlm_init`

Initialize an RLM session with large input content.

```
Input:
  - input: string (required) - The large content to process
  - description: string (optional) - Description of the content

Output:
  - session_id: Unique session identifier
  - input_stats: Length, lines, paragraphs, estimated tokens
  - recommended_strategies: Suggested chunking approaches
```

### `rlm_chunk`

Chunk the input using a specified strategy.

```
Input:
  - strategy: "lines" | "paragraphs" | "tokens" | "semantic" | "sliding" | "custom"
  - size: number (optional) - Chunk size parameter
  - overlap: number (optional) - Overlap for sliding window
  - pattern: string (optional) - Regex for custom strategy

Strategies:
  - lines: Split by line count (default: 100 lines)
  - paragraphs: Split by double newlines (default: 5 paragraphs)
  - tokens: Split by estimated tokens (default: 2000 tokens)
  - semantic: Split by section headers (markdown #, numbered sections)
  - sliding: Sliding window with overlap
  - custom: Split by regex pattern
```

### `rlm_get_chunk`

Retrieve content of a specific chunk.

```
Input:
  - chunk_id: number - The chunk ID (0-indexed)

Output:
  - chunk_id, total_chunks, length, content
```

### `rlm_query_chunk`

Get chunk content with an analysis prompt prepended.

```
Input:
  - chunk_id: number - The chunk ID
  - prompt: string - Analysis question/prompt

Output:
  - formatted_query: Ready-to-analyze text with prompt and chunk content
```

### `rlm_store_result`

Store analysis results for a chunk.

```
Input:
  - chunk_id: number - The chunk this result is for
  - result: string - Analysis findings

Output:
  - progress: "N/M chunks analyzed"
  - remaining: List of unanalyzed chunk IDs
```

### `rlm_get_results`

Get all stored results for aggregation.

```
Input:
  - format: "list" | "combined" | "structured" (default: structured)

Output varies by format:
  - list: Array of result strings
  - combined: Single string with results separated by ---
  - structured: Full metadata with chunk IDs
```

### `rlm_status`

Get current session status.

```
Output:
  - session_id, created_at, input_length
  - strategy used
  - chunks: total, analyzed, remaining
  - chunk_details: Preview of each chunk
```

### `rlm_clear`

Clear the current session and free memory.

## Usage Pattern

The typical RLM workflow:

```
1. rlm_init     → Initialize with large input
2. rlm_chunk    → Choose strategy and create chunks
3. Loop:
   a. rlm_get_chunk or rlm_query_chunk → Get chunk content
   b. [Analyze the chunk using LLM reasoning]
   c. rlm_store_result → Store findings
4. rlm_get_results → Aggregate all results
5. [Synthesize final output]
6. rlm_clear    → Clean up
```

## Example Session

### Analyzing a Large Log File

```
User: Analyze this 50MB log file for error patterns

LLM Actions:

1. Call rlm_init with the log file content
   → Returns: 1.2M chars, 45K lines, recommends "lines" or "sliding" strategy

2. Call rlm_chunk with strategy="lines", size=500
   → Returns: 90 chunks created

3. For each chunk (0-89):
   a. Call rlm_query_chunk with chunk_id=N, prompt="Find ERROR and WARN entries, note patterns"
   b. Analyze the returned content
   c. Call rlm_store_result with findings

4. Call rlm_get_results with format="structured"
   → Returns all 90 chunk analyses

5. Synthesize: "Found 3 main error patterns:
   - Database connection timeouts (chunks 12, 45, 67)
   - Memory pressure warnings (chunks 30-40)
   - API rate limiting (chunks 78-85)"

6. Call rlm_clear
```

### Comparing Multiple Documents

```
User: Compare these 10 API spec documents for inconsistencies

LLM Actions:

1. Concatenate docs with markers, call rlm_init
2. Call rlm_chunk with strategy="custom", pattern="^## Document \\d+"
   → Creates 10 chunks (one per doc)

3. First pass - extract schemas from each:
   For each chunk: rlm_query_chunk → analyze → rlm_store_result

4. Get results, identify schema differences

5. Second pass (if needed) - deeper comparison on specific sections
```

## Chunking Strategy Guide

| Input Type | Recommended Strategy | Size |
|------------|---------------------|------|
| Log files | `lines` or `sliding` | 200-500 lines |
| Documentation | `semantic` or `paragraphs` | Auto or 5-10 |
| Source code | `semantic` | Auto |
| JSON/Data | `tokens` | 2000-4000 |
| Mixed content | `sliding` | 5000 chars, 500 overlap |
| Multiple docs | `custom` with doc markers | N/A |

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Claude / LLM                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  1. Decide to use RLM for large input           │   │
│  │  2. Call MCP tools to chunk and manage state    │   │
│  │  3. Analyze each chunk using own reasoning      │   │
│  │  4. Store results via MCP                       │   │
│  │  5. Aggregate and synthesize                    │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼ MCP Protocol
┌─────────────────────────────────────────────────────────┐
│                  RLM MCP Server                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Session    │  │   Chunking   │  │   Results    │  │
│  │   Storage    │  │   Engine     │  │   Store      │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
```

## Development

```bash
# Run in development mode
npm run dev

# Type check
npm run typecheck

# Build for production
npm run build
```

## License

MIT# rlm-mcp-server
