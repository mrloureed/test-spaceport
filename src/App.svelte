<script lang="ts">
  import { getCookie } from 'typescript-cookie';
  import * as VIAM from '@viamrobotics/sdk';
  import SpaceWindow from './lib/SpaceWindow.svelte';

  let showSpaceWindow = $state(false);

  type CookieData = {
    apiKey: { id: string; key: string };
    credentials: { type: string; payload: string; authEntity: string };
    hostname: string;
    machineId: string;
  };

  type AppStatus = 'loading' | 'connected' | 'no-viam' | 'error';

  const state = $state({
    status: 'loading' as AppStatus,
    error: '',
    cookie: null as CookieData | null,
    resources: [] as VIAM.ResourceName[],
    locationId: '',
    showKey: false,
  });

  // The URL path is /machine/{machine-hostname}/... for single-machine apps.
  // The cookie name matches this hostname segment.
  const machineKey = window.location.pathname.split('/')[2] ?? '';

  function parseLocationId(hostname: string): string {
    const match = hostname.match(/\.([^.]+)\.viam\.cloud$/);
    return match?.[1] ?? '';
  }

  async function init() {
    if (!machineKey) {
      // No Viam URL — running standalone (e.g. GitHub Pages demo)
      state.status = 'no-viam';
      return;
    }

    const raw = getCookie(machineKey);
    if (!raw) {
      state.status = 'no-viam';
      return;
    }

    let parsed: CookieData;
    try {
      parsed = JSON.parse(raw);
      state.cookie = parsed;
      state.locationId = parseLocationId(parsed.hostname);
    } catch {
      state.error = 'Failed to parse machine cookie data.';
      state.status = 'error';
      return;
    }

    try {
      const client = await VIAM.createRobotClient({
        host: parsed.hostname,
        credentials: parsed.credentials,
        signalingAddress: 'https://app.viam.com:443',
      });
      state.resources = await client.resourceNames();
      await client.disconnect();
      state.status = 'connected';
    } catch (e) {
      state.error = `Connection failed: ${String(e)}`;
      state.status = 'error';
    }
  }

  $effect(() => {
    init();
  });
</script>

{#if showSpaceWindow}
  <SpaceWindow onclose={() => (showSpaceWindow = false)} />
{/if}

<main>
  <header>
    <h1>🚀 Spaceport</h1>
    <p class="subtitle">Viam Machine Dashboard</p>
  </header>

  {#if state.status === 'loading'}
    <div class="card">
      <p class="muted center">Connecting to machine…</p>
    </div>
  {:else if state.status === 'no-viam'}
    <div class="card launch-card">
      <button class="launch-btn" onclick={() => (showSpaceWindow = true)}>🚀 Launch Space Window</button>
    </div>
    <div class="card">
      <p class="muted center">Not connected to a Viam machine — running in demo mode.</p>
    </div>
  {:else if state.status === 'error'}
    <div class="card error-card">
      <h2>Connection Error</h2>
      <p class="error-message">{state.error}</p>
    </div>
  {:else}
    <div class="card launch-card">
      <button class="launch-btn" onclick={() => (showSpaceWindow = true)}>🚀 Launch Space Window</button>
    </div>

    <div class="card">
      <h2>✅ Connected</h2>
      <table>
        <tbody>
          <tr>
            <th>Hostname</th>
            <td>{state.cookie?.hostname}</td>
          </tr>
          <tr>
            <th>Machine ID</th>
            <td>{state.cookie?.machineId}</td>
          </tr>
          <tr>
            <th>Location ID</th>
            <td>{state.locationId}</td>
          </tr>
          <tr>
            <th>API Key ID</th>
            <td>
              {#if state.showKey}
                {state.cookie?.apiKey?.id}
              {:else}
                <span class="masked">••••••••••••••••••••••••••••••••••••</span>
              {/if}
              <button class="toggle-btn" onclick={() => (state.showKey = !state.showKey)}>
                {state.showKey ? 'hide' : 'show'}
              </button>
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <div class="card">
      <h2>Resources <span class="badge">{state.resources.length}</span></h2>
      {#if state.resources.length === 0}
        <p class="muted">No resources found.</p>
      {:else}
        <ul class="resource-list">
          {#each state.resources as r}
            <li>
              <span class="resource-type">{r.namespace}:{r.type}/{r.subtype}</span>
              <span class="resource-name">{r.name}</span>
            </li>
          {/each}
        </ul>
      {/if}
    </div>
  {/if}
</main>

<style>
  main {
    max-width: 800px;
    margin: 0 auto;
    padding: 2rem 1rem;
  }

  header {
    text-align: center;
    margin-bottom: 2rem;
  }

  h1 {
    font-size: 2.5rem;
    font-weight: 700;
    color: #7c3aed;
  }

  .subtitle {
    color: #64748b;
    margin-top: 0.25rem;
    font-size: 0.95rem;
  }

  .card {
    background: #1e2130;
    border: 1px solid #2d3148;
    border-radius: 0.75rem;
    padding: 1.5rem;
    margin-bottom: 1rem;
  }

  .card h2 {
    font-size: 1rem;
    font-weight: 600;
    margin-bottom: 1rem;
    color: #cbd5e1;
    display: flex;
    align-items: center;
    gap: 0.5rem;
  }

  .error-card {
    border-color: #7f1d1d;
    background: #1c0f0f;
  }

  .error-card h2 {
    color: #f87171;
  }

  .error-message {
    color: #fca5a5;
    font-family: monospace;
    font-size: 0.875rem;
    white-space: pre-wrap;
    word-break: break-word;
  }

  .badge {
    background: #312e81;
    color: #a5b4fc;
    font-size: 0.75rem;
    padding: 0.1rem 0.5rem;
    border-radius: 9999px;
    font-weight: 500;
  }

  table {
    width: 100%;
    border-collapse: collapse;
  }

  th {
    text-align: left;
    padding: 0.4rem 1rem 0.4rem 0;
    color: #64748b;
    font-weight: 500;
    font-size: 0.8rem;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    width: 120px;
    vertical-align: top;
  }

  td {
    padding: 0.4rem 0;
    font-family: monospace;
    font-size: 0.875rem;
    color: #a5b4fc;
    word-break: break-all;
  }

  .masked {
    color: #475569;
    letter-spacing: 0.1em;
  }

  .toggle-btn {
    margin-left: 0.5rem;
    background: none;
    border: 1px solid #334155;
    border-radius: 0.25rem;
    color: #64748b;
    font-size: 0.7rem;
    padding: 0.1rem 0.4rem;
    cursor: pointer;
    vertical-align: middle;
  }

  .toggle-btn:hover {
    border-color: #7c3aed;
    color: #a5b4fc;
  }

  .resource-list {
    list-style: none;
    display: flex;
    flex-direction: column;
    gap: 0.3rem;
  }

  .resource-list li {
    display: flex;
    align-items: baseline;
    gap: 0.75rem;
    padding: 0.35rem 0.6rem;
    background: #0f172a;
    border-radius: 0.375rem;
    font-size: 0.85rem;
  }

  .resource-type {
    color: #64748b;
    font-family: monospace;
    white-space: nowrap;
    font-size: 0.8rem;
  }

  .resource-name {
    color: #93c5fd;
    font-family: monospace;
    font-weight: 500;
  }

  .muted {
    color: #475569;
    font-size: 0.875rem;
  }

  .center {
    text-align: center;
    padding: 1rem 0;
  }

  .launch-card {
    display: flex;
    justify-content: center;
    padding: 1.25rem;
  }

  .launch-btn {
    background: linear-gradient(135deg, #4c1d95, #7c3aed);
    border: none;
    border-radius: 0.6rem;
    color: #fff;
    font-size: 1rem;
    font-weight: 600;
    padding: 0.75rem 2rem;
    cursor: pointer;
    letter-spacing: 0.03em;
    transition: opacity 0.15s, transform 0.1s;
  }

  .launch-btn:hover {
    opacity: 0.9;
    transform: scale(1.02);
  }
</style>
