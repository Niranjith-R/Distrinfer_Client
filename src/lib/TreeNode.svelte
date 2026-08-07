<script>
  /**
   * @typedef {{ id: string, label: string, type: 'file' | 'prompt', children: any[] }} PromptNode
   * @typedef {{ promptId: string, status: 'idle' | 'pending' | 'polling' | 'completed' | 'error', data: any, error: string | null }} PromptResult
   */

  let {
    node,
    promptResults,
    onPromptClick
  } = $props();

  /** @type {boolean} */
  let expanded = $state(false);
</script>

<div class="tree-node">
  <div
    role="button"
    tabindex="0"
    class="file-header flex items-center gap-1 px-2 py-1 hover:bg-[#3a3f5c] cursor-pointer text-sm rounded"
    onclick={() => (expanded = !expanded)}
    onkeydown={(/** @type {{ key: string }} */ e) => e.key === 'Enter' && (expanded = !expanded)}
  >
    <svg
      class="w-3 h-3 transition-transform flex-none"
      class:rotate-180={expanded}
      fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"
    >
      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7"></path>
    </svg>
    <span class="truncate">{node.label}</span>
  </div>

  {#if expanded}
    <div class="prompt-children ml-4">
      {#each node.children as child (child.id)}
        {@const res = promptResults[child.id]}
          <div
            role="button"
            tabindex="0"
            class="prompt-item flex items-center gap-2 px-2 py-1 text-sm text-white hover:bg-[#2d2f3d] cursor-pointer rounded truncate"
            onclick={() => onPromptClick(child)}
            onkeydown={(/** @type {{ key: string }} */ e) => e.key === 'Enter' && onPromptClick(child)}
          >
          <span
            class="status-dot w-2 h-2 rounded-full flex-none"
            class:bg-gray-400={!res || res.status === 'idle'}
            class:bg-yellow-400={res?.status === 'pending' || res?.status === 'polling'}
            class:bg-green-400={res?.status === 'completed'}
            class:bg-red-400={res?.status === 'error'}
          ></span>
          <span class="truncate">{child.label}</span>
          {#if res?.status === 'polling'}
            <span class="text-xs text-gray-400">(polling)</span>
          {/if}
        </div>
      {/each}
    </div>
  {/if}
</div>
