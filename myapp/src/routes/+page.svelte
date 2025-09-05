<script lang="ts">
  import { mcp } from '$lib/mcp'
  let output = ''

  // Store per-tool argument objects
  let argsByTool: Record<string, any> = {}
  let expanded: Record<string, boolean> = {}

  function ensureArgs(name: string, schema: any) {
    if (argsByTool[name]) return
    const result: Record<string, any> = {}
    const properties = schema?.properties || {}
    for (const key of Object.keys(properties)) {
      const p = properties[key] || {}
      if (p?.default !== undefined) {
        result[key] = p.default
      } else if (p?.type === 'boolean') {
        result[key] = false
      } else if (p?.type === 'number' || p?.type === 'integer') {
        result[key] = 0
      } else if (p?.type === 'array') {
        result[key] = []
      } else if (p?.type === 'object') {
        result[key] = {}
      } else {
        result[key] = ''
      }
    }
    argsByTool = { ...argsByTool, [name]: result }
  }

  function setArg(toolName: string, key: string, value: any) {
    argsByTool = { ...argsByTool, [toolName]: { ...(argsByTool[toolName] || {}), [key]: value } }
  }

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
          {#key tool.name}
            <li style="margin: 0.75rem 0; padding: 0.5rem; border: 1px solid #eee; border-radius: 6px;">
              <div style="display:flex; align-items:center; justify-content:space-between; gap: 1rem;">
                <div>
                  <div><strong>{tool.name}</strong></div>
                  {#if tool.description}
                    <div style="color:#666">{tool.description}</div>
                  {/if}
                </div>
                <button on:click={() => { expanded[tool.name] = !expanded[tool.name]; expanded = { ...expanded }; ensureArgs(tool.name, tool.inputSchema || tool.input_schema || { properties: {} }) }}>
                  {expanded[tool.name] ? 'Hide inputs' : 'Show inputs'}
                </button>
              </div>

              {#if expanded[tool.name]}
                {@const schemaAny = (tool.inputSchema || tool.input_schema || { type:'object', properties:{} }) as any}
                  {#if schemaAny?.properties && Object.keys(schemaAny.properties).length > 0}
                    <div style="margin-top: 0.5rem; display: grid; gap: 0.5rem;">
                      {#each Object.entries(schemaAny.properties) as [key, prop]}
                        {#if (prop as any).enum}
                          <label style="display:block;">
                            <span>{key}{schemaAny?.required?.includes(key) ? ' *' : ''}</span>
                            <select value={argsByTool[tool.name]?.[key]} on:change={(e) => setArg(tool.name, key, (e.target as HTMLSelectElement).value)}>
                              {#each (prop as any).enum as opt}
                                <option value={String(opt)}>{String(opt)}</option>
                              {/each}
                            </select>
                          </label>
                        {:else if (prop as any).type === 'boolean'}
                          <label style="display:flex; gap:0.5rem; align-items:center;">
                            <input type="checkbox" checked={Boolean(argsByTool[tool.name]?.[key])} on:change={(e) => setArg(tool.name, key, (e.target as HTMLInputElement).checked)} />
                            <span>{key}{schemaAny?.required?.includes(key) ? ' *' : ''}</span>
                          </label>
                        {:else if (prop as any).type === 'number' || (prop as any).type === 'integer'}
                          <label style="display:block;">
                            <span>{key}{schemaAny?.required?.includes(key) ? ' *' : ''}</span>
                            <input type="number" value={argsByTool[tool.name]?.[key]} on:input={(e) => setArg(tool.name, key, Number((e.target as HTMLInputElement).value))} />
                          </label>
                        {:else if (prop as any).type === 'string'}
                          <label style="display:block;">
                            <span>{key}{schemaAny?.required?.includes(key) ? ' *' : ''}</span>
                            <input type="text" value={argsByTool[tool.name]?.[key]} on:input={(e) => setArg(tool.name, key, (e.target as HTMLInputElement).value)} />
                          </label>
                        {:else}
                          <label style="display:block;">
                            <span>{key} (JSON){schemaAny?.required?.includes(key) ? ' *' : ''}</span>
                            <textarea rows="3" on:input={(e) => {
                              try { setArg(tool.name, key, JSON.parse((e.target as HTMLTextAreaElement).value || 'null')) } catch { /* ignore */ }
                            }}>{JSON.stringify(argsByTool[tool.name]?.[key] ?? null)}</textarea>
                          </label>
                        {/if}
                      {/each}
                    </div>
                  {:else}
                    <div style="margin-top:0.5rem; color:#666;">No input parameters.</div>
                  {/if}

                <div style="margin-top: 0.5rem;">
                  <button on:click={() => callTool(tool.name, argsByTool[tool.name] || {})}>Call</button>
                </div>
              {/if}
            </li>
          {/key}
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
