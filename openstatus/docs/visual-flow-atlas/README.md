# Visual Flow Atlas

This suite is for spatial learners. It teaches OpenStatus as maps, sequences, shapes, and risk zones. Use it when prose feels too flat and you need to see how the parts connect.

This suite complements the existing docs without depending on them. It focuses on the same real code paths: dashboard status-report creation, tRPC routing, service transactions, Drizzle schema, and public status-page rendering (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:23-128`; `packages/api/src/router/statusReport.ts:58-222`; `packages/services/src/status-report/create.ts:24-99`; `packages/db/src/schema/status_reports/status_reports.ts:15-89`; `packages/api/src/router/statusPage.ts:65-400`; `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:56-414`).

Recommended route:

1. `01-system-maps.md`: learn the terrain.
2. `02-sequence-diagrams.md`: follow requests as moving signals.
3. `03-data-shape-gallery.md`: see what each layer thinks data looks like.
4. `04-risk-heatmaps.md`: learn where bugs and design pressure collect.

Every diagram is backed by file references. Fake code examples are labeled "Illustrative fake code" and exist only to explain a pattern more simply than production code can.

