# Distrinfer Client

A Svelte + Vite frontend for [Distrinfer](https://github.com/niranjithr/Distrinfer) — an open-source distributed inference platform. Upload prompt files, send queries to the inference backend, and view results in a dark-themed interface.

## Features

- **File Upload** — Upload `.txt` files containing one prompt per line. Each line is parsed into an individual prompt node.
- **Prompt Tree** — Prompts are organized in a collapsible sidebar tree, with per-prompt status indicators (idle, pending, polling, completed, error).
- **Real-time Polling** — Submitted prompts are polled until the backend returns a result.
- **Chat UI** — Interact with the inference backend via a conversational interface.
- **Skeleton + Tailwind CSS** — Styled with the Skeleton design system.

## Requirements

- Node.js 20+
- A running Distinfer backend (default: `http://127.0.0.1:8000`)

## Setup

```bash
git clone https://github.com/niranjithr/Distrinfer-client.git
cd Distrinfer-client/Distrinfer
npm install
```

### Configure the API Endpoint

The client defaults to `http://127.0.0.1:8000`. Edit `src/App.svelte` to change the `API_BASE` value, or override it at runtime via the input field in the UI.

## Development

```bash
npm run dev
```

Opens [http://localhost:5173](http://localhost:5173) in your browser.

## Build

```bash
npm run build
```

Outputs static files to `dist/` for deployment.

## Preview

```bash
npm run preview
```

Serves the production build locally for preview.

## Project Structure

```
src/
├── App.svelte              # Root component: sidebar + main chat area
├── main.js                 # Svelte mount entry point
├── app.css                 # Tailwind imports + Skeleton theme + custom styles
└── lib/
    ├── TreeNode.svelte     # Collapsible tree node with prompt status dots
    ├── PromptDetail.svelte # Detail view for a selected prompt result
    └── Counter.svelte      # Placeholder demo component
```

## API

The client communicates with the Distinfer backend at two endpoints:

| Method | Endpoint         | Purpose                          |
|--------|------------------|----------------------------------|
| POST   | `/query`         | Submit a prompt, receive a hash  |
| GET    | `/query/{hash}`  | Poll for result by hash          |

## License

GPL 3.0 — see the [LICENSE](LICENSE) file.
