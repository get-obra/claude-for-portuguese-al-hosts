# Examples

End-to-end worked transcripts showing how the skills in this pack compose into real workflows. Each example walks through a complete scenario with synthetic but representative data, calling out where the AI proposes, where the host signs off, and what lands in the audit chain.

## Planned examples

| # | File | Scenario |
|---|---|---|
| 01 | `01-french-guest-arrival.md` | A French-speaking guest books for next week. Pre-arrival welcome -> passport extraction with agreement gate -> SEF submission. Host signs off at each gate. |
| 02 | `02-wifi-question-saturday-night.md` | A guest messages the host at 11 PM on a Saturday asking for the wifi password. In-stay support handles routine; escalates a follow-up complaint about noise. |
| 03 | `03-month-end-accountant.md` | End of the month. Monthly record generated, host approves figures, accountant signs off. |

## What each example shows

For each scenario, the example file shows:

- The trigger (what brought the work in)
- Every skill invocation in order
- The structured payloads being passed between skills
- Each gate (proposal, adversarial review, human confirmation) with what was checked
- The audit chain events that landed
- The total wall-clock time and token cost (approximated against representative pricing)

## Synthetic data only

All data in these examples is synthetic. No real guest names, real passport numbers, real reservation amounts, or real correspondence are used. The intent is to show the workflow shape without anyone's privacy being at stake.

## Status

Active development. The first example transcript (01) will land alongside the first skill files. Contributions welcome once example 01 is published.
