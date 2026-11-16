# CrowdfundAudit — test flow

Two Studio test accounts: A = campaign creator (deployer), B = backer.
Deployed contract address: 0x3240C5df1Cb14d3F683907d769A2AC13eF6cC626

Fixture files must be live on the repo's main branch BEFORE running the assessment steps (validators fetch them at assessment time):
- campaign: https://raw.githubusercontent.com/wallechj/genlayer-crowdfund-audit/main/examples/campaign.md
- updates: https://raw.githubusercontent.com/wallechj/genlayer-crowdfund-audit/main/examples/updates.md
- forum post: https://raw.githubusercontent.com/wallechj/genlayer-crowdfund-audit/main/examples/forum_post.md

## The delivery boundary

Non-delivery can only be claimed once the delivery deadline has passed: before
that, the creator still has time to deliver, so no claim exists yet. The demo
campaign's deadline (30 June 2025, unix 1751241600) is in the past, so claims
on it can be filed immediately. A second campaign with a future deadline
(30 June 2030, unix 1909008000) is used to test the boundary: filing a claim
on it must fail.

## 1. Create campaign

As A, call create_campaign with examples/sample_campaign.json:
- title: "Tidebound — A Co-op Board Game"
- campaign_url: the campaign fixture URL above
- updates_url: the updates fixture URL above
- delivery_deadline: 1751241600
- total_raised: 48500

Expected: returns 0. Read back get_campaign(0): creator = A, total_raised 48500, refundable_amount 0, status "open".

Negative tests (must fail with the exact message):
- campaign_url "ftp://example.com/campaign" → "campaign_url must be an http(s) URL"
- total_raised 0 → "total_raised must be greater than zero"
- delivery_deadline 0 → "delivery_deadline must be greater than zero"

## 2. Submit claims 0–2

As B, call submit_claim three times (examples/sample_claim_game.json, sample_claim_soundtrack.json, sample_claim_shirt.json):
- 0: "Collector's Edition never shipped", $120, evidence forum_post.md, severity high
- 1: "Digital soundtrack never delivered", $25, evidence updates.md, severity medium
- 2: "T-shirt never delivered", $35, evidence updates.md, severity medium

Expected: returns 0, 1, 2. Read back get_claim(i): status "submitted".

Negative tests:
- as A, submit a claim → "campaign creator cannot submit claims"
- pledge_amount 0 → "pledge_amount must be greater than zero"
- pledge_amount 50000 (over total raised) → "pledge_amount cannot exceed total raised"
- evidence_url "ftp://example.com/x" → "evidence_url must be an http(s) URL"
- claimed_severity "huge" → "claimed_severity must be one of critical/high/medium/low"

## 3. Assess claims 0–2

Call assess_claim(0), assess_claim(1), assess_claim(2) from either account.

Expected:
- 0: valid/high — the campaign page says Collector's Edition shipping is delayed indefinitely with no tracking numbers, and Update #15 confirms no ship date. get_refundable(0) = 120.
- 1: invalid — Update #12 posted the soundtrack download link for all backers in April 2025; it was delivered. Refundable stays 120.
- 2: invalid — Update #14 says all T-shirts shipped after the size survey, with tracking numbers issued. Refundable stays 120.
- Severity invariant holds on all three: valid carries a real bucket, invalid carries "none".

## 4. Repeat claims

As B, submit two more claims on campaign 0:
- 3: same as 0 (repeat of the accepted non-delivery finding), $120, evidence forum_post.md, severity high (examples/sample_claim_repeat_accepted.json)
- 4: same as 1 (repeat of the rejected soundtrack claim), $25, evidence updates.md, severity medium (examples/sample_claim_repeat_rejected.json)

Expected: returns 3, 4.

Call assess_claim(3), assess_claim(4).

Expected:
- 3: invalid as a duplicate — it matches prior accepted finding 0. Refundable stays 120.
- 4: invalid on its own grounding, NOT called a duplicate — the prior claim 1 was rejected, so it is not duplicate authority. Refundable stays 120.

## 5. Second campaign — future deadline (boundary test)

As A, call create_campaign with examples/sample_campaign_future.json:
- title: "Starlight Dice Tower"
- campaign_url: the campaign fixture URL above
- updates_url: the updates fixture URL above
- delivery_deadline: 1909008000
- total_raised: 20000

Expected: returns 1.

As B, submit a claim on campaign 1 (same inputs as claim 0 but campaign_id 1).

Expected: ERROR with "delivery deadline has not passed" — the creator still has until 2030 to deliver.

## 6. Third campaign — dead campaign page

As A, call create_campaign with examples/sample_campaign_dead.json:
- title: "Phantom Card Game"
- campaign_url: https://raw.githubusercontent.com/wallechj/genlayer-crowdfund-audit/main/examples/nonexistent_campaign.md (does not exist)
- updates_url: the updates fixture URL above
- delivery_deadline: 1751241600
- total_raised: 10000

Expected: returns 2.

As B, submit claim 5 on campaign 2 (examples/sample_claim_dead.json): "Rewards claimed delivered with no campaign page", $100, evidence forum_post.md, severity high. Expected: returns 5.

Call assess_claim(5). Expected: invalid — the campaign page could not be retrieved, so nothing corroborates the claim and the campaign itself cannot be verified. Refundable of campaign 2 stays 0.

## 7. Close and reopen

Negative test: assess_claim(0) again → "claim has already been assessed".

As A, call close_campaign(0). Expected: null; get_campaign(0).status = "closed".

Negative test: as B, submit_claim(0, ...) on the closed campaign → "campaign is not open for claims".

As A, call reopen_campaign(0). Expected: null; get_campaign(0).status = "open".

## 8. Final read-backs

- get_campaign_count() = 3
- get_claim_count() = 6
- get_refundable(0) = 120 ($120 upheld of $48,500 raised)
- get_refundable(1) = 0
- get_refundable(2) = 0
- get_claim(0): status "valid", ruled_severity "high"
- get_claim(1)–(5): status "invalid", ruled_severity "none"
- get_claim_evidence(0) names the campaign page and Update #15

## Expected transaction count

Deploy (1) + successful calls (create 0, submit 0–2, assess 0–2, submit 3–4, assess 3–4, create 1, create 2, submit 5, assess 5, close, reopen = 17) + negative tests (11) = 29 transactions: 18 SUCCESS, 11 intentional ERRORs.
