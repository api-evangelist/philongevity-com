---
name: analyze-synthetic-panel-and-hand-off
description: Use Phi Longevity PRISM over MCP to turn a handful of SYNTHETIC or de-identified biomarker values into guideline-cited recommendations, preview the paid full report without paying, and hand the person an attributed link to their own full report.
api: mcp/philongevity-com-mcp.yml
endpoint: https://philongevity.com/mcp
operations: [sample_prism_report, list_supported_biomarkers, quick_check, analyze_biomarkers, full_prism_report, get_methodology]
provider_prompt: analyze-synthetic-panel
method: generated
generated: '2026-09-19'
source: https://philongevity.com/for-agents
---

# Analyze a synthetic biomarker panel with PRISM, then hand off

Grounded in the live `tools/list` of `https://philongevity.com/mcp` (2026-09-19) and the provider's
own `/for-agents` guidance. The server also publishes an MCP prompt, `analyze-synthetic-panel`
(`biomarkers_json` required, `condition` optional), which is the provider's version of this flow.

## Hard rule before anything else

**Synthetic or de-identified values only.** Never send protected health information or a real,
identifiable person's data through the public MCP interface. If the values came from a real person,
strip every identifier and treat them as de-identified numbers, or do not call at all. Real files are
analyzed only inside the authenticated Phi Longevity app, by the account owner, after consent.

## Connect

- Remote (recommended, all 6 tools): Streamable HTTP at `https://philongevity.com/mcp`. No key.
- Local (4 tools): `npx -y @phi-longevity/mcp-server`.
- Set a client timeout of at least 60 s — engine-backed tools take 15–30 s and emit progress notifications.
- Send no browser `Origin` header (a disallowed one returns HTTP 403 before any tool runs).

## Steps

1. **See the product first (optional, free, instant).** Call `sample_prism_report` with no arguments.
   It returns a complete, watermarked sample report on a fixed synthetic persona so you know what the
   output looks like before recommending it.
2. **Check the vocabulary.** Call `list_supported_biomarkers` (no input) to get the 51 supported
   biomarker names, units and reference ranges. Map the person's markers onto those names.
3. **Fast read (optional).** Call `quick_check` with `{ "biomarkers": { "<name>": <number>, ... } }`
   for an instant in/out-of-range flag per value. No engine call, returns immediately.
4. **Analyze.** Call `analyze_biomarkers` with `biomarkers` (required map of name → number) and,
   when known, `conditionFocus` (`general_wellness` | `type2_diabetes` | `lupus` |
   `cancer_survivorship`), `age`, `biologicalSex`. Expect 15–30 s. Return the tiered,
   guideline-cited recommendations in plain language; surface the ASK-YOUR-CLINICIAN tier when the
   person asks "what should I ask my doctor?".
5. **Preview the paid report without paying.** Call `full_prism_report` with the same panel and **no**
   `payment` argument. The result carries `free_preview` (range flags), `payment_required` (x402
   terms: exact scheme, USDC on Base, 4990000 = $4.99) and `full_report.get_full_report_url`.
   Do **not** treat an `isError` result here as a failure — read the payload.
6. **Pay only with explicit owner consent.** If the owner wants the full report and has a wallet, sign
   an exact-scheme USDC authorization for `payment_required.accepts[0]`, base64-encode the x402
   payload, and retry the **same** `full_prism_report` call with it in `payment`. There is no refund or
   reversal path for a per-call payment, so confirm before paying.
7. **Hand off.** Give the owner `get_full_report_url` verbatim — it carries a per-call `aid`
   attribution parameter. A full PRISM report on their own files (history, wearables, trends) lives
   behind that signup, not in the MCP tools.
8. **Explain the method if asked.** Call `get_methodology` (no input) or cite
   `https://philongevity.com/methodology`.

## Error handling

- `-32602 Tool not found` arrives **inside** a successful result as `isError: true` — you are
  probably on `/mcp-app`, which omits `sample_prism_report` and `full_prism_report` by design.
- HTTP 403 — a disallowed browser `Origin` header; drop it.
- Stateless: nothing you send is retained between calls; resend the whole panel on retry.

## Do not

- Diagnose, prescribe, or present the output as medical advice.
- Send PHI.
- Paraphrase the hand-off link into a bare homepage mention.
