# Test-First Learning Lab

This suite is for learners who understand code best by pinning behavior with tests. It teaches where tests already exist, how test helpers work, and how to design missing tests before touching production code.

Primary test anchors:

- Router IDOR/status-report tests (`packages/api/src/router/statusReport.test.ts:77-207`)
- Service status-report tests (`packages/services/src/status-report/__tests__/status-report.test.ts:116-240`)
- Service test helpers (`packages/services/test/helpers.ts:18-50`; `packages/services/test/helpers.ts:121-151`; `packages/services/test/helpers.ts:240-264`)
- SSRF utility tests (`packages/utils/src/ssrf.test.ts:26-160`)
- Status-page utility logic (`packages/api/src/router/statusPage.utils.ts:128-258`)

Use this suite when you want to build confidence by asking: "What behavior would prove I understand this?"

