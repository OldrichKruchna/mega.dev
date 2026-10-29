<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, readAs, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // The problems inside a merged ticket, counted the desk's way (as CauseTable does): every report
  // under its cause. The one-day moves get their own row too, but stay counted where they're filed.
  let { ticket, links = {} }: { ticket: number; links?: Record<string, string> } = $props();

  const LABEL: Record<string, string> = {
    parser: 'parser reads dates month first',
    hint: 'booking-v3 form hint says MM/DD/YYYY',
    'one-day': 'date moves by one day',
    unplaced: 'no tenant on file',
  };

  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const groups = $derived(groupByCause(merged, tenantOf));
  const oneDay = $derived(merged.filter((r) => notes.oneDayMove.includes(r.id)));
  const rows = $derived.by(() => {
    const days = oneDay.map((r) => r.receivedAt.slice(0, 10)).sort();
    return [
      ...groups.map((g) => ({ problem: g.cause, n: g.reports.length, tenants: g.tenants, first: g.firstSeen, last: g.lastSeen, also: '' })),
      {
        problem: 'one-day',
        n: oneDay.length,
        tenants: new Set(oneDay.map((r) => r.tenant)).size,
        first: days[0],
        last: days[days.length - 1],
        also: groups.filter((g) => readAs(g, notes.oneDayMove)).map((g) => `also counted under ${g.cause}`).join(', '),
      },
    ];
  });
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports merged into #{ticket} · <strong>{rows.length}</strong> problems
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each rows as row (row.problem)}
      <tr>
        <td>{LABEL[row.problem]}{#if row.also}<br /><small>{row.also}</small>{/if}</td>
        <td>{row.n}</td>
        <td>{row.tenants}</td>
        <td>{row.first} → {row.last}</td>
        <td>{#if links[row.problem]}<a href={links[row.problem]}>{links[row.problem].split('/').slice(-3).join('/')}</a>{:else}—{/if}</td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .sum { font-size: 1.1rem; }
  small { color: var(--muted); }
</style>
