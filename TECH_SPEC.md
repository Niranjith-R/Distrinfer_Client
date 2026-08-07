# Technical Specification: Hierarchical Prompt Tree View for `.txt` File Processing

> **Status:** Implemented. See `src/App.svelte`, `src/lib/TreeNode.svelte`, `src/lib/PromptDetail.svelte`.

## 1. Overview

This document describes the architecture and implementation plan for a feature that allows users to upload `.txt` files (each containing a collection of prompts), renders a hierarchical tree view of files and their prompts, and lets users click on individual prompts to dispatch asynchronous inference requests with real-time status feedback.

## 2. Goals

1. Accept only `.txt` file uploads (already constrained at the `FileUpload` level via `accept=".txt"`).
2. Parse each uploaded `.txt` file into individual prompts (one prompt per non-empty line).
3. Display a **tree view** where each **filename** is a parent node and each **prompt** (line) is a child node.
4. Allow clicking any node (file or prompt) to expand/collapse (for files) or trigger an inference request (for prompts).
5. Show real-time status (`Pending`, `Processing`, `Error`) and the final `infered` output when the request completes.

## 3. Data Model

### 3.1 PromptNode

A node in the tree view. The tree has a fixed two-level depth — **no recursive rendering** is needed.

```ts
interface PromptNode {
  /** Unique identifier: `file:<filename>` for file nodes, `file:<filename>:<lineIndex>` for prompts */
  id: string;
  /** The filename (file nodes) or the prompt text (prompt nodes) */
  label: string;
  /** 'file' | 'prompt' */
  type: 'file' | 'prompt';
  /** Child prompt nodes. Only populated for file nodes. */
  children: PromptNode[];
}
```

### 3.2 PromptStatus

Tracks the lifecycle of an individual prompt's inference request.

```ts
type PromptStatus = 'idle' | 'pending' | 'polling' | 'completed' | 'error';

interface PromptResult {
  /** Matches the `hash` returned by `POST /query` */
  promptId: string;
  status: PromptStatus;
  /** The inferred output, available when status === 'completed' */
  data: any | null;
  /** Error message, available when status === 'error' */
  error: string | null;
}
```

### 3.3 Application State

```ts
// Top-level state held in App.svelte
const treeNodes: PromptNode[]   // the tree structure shown in the sidebar
const promptResults: Record<string, PromptResult>  // keyed by prompt node id
```

## 4. Component Architecture

```
App.svelte
├── Sidebar (existing left panel)
│   ├── UploadedFiles (existing — file name list)
│   └── TreeNode (NEW — renders file → prompt list, no recursion)
│       ├── File node (parent)
│       │   └── Prompt nodes (children, no nesting)
│       └── PromptDetail (NEW — right-hand detail panel, triggered by clicking a prompt)
└── Main Content (existing right panel)
    ├── FileUpload (existing, already constrained to .txt)
    ├── PromptInput / QueryArea (existing)
    └── ResultDisplay (existing)
```

### 4.1 New Svelte Components

#### `src/lib/TreeNode.svelte`

Renders the flat two-level tree (file → prompts). **No recursion.**

**Props:**
- `node: PromptNode` (a file node, `type: 'file'`)
- `promptResults: Record<string, PromptResult>`
- `onSelect: (node: PromptNode) => void` — callback when a prompt is clicked

**Behavior:**
- **File node (`type === 'file'`):**
  - Renders a clickable header bar with a chevron toggle.
  - Clicking the chevron toggles `expanded` (local boolean state).
  - When expanded, shows all child prompt nodes beneath it.
- **Prompt node (`type === 'prompt'`):**
  - Indented beneath the file header.
  - Clicking dispatches `onSelect` with the prompt node (triggers inference).
  - Shows a status dot colored by `promptResults[node.id].status`:
    - `idle` → gray
    - `pending` / `polling` → yellow (with "Polling…" label)
    - `completed` → green
    - `error` → red

**Markup skeleton (flat, no recursion):**
```svelte
<div class="tree-node">
  <!-- File header -->
  <div class="file-header" onclick={toggleExpand}>
    <ChevronDown class="chevron" class:rotated={expanded} />
    <span>{node.label}</span>
  </div>

  <!-- Prompt children -->
  {#if expanded}
    <div class="prompt-children">
      {#each node.children as child (child.id)}
        <div class="prompt-item" onclick={() => onSelect(child)}>
          <span class="status-dot" class:status-pending={...} class:status-completed={...} class:status-error={...} />
          <span class="truncate">{child.label}</span>
        </div>
      {/each}
    </div>
  {/if}
</div>
```

#### `src/lib/PromptDetail.svelte`

Shows the detail view for the currently selected prompt node (right-side panel).

**Props:**
- `node: PromptNode | null` — the selected prompt node
- `result: PromptResult | null` — the inference result for that node

**Markup logic:**
```svelte
{#if !node}
  <p>Select a prompt to view details.</p>
{:else}
  <h3>Prompt</h3>
  <pre>{node.label}</pre>
  {#if result?.status === 'pending' || result?.status === 'polling'}
    <div class="status">⏳ Polling...</div>
  {:else if result?.status === 'error'}
    <div class="error">{result.error}</div>
  {:else if result?.status === 'completed'}
    <h4>Result</h4>
    <pre>{JSON.stringify(result.data, null, 2)}</pre>
  {/if}
{/if}
```

## 5. API Integration

The existing `sendPrompt` / `pollForResult` logic is refactored into a reusable, per-prompt function.

### 5.1 `runPrompt(promptText: string, nodeId: string)`

```ts
async function runPrompt(promptText, nodeId) {
  // Initialize status
  promptResults[nodeId] = {
    promptId: '',
    status: 'pending',
    data: null,
    error: null
  };

  try {
    // Step 1: POST the prompt
    const res = await fetch(`${API_BASE}/query`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ prompt: promptText })
    });
    const data = await res.json();
    const hash = data.hash;

    // Step 2: Poll for result
    promptResults[nodeId].promptId = hash;
    promptResults[nodeId].status = 'polling';

    pollForResult(hash, nodeId);  // non-blocking, updates state on completion
  } catch (err) {
    promptResults[nodeId].status = 'error';
    promptResults[nodeId].error = err.message;
  }
}
```

### 5.2 `pollForResult(hash: string, nodeId: string)`

Same polling logic as the existing `pollForResult`, but keyed by `nodeId` instead of global `result`/`loading` state.

```ts
function pollForResult(hash, nodeId) {
  const POLL_INTERVAL = 1500;
  const poll = async () => {
    try {
      const res = await fetch(`${API_BASE}/query/${hash}`);
      const data = await res.json();
      if (data.status === 'Pending' || data.status === 'Processing') {
        setTimeout(poll, POLL_INTERVAL);
      } else {
        promptResults[nodeId].status = 'completed';
        promptResults[nodeId].data = data;
      }
    } catch (err) {
      promptResults[nodeId].status = 'error';
      promptResults[nodeId].error = err.message;
    }
  };
  poll();
}
```

### 5.3 Concurrency Rules

- Multiple prompts can be queried **concurrently** — each has its own entry in `promptResults`.
- The existing single-prompt input box (bottom bar) and its result display remain, but the tree view becomes the primary interaction surface for file-based prompts.
- To avoid state collision, the tree-view path uses `promptResults` (a map), while the text-input path continues to use the existing `result`/`loading`/`polling` state.

## 6. File Parsing Logic

When files are accepted via `onFileAccept`, each file must be parsed.

### 6.1 Parsing Steps

1. Read file contents as text via `File.text()`.
2. Split into lines: `content.split('\n')`.
3. Trim each line; discard empty lines.
4. Each non-empty line becomes a prompt `PromptNode` child of the file node.

### 6.2 `buildTreeNode(file: File): PromptNode`

```ts
async function buildTreeNode(file) {
  const content = await file.text();
  const lines = content.split('\n')
    .map(l => l.trim())
    .filter(l => l.length > 0);

  return {
    id: `file:${file.name}`,
    label: file.name,
    type: 'file',
    children: lines.map((line, idx) => ({
      id: `file:${file.name}:${idx}`,
      label: line,
      type: 'prompt',
      children: []
    }))
  };
}
```

### 6.3 Integration with `handleFileAccept`

```ts
async function handleFileAccept(details) {
  for (const file of details.files) {
    const node = await buildTreeNode(file);
    treeNodes = [...treeNodes, node];
    // Also track in uploadedFiles sidebar if desired (existing behavior)
    if (!uploadedFiles.includes(file.name)) {
      uploadedFiles = [...uploadedFiles, file.name];
    }
  }
}
```

## 7. UI / UX Specification

### 7.1 Sidebar (Left Panel — `w-85`)

- **Top:** "DISTRINFER" title (monoton font, 5xl).
- **Below title:** New **TreeView** component taking the remaining height (`flex-1 overflow-y-auto`).
- **Bottom (existing API endpoint input):** Remains at the bottom, unchanged.

### 7.2 Tree View Styling

| Element            | Style                                              |
| ------------------ | -------------------------------------------------- |
| File node header   | `px-2 py-1 hover:bg-[#3a3f5c] cursor-pointer`      |
| Chevron icon       | Rotates 90° when file node is expanded             |
| Prompt node        | `px-4 py-1 text-sm` with left border indentation   |
| Status dot         | Inline `●` with color per `PromptStatus`           |
| Prompt node hover  | `bg-[#2d2f3d]`                                     |

### 7.3 Detail Panel (Right Side)

- Below the existing API endpoint input and above the prompt input bar, add a **PromptDetail** card.
- Shows the selected node's info and its current inference status/output.
- Falls back to the existing result display when a node is not selected.

## 8. State Management Decision

Given the project's current scale (single `App.svelte`, no state management library), all state (`treeNodes`, `promptResults`, `selectedNode`) is held as Svelte 5 `$state` reactive variables in `App.svelte` and passed down via props/events. No external store (e.g., Svelte stores, Redux) is introduced, keeping the implementation lightweight and consistent with the existing codebase.

If the app grows, these can be hoisted into a Svelte `derived` store or a `context` later.

## 9. File / Component Breakdown

| File                          | Action       | Purpose                                  |
| ----------------------------- | ------------ | ---------------------------------------- |
| `src/App.svelte`              | **Modify**   | Add state, parse files, render tree      |
| `src/lib/TreeView.svelte`     | **Create**   | Container for tree rendering             |
| `src/lib/TreeNode.svelte`     | **Create**   | Flat two-level node (file → prompts)       |
| `src/lib/PromptDetail.svelte` | **Create**   | Right-side detail panel                  |
| `src/app.css`                 | **Modify**   | Optional: tree indentation / dot colors  |

## 10. Validation & Type Safety

- JSDoc annotations (`/** @type {...} */`) are used for all new state variables, consistent with the existing code style (e.g., `let error = $state(/** @type {string | null} */ (null))`).
- The `accept=".txt"` attribute on `FileUpload` prevents non-`.txt` uploads at the browser level.
- File parsing gracefully handles files with trailing newlines or empty lines.

## 11. Edge Cases

| Case                         | Handling                                           |
| ---------------------------- | -------------------------------------------------- |
| File with no prompts         | File node rendered with empty `children` list      |
| Duplicate file names         | Node `id` includes filename; if two files share a |
|                              | name, child IDs may collide — use a unique suffix  |
| Prompt fails to submit       | Status becomes `error`, error message shown        |
| Prompt returns instantly     | Status goes `pending` → `polling` → `completed`    |
| No node selected             | Detail panel shows placeholder text                |
| Polling network failure      | Status becomes `error`                             |
