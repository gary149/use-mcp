<script lang="ts">
  import { mcp } from '$lib/mcp'
  let output = ''

  async function callTool(name: string, args: Record<string, unknown> = {}) {
    if (!mcp) return
    try {
      const res = await mcp.callTool(name, args)
      output = JSON.stringify(res, null, 2)
    } catch (e) {
      output = 'Tool call failed: ' + (e instanceof Error ? e.message : String(e))
    }
  }
</script>

{#if mcp}
  {#if $mcp.state === 'failed'}
    <p>Connection failed: {$mcp.error}</p>
    <button on:click={() => mcp.retry()}>Retry</button>
    <button on:click={() => mcp.authenticate()}>Authenticate Manually</button>
  {:else if $mcp.state !== 'ready'}
    <p>Connecting to AI service… (state: {$mcp.state})</p>
  {:else}
    <h2>Available Tools: {$mcp.tools.length}</h2>

    {#if $mcp.tools.length > 0}
      <ul>
        {#each $mcp.tools as tool}
          <li style="margin: 0.5rem 0;">
            <div><strong>{tool.name}</strong></div>
            {#if tool.description}
              <div style="color:#666">{tool.description}</div>
            {/if}
            <div style="margin-top: 0.25rem;">
              <button on:click={() => callTool(tool.name)}>Call</button>
            </div>
          </li>
        {/each}
      </ul>
    {/if}

    {#if output}
      <pre>{output}</pre>
    {/if}
  {/if}
{:else}
  <p>Loading…</p>
{/if}

