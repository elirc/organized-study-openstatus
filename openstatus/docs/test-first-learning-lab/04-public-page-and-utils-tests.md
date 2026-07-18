# Public Page And Utility Tests

## Test Target 1: `getEvents`

`getEvents` collects maintenances, incidents, and reports into a common event shape (`packages/api/src/router/statusPage.utils.ts:128-258`).

Important branches:

- Maintenance filtering prefers `pageComponentId`, then falls back to `monitorId`, then includes all (`packages/api/src/router/statusPage.utils.ts:158-183`).
- Static components skip incidents (`packages/api/src/router/statusPage.utils.ts:185-203`).
- Reports filter through page component or monitor associations (`packages/api/src/router/statusPage.utils.ts:205-219`).
- Resolved reports produce `status: "success"` with `to` set to last update date (`packages/api/src/router/statusPage.utils.ts:232-243`).
- Unresolved reports produce `status: "degraded"` and may have `to: null` (`packages/api/src/router/statusPage.utils.ts:246-257`).

Illustrative fake test:

```ts
// Illustrative fake code.
test("static component ignores incidents", () => {
  const events = getEvents({
    maintenances: [],
    incidents: [incidentForMonitor],
    reports: [],
    componentType: "static",
  });
  expect(events.some((event) => event.type === "incident")).toBe(false);
});
```

## Test Target 2: Public Report Sorting

The public router sorts status reports after computing trackers and events (`packages/api/src/router/statusPage.ts:364-379`). Sorting is user-visible because feed order tells the incident story.

Illustrative fake test:

```ts
// Illustrative fake code.
test("reports with updates sort newest update first after empty reports", () => {
  const sorted = sortStatusReports([
    report({ id: 1, updateDates: ["2026-01-01"] }),
    report({ id: 2, updateDates: [] }),
    report({ id: 3, updateDates: ["2026-02-01"] }),
  ]);
  expect(sorted.map((r) => r.id)).toEqual([2, 3, 1]);
});
```

Test-first note: there is no exported `sortStatusReports` helper today in the inspected code. A senior refactor would extract pure sorting logic from `statusPage.get` before testing it directly (`packages/api/src/router/statusPage.ts:364-379`).

## Test Target 3: SSRF Safety

Existing SSRF tests are a model for exhaustive edge-case testing. They cover:

- allowed public URLs (`packages/utils/src/ssrf.test.ts:6-24`),
- blocked protocols (`packages/utils/src/ssrf.test.ts:26-40`),
- private IPv4 (`packages/utils/src/ssrf.test.ts:42-61`),
- private hostnames (`packages/utils/src/ssrf.test.ts:63-77`),
- private IPv6 and IPv4-mapped IPv6 (`packages/utils/src/ssrf.test.ts:79-126`),
- invalid URLs (`packages/utils/src/ssrf.test.ts:128-137`),
- async DNS-aware checks (`packages/utils/src/ssrf.test.ts:139-160`).

Pattern to copy:

```ts
// Illustrative fake code.
const blocked = ["bad-case-1", "bad-case-2"];
for (const value of blocked) {
  it(`blocks ${value}`, () => {
    expect(() => functionUnderTest(value)).toThrow();
  });
}
```

Why this matters: safety code needs a table of hostile examples, not one happy-path test.

## Test Target 4: Public Page Query Shape

`statusPage.get` output is parsed by `selectPublicPageSchemaWithRelation` (`packages/api/src/router/statusPage.ts:76-77`; `packages/api/src/router/statusPage.ts:385-400`). If you add a public field such as severity or draft visibility, test the output shape where the public read model is assembled.

Illustrative fake integration test:

```ts
// Illustrative fake code.
test("public page does not include draft reports", async () => {
  const caller = edgeRouter.createCaller(publicCtx);
  const page = await caller.statusPage.get({ slug: testSlug });
  expect(page?.statusReports.every((report) => !report.draft)).toBe(true);
  expect(page?.openEvents.every((event) => event.type !== "draft")).toBe(true);
});
```

Why not only client test: the public client filters and maps already-computed `page.statusReports` and `page.lastEvents` (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-360`). Draft visibility must be correct before that.

