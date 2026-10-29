<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem in the ticket, counted the desk's way: every report under its tenant's cause.
  // Reports the desk read as a one-day move (desk-notes.json) are their own problem too, so they
  // also get a row of their own: a month/day flip cannot move a date by one day.
  let { ticket, issues = {} }: { ticket: number; issues?: Record<string, string> } = $props();

  const LABEL: Record<string, string> = {
    parser: 'parser reads day-first dates month first',
    hint: 'booking-v3 form hint says MM/DD, form reads DD/MM',
    'one-day': 'date moves by one day',
    unplaced: 'no tenant on file',
  };

  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const rows = $derived.by(() => {
    const moved = merged.filter((r) => notes.oneDayMove.includes(r.id));
    const groups = groupByCause(merged, tenantOf).map((g) => ({ key: g.cause as string, reports: g.reports }));
    if (moved.length) groups.push({ key: 'one-day', reports: moved });
    return groups.map((g) => {
      const days = g.reports.map((r) => r.receivedAt.slice(0, 10)).sort();
      return {
        ...g,
        tenants: new Set(g.reports.map((r) => r.tenant)).size,
        firstSeen: days[0],
        lastSeen: days[days.length - 1],
      };
    });
  });
  const moved = $derived(rows.find((r) => r.key === 'one-day'));
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports merged into #{ticket} · <strong>{rows.length}</strong> problems
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each rows as r (r.key)}
      <tr>
        <td><code>{r.key}</code> {LABEL[r.key] ?? ''}</td>
        <td>{r.reports.length}</td>
        <td>{r.tenants}</td>
        <td>{r.firstSeen} → {r.lastSeen}</td>
        <td>{#if issues[r.key]}<a href={issues[r.key]}>{issues[r.key].replace(/^.*\/issues\//, '#')}</a>{:else}—{/if}</td>
      </tr>
    {/each}
  </tbody>
</table>

{#if moved}
  <p class="note">The {moved.reports.length} <code>one-day</code> reports are also counted under their tenant's cause, where the desk filed them.</p>
{/if}

<style>
  .sum { font-size: 1.1rem; }
  .note { color: var(--muted); font-size: 0.85em; }
</style>
