# CrowdfundAudit

On-chain verification for crowdfunding delivery. A campaign creator registers a campaign with its campaign page and updates page; backers can claim a promised reward was never delivered. Validators fetch the campaign page and the updates themselves, rule on each claim, and the refundable amount is a deterministic on-chain sum of upheld pledges.

## Why this exists

Crowdfunding's oldest failure mode is a creator who takes the money and never ships. Backers argue in comment threads; nobody can prove anything. The evidence that settles it lives in two public places: the campaign page (what was promised, and when) and the creator's updates (what actually shipped). CrowdfundAudit puts the check on-chain: validators fetch both, not the backer's word.

## How it works

1. **create_campaign** — the creator registers a campaign, naming the campaign URL, the updates URL, the delivery deadline (unix timestamp), and the total raised. The campaign and updates URLs are declared by the creator at creation, so validators know exactly which documents govern the campaign.
2. **submit_claim** — any backer except the creator can claim a reward was never delivered: a title, a statement, the pledge amount, a supporting evidence URL, and a claimed severity. The pledge cannot exceed the total raised. Claims can only be filed once the delivery deadline has passed — before that, the creator still has time to deliver, so no claim exists yet. This deadline is the non-bypassable boundary: it is enforced in the contract, not by convention.
3. **assess_claim** — validators fetch the campaign page, the updates page, and the backer's evidence page themselves, then decide:
   - **valid** only if the fetched campaign page and updates corroborate the non-delivery — the named reward tier exists, its promised ship date is past, and no update shows it shipped or delivered. A claim is never valid on the backer's word alone, and never valid on the backer's evidence page alone: the campaign page and updates must corroborate it. If the campaign page could not be retrieved, or the updates show the reward shipped or delivered, the claim is invalid.
   - **invalid** otherwise — including a claim that matches a rejected prior claim (judged on its own grounding) or a claim that repeats an accepted finding (a duplicate).
   - `valid` must be a true JSON boolean; a valid claim always carries a real severity bucket (critical/high/medium/low), an invalid one always carries "none".
   - The refundable amount is then updated deterministically on-chain: `refundable = sum(upheld pledges)`. No LLM ever touches the money math.
4. **close_campaign / reopen_campaign** — the creator closes or reopens the campaign (no claims or assessments while closed).

Every verdict carries a one-sentence evidence note stored on-chain, naming the fetched material that decided it.

## What validators actually fetch

- The campaign page (declared at campaign creation).
- The creator updates page (declared at campaign creation).
- The backer's supporting evidence page (declared per claim).
- Nothing else. No party-supplied text is treated as proof; the backer's statement is read as an allegation and weighed against the fetched campaign page and updates.

## The demo campaign

`examples/` holds the full demo flow, with fixture files:

| Claim | Claim | Expected |
| --- | --- | --- |
| 0 | Collector's Edition ($120) never shipped | valid/high — campaign page says shipping delayed indefinitely with no tracking numbers; Update #15 confirms no ship date. Refundable $120 |
| 1 | Digital soundtrack ($25) never delivered | invalid — Update #12 posted the download link for all backers in April 2025 |
| 2 | T-shirt ($35) never delivered | invalid — Update #14 says all shirts shipped after the size survey, with tracking numbers |
| 3 | Repeat of claim 0 | invalid as a duplicate — it matches prior accepted finding 0 |
| 4 | Repeat of claim 1 | invalid on its own grounding, NOT a duplicate — the prior claim was rejected |
| 5 | Rewards claimed delivered with no campaign page (third campaign, dead campaign URL) | invalid — the campaign page could not be retrieved |

Refundable $120 of $48,500 raised. A second campaign with a future delivery deadline (2030) proves the boundary: filing a claim on it fails with "delivery deadline has not passed".

## Files

- `contracts/crowdfund_audit.py` — the contract
- `test_flow.md` — step-by-step verification, including negative tests
- `examples/sample_campaign.json`, `examples/sample_campaign_future.json`, `examples/sample_campaign_dead.json`, `examples/sample_claim_*.json` — exact inputs for the demo
- `examples/` — campaign page, updates page, and backer forum post, as plain fetchable files

## Deploy

1. Upload `contracts/crowdfund_audit.py` to GenLayer Studio and deploy.
2. Follow `test_flow.md` with two test accounts (campaign creator, backer).
3. Verify every write with a view read-back (`get_campaign`, `get_claim`, `get_refundable`).

## Verification

- Deployed contract: (fill in after deploy)
- Studio verification: (fill in after deploy — expected 28 transactions: 17 SUCCESS, 11 intentional ERRORs)
- Known boundary (same class as earlier contracts, not fixed): reading a nonexistent campaign or claim fails with a raw `KeyError` (TreeMap lookup) instead of a friendly message — still rejected, not a valid read.
