<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { causeOf, groupByCause, readAs, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // The desk's own view (CauseTable): a report counts where it is filed. The reports the desk read as
  // a one-day move stay counted under their cause, and get a ticket of their own as well.
  // tickets: the ticket each problem is filed as, problem → URL. states: where each ticket stands, in words.
  let {
    ticket,
    tickets = {},
    states = {},
  }: { ticket: number; tickets?: Record<string, string>; states?: Record<string, string> } = $props();

  const rows = $derived((reports as Report[]).filter((r) => r.mergedInto === ticket));
  const merged = $derived(uniqueById(rows));
  const groups = $derived(groupByCause(merged, tenantOf));
  const oneDay = $derived(merged.filter((r) => notes.oneDayMove.includes(r.id)));
  const mistagged = $derived(merged.filter((r) => r.meta.intakeTag !== causeOf(r, tenantOf(r))).length);

  const seen = (rs: Report[]) => {
    const days = rs.map((r) => r.receivedAt.slice(0, 10)).sort();
    return `${days[0]} → ${days[days.length - 1]}`;
  };
  const problems = $derived([
    ...groups.map((g) => ({ problem: g.cause, reports: g.reports, tenants: g.tenants, seen: `${g.firstSeen} → ${g.lastSeen}` })),
    { problem: 'one-day', reports: oneDay, tenants: new Set(oneDay.map((r) => r.tenant)).size, seen: seen(oneDay) },
  ]);

  const WHAT: Record<string, string> = {
    parser: 'day and month swapped: the shared parser reads month first',
    hint: 'day and month swapped: the booking-v3 form asks for the month first',
    'one-day': 'date moved by one day',
    unplaced: 'no tenant on file',
  };
  const label = (url: string) => url.replace(/^https:\/\/github\.com\/[^/]+\/([^/]+)\/issues\//, '$1#');
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports merged into #{ticket} ·
  <strong>{problems.length}</strong>
  {problems.length === 1 ? 'problem' : 'problems'}
</p>

<table>
  <thead>
    <tr><th>problem</th><th>what the reports say</th><th>reports</th><th>tenants</th><th>seen</th><th>ticket</th><th>state</th></tr>
  </thead>
  <tbody>
    {#each problems as p (p.problem)}
      <tr>
        <td><code>{p.problem}</code></td>
        <td>{WHAT[p.problem]}</td>
        <td>{p.reports.length}</td>
        <td>{p.tenants}</td>
        <td>{p.seen}</td>
        <td>{#if tickets[p.problem]}<a href={tickets[p.problem]}>{label(tickets[p.problem])}</a>{:else}none yet{/if}</td>
        <td>{states[p.problem] ?? ''}</td>
      </tr>
    {/each}
  </tbody>
</table>

{#each groups as g (g.cause)}
  {@const moved = readAs(g, notes.oneDayMove)}
  {#if moved}
    <p class="note">{moved} of the <code>{g.cause}</code> reports read as a one-day move: counted where they're filed, and filed again as <code>one-day</code>.</p>
  {/if}
{/each}

<p class="note">
  Counted once per report id: the export holds {rows.length} rows for {merged.length} reports. Intake's tag names
  a different cause than the tenant on {mistagged} of them.
</p>

<style>
  .sum { font-size: 1.1rem; }
  .note { color: var(--muted); font-size: 0.85em; }
</style>
