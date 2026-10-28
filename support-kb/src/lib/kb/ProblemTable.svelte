<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem in a merged ticket. Same grouping as CauseTable, except the reports the desk
  // read as a one-day move (desk-notes.json) get their own row: their bodies describe a date shifted
  // by one day, not a day and month swapped, so they are a different bug, not a footnote.
  let { ticket, issues = {} }: { ticket: number; issues?: Record<string, string> } = $props();

  type Row = { problem: string; reports: Report[] };

  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const rows = $derived.by((): Row[] => {
    const moved = (r: Report) => notes.oneDayMove.includes(r.id);
    const byCause = groupByCause(merged.filter((r) => !moved(r)), tenantOf).map((g) => ({
      problem: g.cause as string,
      reports: g.reports,
    }));
    return [...byCause, { problem: 'one-day', reports: merged.filter(moved) }]
      .filter((r) => r.reports.length)
      .sort((a, b) => b.reports.length - a.reports.length);
  });
  const tenants = (rs: Report[]) => new Set(rs.map((r) => r.tenant)).size;
  const seen = (rs: Report[]) => {
    const days = rs.map((r) => r.receivedAt.slice(0, 10)).sort();
    return `${days[0]} → ${days[days.length - 1]}`;
  };
  const tags = (rs: Report[]) =>
    Object.entries(rs.reduce<Record<string, number>>((n, r) => ({ ...n, [r.meta.intakeTag]: (n[r.meta.intakeTag] ?? 0) + 1 }), {}))
      .sort((a, b) => b[1] - a[1])
      .map(([tag, n]) => `${tag} ×${n}`)
      .join(' · ');
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports merged into #{ticket} ·
  <strong>{rows.length}</strong>
  {rows.length === 1 ? 'problem' : 'problems'}
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tagged as</th><th>tenants</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each rows as row (row.problem)}
      <tr>
        <td><code>{row.problem}</code></td>
        <td>{row.reports.length}</td>
        <td class="tags">{tags(row.reports)}</td>
        <td>{tenants(row.reports)}</td>
        <td>{seen(row.reports)}</td>
        <td>
          {#if issues[row.problem]}<a href={issues[row.problem]}>{issues[row.problem].replace('https://github.com/', '').replace('/issues/', '#')}</a>{:else}none yet{/if}
        </td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .sum { font-size: 1.1rem; }
  .tags { color: var(--muted); font-size: 0.85em; }
</style>
