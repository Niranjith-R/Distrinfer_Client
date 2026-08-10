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
  <!-- <div class="max-w-2xl mx-auto mb-6 p-4 bg-[#2d2f3d] rounded-xl text-gray-400 text-sm"> -->
    <!-- Click a prompt in the sidebar to view details. -->
  <!-- </div> -->
{:else}
  <div class="max-w-2xl mx-auto mb-6 space-y-4">
    <!-- User prompt bubble -->
    <div class="flex justify-end">
      <div class="bg-[#3a3f5c] text-white rounded-2xl rounded-br-md px-4 py-2 max-w-[70%] text-sm">
        {selectedNode.label}
      </div>
    </div>

    <!-- Result bubble -->
    {#if result?.status === 'pending' || result?.status === 'polling'}
      <div class="flex justify-start">
        <div class="bg-[#2d2f3d] text-white rounded-2xl rounded-bl-md px-4 py-3 max-w-[70%] flex items-center gap-3 text-sm">
          <div class="dot-loader">
            <span class="dot-loader_dot"></span>
            <span class="dot-loader_dot"></span>
            <span class="dot-loader_dot"></span>
          </div>
        </div>
      </div>
    {:else if result?.status === 'error'}
      <div class="flex justify-start">
        <div class="bg-[#2d2f3d] text-red-300 rounded-2xl rounded-bl-md px-4 py-2 max-w-[70%] text-sm">
          {result.error}
        </div>
      </div>
    {:else if result?.status === 'completed'}
      <div class="flex justify-start">
        <div class="bg-[#2d2f3d] text-white rounded-2xl rounded-bl-md px-4 py-2 max-w-[70%] text-sm">
          <span class="text-gray-400 text-xs">Host: {result.data?.Data?.host ?? 'unknown'}</span>
          <span class="block mt-1">{result.data?.Data?.infered ?? JSON.stringify(result.data)}</span>
        </div>
      </div>
    {/if}
  </div>
{/if}
