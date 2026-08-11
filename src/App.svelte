<script>
  import { ArrowRight } from '@lucide/svelte';
  import { FileUpload } from '@skeletonlabs/skeleton-svelte';
  import TreeNode from './lib/TreeNode.svelte';
  import PromptDetail from './lib/PromptDetail.svelte';

  let API_BASE = $state('http://127.0.0.1:8000'); // replace with your actual host
//   const API_KEY = 'YOUR_API_KEY'; // replace with your actual key

  let prompt = $state('');
  let loading = $state(false);
  let polling = $state(false);
  let error = $state(/** @type {string | null} */ (null));
  let result = $state(/** @type {any | null} */ (null));
  let userPrompt = $state('');
  let acceptedFiles = $state(/** @type {File[]} */ ([]));

  /** @typedef {{ id: string, label: string, type: 'file' | 'prompt', children: any[] }} PromptNode */
  /** @typedef {{ promptId: string, status: 'idle' | 'pending' | 'polling' | 'completed' | 'error', data: any, error: string | null }} PromptResult */

  /** @type {PromptNode[]} */
  let treeNodes = $state([]);
  /** @type {Record<string, PromptResult>} */
  let promptResults = $state({});
  let selectedPromptNodeId = $state(/** @type {string | null} */ (null));

  /**
   * @param {File} file
   * @returns {Promise<PromptNode>}
   */
  async function buildTreeNode(file) {
    const content = await file.text();
    const lines = content
      .split('\n')
      .map((/** @type {string} */ l) => l.trim())
      .filter((/** @type {string} */ l) => l.length > 0);

    return {
      id: `file:${file.name}`,
      label: file.name,
      type: 'file',
      children: lines.map((/** @type {string} */ line, /** @type {number} */ idx) => ({
        id: `file:${file.name}:${idx}`,
        label: line,
        type: 'prompt',
        children: []
      }))
    };
  }

  /**
   * @param {{ files: File[] }} details
   */
  async function handleFileAccept(details) {
    for (const file of details.files) {
      const node = await buildTreeNode(file);
      treeNodes = [...treeNodes, node];
      for (const child of node.children) {
        runPrompt(child.label, child.id);
      }
    }
  }

  /**
   * @param {string} promptText
   * @param {string} nodeId
   */
  async function runPrompt(promptText, nodeId) {
    promptResults[nodeId] = {
      promptId: '',
      status: 'pending',
      data: null,
      error: null
    };

    try {
      const res = await fetch(`${API_BASE}/query`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ prompt: promptText })
      });

      const data = await res.json();
      const hash = data.hash;

      promptResults[nodeId].promptId = hash;
      promptResults[nodeId].status = 'polling';

      pollForPromptResult(hash, nodeId);
    } catch (/** @type {any} */ err) {
      promptResults[nodeId].status = 'error';
      promptResults[nodeId].error = err.message;
    }
  }

  /**
   * @param {string} hash
   * @param {string} nodeId
   */
  function pollForPromptResult(hash, nodeId) {
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
      } catch (/** @type {any} */ err) {
        promptResults[nodeId].status = 'error';
        promptResults[nodeId].error = err.message;
      }
    };

    poll();
  }

  /**
   * @param {PromptNode} node
   */
  function handlePromptClick(node) {
    selectedPromptNodeId = node.id;
    userPrompt = '';
    result = null;
    error = null;
    polling = false;
    loading = false;
    const existing = promptResults[node.id];
    if (!existing || existing.status === 'idle' || existing.status === 'error') {
      runPrompt(node.label, node.id);
    }
  }

  async function sendPrompt() {
    if (!prompt.trim() || loading) return;

    loading = true;
    error = null;
    result = null;

    try {
      const res = await fetch(`${API_BASE}/query`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ prompt })
      });

      const data = await res.json();
      const promptId = data.hash;

      userPrompt = prompt;
      prompt = '';
      polling = true;
      await pollForResult(promptId);
    } catch (err) {
      error = /** @type {Error} */ (err).message;
      loading = false;
      polling = false;
    }
  }

  /** @param {string} promptId */
  async function pollForResult(promptId) {
    const POLL_INTERVAL = 1500; // ms between polls

    const poll = async () => {
      try {
        const res = await fetch(`${API_BASE}/query/${promptId}`);
        const data = await res.json();

        if (data.status === 'Pending' || data.status === 'Processing') {
          setTimeout(poll, POLL_INTERVAL);
        } else {
          result = data;
          loading = false;
          polling = false;
        }
      } catch (err) {
        error = /** @type {Error} */ (err).message;
        loading = false;
        polling = false;
      }
    };

    poll();
  }
</script>

<div class="flex">
  <div class="bg-[#272935] w-85 h-screen flex flex-col text-white">
    <div class="flex flex-col items-center font-['monoton'] text-5xl pt-20 pb-20">
      <a href="/">
        DISTRINFER
      </a>
    </div>
    <hr class="hr" />
    <div class="flex-1 overflow-y-auto px-2 py-2 space-y-1">
      {#if treeNodes.length === 0}
        <p class="text-xs text-gray-500 text-center pt-4">No files uploaded</p>
      {:else}
        {#each treeNodes as node (node.id)}
          <TreeNode {node} promptResults={promptResults} onPromptClick={handlePromptClick} />
        {/each}
      {/if}
    </div>
  </div>

  <div class="flex-1 p-6 flex flex-col h-screen">
    <div class="flex-1 overflow-y-auto">


        <div class="flex items-center gap-2 mt-auto">
            <input
            type="text"
            class="input rounded-full mb-15"
            placeholder="API Endpoint"
            bind:value={API_BASE}
        >
        </div>

      <PromptDetail
        selectedNodeId={selectedPromptNodeId}
        nodes={treeNodes}
        results={promptResults}
      />

      <div class="max-w-2xl mx-auto space-y-4">
        {#if userPrompt}
          <div class="flex justify-end">
            <div class="bg-[#3a3f5c] text-white rounded-2xl rounded-br-md px-4 py-2 max-w-[70%]">
              {userPrompt}
            </div>
          </div>
        {/if}

         {#if result}
          <div class="flex justify-start">
            <div class="bg-[#2d2f3d] text-white rounded-2xl rounded-bl-md px-4 py-2 max-w-[70%]">
              <span class="text-gray-400 text-xs">Host: {result?.Data?.host ?? 'unknown'}</span>
              <span class="block mt-1">{result?.Data?.infered ?? JSON.stringify(result)}</span>
            </div>
          </div>
        {/if}

        {#if polling}
          <div class="flex justify-start">
            <div class="bg-[#2d2f3d] text-white rounded-2xl rounded-bl-md px-4 py-3 max-w-[70%]">
              <div class="dot-loader">
                <span class="dot-loader_dot"></span>
                <span class="dot-loader_dot"></span>
                <span class="dot-loader_dot"></span>
              </div>
            </div>
          </div>
        {/if}

        {#if error}
          <p class="text-red-400 text-sm">{error}</p>
        {/if}
      </div>
    </div>
    <div class="">
        <FileUpload
        accept=".txt"
        maxFiles={Infinity}
        bind:acceptedFiles
        onFileAccept={handleFileAccept}
        >
        <FileUpload.Dropzone>
            <FileUpload.Trigger />
            <FileUpload.HiddenInput />
        </FileUpload.Dropzone>
        <FileUpload.ItemGroup>
            {#each acceptedFiles as file (file.name)}
                <FileUpload.Item {file}>
                <FileUpload.ItemName />
                <FileUpload.ItemSizeText />
                <FileUpload.ItemDeleteTrigger />
                </FileUpload.Item>
            {/each}

        </FileUpload.ItemGroup>
        <FileUpload.ClearTrigger />
    </FileUpload>
    </div>
    <div class="flex items-center gap-2 mt-auto">
      <input
        type="text"
        class="input rounded-full"
        placeholder="Type here..."
        bind:value={prompt}
        disabled={loading}
        onkeydown={(e) => e.key === 'Enter' && sendPrompt()}
      />
      <button
        type="button"
        class="btn-icon btn-icon-xl preset-filled rounded-full"
        title="Go"
        aria-label="Go"
        onclick={sendPrompt}
        disabled={loading}
      >
        <ArrowRight class="w-5 h-5" />
      </button>
    </div>
  </div>
</div>
