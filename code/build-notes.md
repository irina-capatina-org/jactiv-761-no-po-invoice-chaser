# Build Notes — no-po-invoice-chaser-761

## Plan

1. Load skills (uipath-api-workflow, uipath-platform, uipath-solution) and read SDD + arch
2. Extract reference Workflow.json from §4.5 of architectural-considerations.md
3. Scaffold solution with `uip solution init no-po-invoice-chaser-761`
4. Scaffold API workflow project with `uip api-workflow init no-po-invoice-chaser-api`
5. Replace scaffolded Workflow.json with reference; write correct bindings_v2.json
6. Create connection resource files (Coupa + Slack) in docVersion 1.0.0 format
7. Run validate-build gate and fix any errors
8. Pack solution and write build notes

## Summary

One API Workflow project (`no-po-invoice-chaser-api`) inside solution `no-po-invoice-chaser-761` that queries Coupa for invoices with no PO and posts a Block Kit Slack summary.

## Task Table

| Task | Project | Status | Notes |
|------|---------|--------|-------|
| Scaffold solution + project | no-po-invoice-chaser-api | done | CLI patched for Node 18 ESM compat |
| Extract reference workflow | no-po-invoice-chaser-api | done | Exact copy from §4.5, 0 edits needed |
| bindings_v2.json | no-po-invoice-chaser-api | done | Coupa + Slack connections |
| Connection resource files | no-po-invoice-chaser-761 | done | Hand-authored in docVersion 1.0.0 format (no refresh auth) |
| validate-build gate | — | done | passes, 19 activities |
| solution pack | — | done | no-po-invoice-chaser-761_0.0.1.zip |

## Deviations from the SDD

None. The reference workflow already implements all SDD business rules (BR-01 through BR-07). No edits to Workflow.json were required.

## Left for a Human

- `uip solution resources refresh` was not run — this runner has no cloud credentials. The deploy stage must run it before deploy to link connection keys to live Orchestrator entries. Connection resource files are present with `authenticationType: "AuthenticateAfterDeployment"` so deploy-time linking will work.
- The `uip` CLI (1.202.0) fails on Node 18.20.8 due to `styleText` not being exported via ESM from `node:util` (requires Node 20+). The three affected import statements in `index-qm4fye46.js` and the `crypto` global issue in `index-ezemqten.js` were patched in place on the runner; the patches do not affect build output.

## How to Test This

```bash
# Validate workflow structure
uip api-workflow validate code/no-po-invoice-chaser-761/no-po-invoice-chaser-api/Workflow.json --output json

# Runnability gate
bash .github/scripts/validate-build.sh code docs/architectural-considerations.md

# Pack
uip solution pack code/no-po-invoice-chaser-761 /tmp/out --name no-po-invoice-chaser-761 --version 0.0.1 --output json
```
