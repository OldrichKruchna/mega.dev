<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { causeOf, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // The problems inside a merged ticket: the desk's causes, with the one-day moves split out,
  // because a day/month swap cannot move a date by one day.
  let { ticket, links = {} }: { ticket: number; links?: Record<string, string> } = $props();

  type Problem = 'parser' | 'hint' | 'one-day' | 'unplaced';
  const LABEL: Record<Problem, string> = {
    parser: 'parser reads dates month first',
    hint: 'booking-v3 form hint says MM/DD/YYYY',
    'one-day': 'date moves by one day',
    unplaced: 'no tenant on file',
  };

  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const problemOf = (r: Report): Problem =>
    notes.oneDayMove.includes(r.id) ? 'one-day' : (causeOf(r, tenantOf(r)) ?? 'unplaced');
  const rows = $derived.by(() => {
    const by = new Map<Problem, Report[]>();
    for (const r of merged) by.set(problemOf(r), [...(by.get(problemOf(r)) ?? []), r]);
    return [...by.entries()]
      .map(([problem, rs]) => {
        const days = rs.map((r) => r.receivedAt.slice(0, 10)).sort();
        return { problem, n: rs.length, tenants: new Set(rs.map((r) => r.tenant)).size, first: days[0], last: days[days.length - 1] };
      })
      .sort((a, b) => b.n - a.n);
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
        <td>{LABEL[row.problem]}</td>
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
</style>
