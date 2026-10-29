<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, readAs, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem merged into `ticket`, counted the desk's way (by tenant, as CauseTable
  // does), plus the one-day moves the desk noted (desk-notes.json), which no cause in causes.ts
  // explains. `tickets` maps each row to the ticket that tracks it and where that ticket stands.
  type Tracked = { url: string; state: string };
  let { ticket, tickets }: { ticket: number; tickets: Record<string, Tracked> } = $props();

  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const shifted = $derived(merged.filter((r) => notes.oneDayMove.includes(r.id)));
  const shiftDays = $derived(shifted.map((r) => r.receivedAt.slice(0, 10)).sort());
  const rows = $derived([
    ...groupByCause(merged, tenantOf).map((g) => ({
      cause: g.cause as string,
      reports: g.reports.length,
      tenants: g.tenants,
      seen: `${g.firstSeen} → ${g.lastSeen}`,
      alsoShift: readAs(g, notes.oneDayMove),
    })),
    {
      cause: 'shift',
      reports: shifted.length,
      tenants: new Set(shifted.map((r) => r.tenant)).size,
      seen: `${shiftDays[0]} → ${shiftDays.at(-1)}`,
      alsoShift: 0,
    },
  ]);
  const issue = (url: string) => `#${url.split('/').pop()}`;
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports merged into #{ticket} ·
  <strong>{rows.length}</strong>
  {rows.length === 1 ? 'problem' : 'problems'}
</p>

<table>
  <thead>
    <tr><th>cause</th><th>reports</th><th>tenants</th><th>seen</th><th>ticket</th><th>state</th></tr>
  </thead>
  <tbody>
    {#each rows as r (r.cause)}
      {@const t = tickets[r.cause]}
      <tr class:open={!t || t.state === 'open'}>
        <td><code>{r.cause}</code></td>
        <td>{r.reports}</td>
        <td>{r.tenants}</td>
        <td>{r.seen}</td>
        <td>{#if t}<a href={t.url}>{issue(t.url)}</a>{:else}no ticket yet{/if}</td>
        <td>{t?.state ?? 'open'}</td>
      </tr>
    {/each}
  </tbody>
</table>

{#each rows as r (r.cause)}
  {#if r.alsoShift}
    <p class="note">{r.alsoShift} of the <code>{r.cause}</code> reports are the one-day moves: the desk files them under <code>{r.cause}</code> by tenant, and they are counted again under <code>shift</code>.</p>
  {/if}
{/each}

<style>
  .sum { font-size: 1.1rem; }
  .note { color: var(--muted); font-size: 0.85em; }
  .open td { background: var(--warn-bg); }
  .open td:last-child { color: var(--warn); font-weight: 600; }
</style>
