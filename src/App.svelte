<script>
  import { ArrowRight } from '@lucide/svelte';

  let API_BASE = $state('http://127.0.0.1:8000'); // replace with your actual host
//   const API_KEY = 'YOUR_API_KEY'; // replace with your actual key

  let prompt = $state('');
  let loading = $state(false);
  let polling = $state(false);
  let error = $state(/** @type {string | null} */ (null));
  let result = $state(/** @type {any | null} */ (null));
  let userPrompt = $state('');



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
      DISTRINFER
    </div>
    <hr class="hr" />
  </div>

  <div class="flex-1 p-6 flex flex-col h-screen">
    <div class="flex-1 overflow-y-auto">


        <div class="flex items-center gap-2 mt-auto">
            <input
            type="text"
            class="input rounded-full mb-15"
            placeholder="API Endpoint"
            bind:value={API_BASE}
        />
        </div>

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
              {result?.Data?.infered ?? JSON.stringify(result)}
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