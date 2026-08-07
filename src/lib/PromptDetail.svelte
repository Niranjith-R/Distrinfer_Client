<script>
  /**
   * @typedef {{ id: string, label: string, type: 'file' | 'prompt', children: any[] }} PromptNode
   * @typedef {{ promptId: string, status: 'idle' | 'pending' | 'polling' | 'completed' | 'error', data: any, error: string | null }} PromptResult
   */

  let {
    selectedNodeId,
    nodes,
    results
  } = $props();

  /** @returns {PromptNode | null} */
  function getSelectedNode() {
    for (const fileNode of nodes) {
      for (const child of fileNode.children) {
        if (child.id === selectedNodeId) {
          return child;
        }
      }
    }
    return null;
  }

  const selectedNode = $derived(getSelectedNode());
  const result = $derived(selectedNodeId ? results[selectedNodeId] ?? null : null);
</script>

{#if !selectedNode}
  <div class="max-w-2xl mx-auto mb-6 p-4 bg-[#2d2f3d] rounded-xl text-gray-400 text-sm">
    Click a prompt in the sidebar to view details.
  </div>
{:else}
  <div class="max-w-2xl mx-auto mb-6 space-y-3">
    <div class="p-4 bg-[#2d2f3d] rounded-xl">
      <h3 class="text-xs text-gray-400 mb-1">Prompt</h3>
      <pre class="text-sm text-white whitespace-pre-wrap break-words">{selectedNode.label}</pre>
    </div>

    {#if result?.status === 'pending' || result?.status === 'polling'}
      <div class="p-4 bg-[#2d2f3d] rounded-xl flex items-center gap-3 text-sm text-white">
        <span class="animate-spin">⏳</span>
        <span>{result.status === 'pending' ? 'Sending...' : 'Polling...'}</span>
      </div>
    {:else if result?.status === 'error'}
      <div class="p-4 bg-[#2d2f3d] rounded-xl">
        <h3 class="text-xs text-red-400 mb-1">Error</h3>
        <p class="text-sm text-red-300">{result.error}</p>
      </div>
    {:else if result?.status === 'completed'}
      <div class="p-4 bg-[#2d2f3d] rounded-xl">
        <h3 class="text-xs text-gray-400 mb-1">Result</h3>
        <pre class="text-sm text-white whitespace-pre-wrap break-words">{JSON.stringify(result.data, null, 2)}</pre>
      </div>
    {/if}
  </div>
{/if}
