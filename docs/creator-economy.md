# VRChat Creator Economy: In-Depth Report (2025–2026)

---

## 1. Foundations: VRChat Credits, Tilia, and Payouts

### VRChat Credits

The platform's premium currency. Users buy bundles through the client (Main Menu → Marketplace → Wallet) paying via **Steam, Meta, or Google Play**:

| Credits | USD | | Credits | USD |
|---|---|---|---|---|
| 600 | $4.99 | | 4,800 | $39.99 |
| 1,200 | $9.99 | | 6,000 | $49.99 |
| 2,400 | $19.99 | | 9,000 | $74.99 |
| 3,000 | $24.99 | | 12,000 | $99.99 |
| 3,600 | $29.99 | | | |

Effective purchase rate ≈ 120 credits per $1. **Bought Credits** vs **Earned Credits**: bought credits spendable on other sellers' content but never payout-eligible; if a wallet holds both, bought are spent first.

### Tilia (now Thunes) onboarding

Tilia — VRChat's payment partner (rebranded under Thunes in 2025) — handles accounts, wallets, payments, payouts. Flow (creators.vrchat.com/economy/getting-started):
1. vrchat.com → My Store → Storefront → "Get Started with Tilia".
2. Agree to ToS/Privacy, submit KYC identity verification.
3. Wait ~5 business days (rarely up to 30).
4. Once accepted, the Storefront shows "Authorized"; earnings from then on are payout-eligible.

Payouts via **PayPal** (other methods hinted for the future), usually processed within 1–2 business days; rejections most often = PayPal/Tilia name mismatch or unsupported country.

### Fees, rate, and revenue share

Per the [Creator Economy Program Rules](https://hello.vrchat.com/legal/economy):
- **Exchange rate:** $0.005 per Earned Credit (200 credits ≈ $1).
- **Seller Transaction Fee:** ~15.3% per sale (rounded up to nearest whole credit).
- **Payout Transaction Fee:** varies by payout method.
- **Net:** roughly **50% of what users spend** reaches the seller. Published split ≈ 50% creator / 30% platform store / 20% VRChat + partners.
- **Payout minimum:** $100 (20,000 Earned Credits); no payout requests within two weeks of the previous; requests can take up to 30 days per the Rules (practice: 1–2 days).

The Rules stress: a payout is legally a *revenue share* for services, not currency conversion; VRChat may modify or cancel the program, change the rate, or deny payouts at its discretion.

### Eligibility and application

Per CE Rules §3 + wiki: 18+; account active ≥30 days; verified email; good standing (generally no suspension/ban in prior 12 months); compliance with CE Rules/ToS/Community Guidelines; no scamming, off-platform credit exchange, or illegal activity; tax forms + completed KYC. Must have uploaded content and VCC access. **Avatar Marketplace is a separate application** from the worlds/groups Creator Economy.

### Taxes

Payouts = payment for services. Sellers solely responsible for taxes; VRChat withholds nothing. US sellers: W-9 + **1099**. Non-US: not reported to home countries.

## 2. Avatar Marketplace

Launched mid-2025, deliberately gated — VRChat accepts a limited number of sellers ("high-quality avatars are more likely to be accepted"), reviewing each seller and each avatar individually.

**Listing requirements & moderation:**
- Upload via SDK, submit on the website. **Every listing AND every update** passes moderation before going live.
- Performance must rank **Excellent/Good/Medium/Poor**; "Very Poor" case-by-case.
- High visual/technical quality; cross-platform versions must look similar; expression menus bug-free.
- **No NSFW**; must meet public-avatar content guidelines.
- No movement cheats via colliders/stations (GoGo Loco yes, GoGo Flight no).
- Updates limited to bug fixes, performance, new features; material redesigns of a purchased base require contacting the Economy Team ("don't turn a dolphin into a squid").
- PC-only allowed but platform support is shown to buyers and incentivized.

**Pricing & revenue:** floor **≥1,200 credits per distinct avatar** (variants don't multiply); general range 100–100,000. Sellers earn ~50% of spend. **No exclusivity** — Booth sellers can also list. No revenue splitting/co-author splits yet (Acknowledgements field credits collaborators). Sellers can unpublish but cannot revoke existing buyers' access. Buyers cannot extract/download purchased avatars via the SDK.

**What sells:** browse categories (Popular, Anime, Furry, Human, Sci-Fi, Fantasy, Realistic, Pop Culture, Fashion) signal demand: stylized anime/furry/humanoid avatars with strong visual identity, cross-platform builds, variant systems. Presentation matters — listings show images, previews, rank, platform support, tags, file size.

## 3. World Stores and Udon Product Gating

Three store types: **Avatar Marketplace**, **World stores** (world page → Store tab), **Group stores** (group page). Created in Store Manager (vrchat.com → My Store → Storefront → Stores).

**Listings bundle 1–15 products** of three types:
- **Avatar products** — unlock uploaded avatars; permanent listings only.
- **Udon products** — unlock world-side benefits (VIP areas, cosmetics, items, currency); any store/listing type.
- **Subscription role products** — group roles; subscriptions only.

**Listing duration types:** **Permanent** (lifetime), **Temporary** (5 min – 3 months, repurchasable), **Instant** (immediate Udon-delivered reward; must be in-world to buy; quantity purchases up to 99 via `OnPurchaseConfirmedMultiple`).

**Store UX patterns:**
- Same listing in multiple world stores.
- **Udon store prefabs** open a listing's purchase screen directly in-world — even for unpublished stores — keeping players in the experience. Standard for spawnables, VIP doors, boosts, trials.
- Non-payers must not get a strictly negative experience: gate content, but explain why and/or offer a purchase path.
- No lotteries/raffles/gambling with real currency.
- Costs always displayed as "VRChat Credits"/"Credits"/credit icon/"\V" — never local currency.
- Test repurchases by revoking the purchase on the Revenue page (My Sales).

## 4. Group Subscriptions

The original CE feature (open beta Nov 2023), still the primary group monetization tool.

- **Structure:** subscription → subscription role product (auto-created) → subscription **role** + up to 10 total products (role + Udon).
- **Pricing:** 100–10,000 credits/month, 1/3/6/12-month plans (wiki documents custom durations up to 99 months). Sellers discount multi-month plans for retention.
- **Role permissions:** view members, enter group instances, **instance queue priority**, "Enable Free Join for Subscribers" (instant join of invite-only groups on subscribe). Udon products layer VIP rooms, supporter vehicles, event access, nameplate icons.
- **Retention:** subscriptions **do not auto-renew** (recurring billing planned, not live). Users get expiry notifications and can extend — sellers lean on in-world renewal prompts and visibly-expiring perks. First-month credits immediate; multi-month revenue month-by-month; cancellations stop future delivery.

## 5. The Gifting Economy

- **Gifting listings:** any listing giftable to a friend or someone in your instance; gifting from the Shop requires Trust Rank "User".
- **Gift Drops:** VRC+ subscribers drop gifts into an instance for others to claim.
- **Hype Trains:** introduced June 1, 2026 (Pride), removed July 1, 2026 — instance-wide collective gift-drop chain (goal 50, 10-min timer reset per drop) unlocking a reward for everyone. Its removal suggests VRChat is still iterating on collective gifting; gift drops persist.
- No formal direct "tips" — support flows through listings, subscriptions, gift drops. Credits may not be used for **fundraising**; donations must be handled off-platform in fiat.

## 6. Legal: Creator Economy Program Rules

Key clauses:
- **Unilateral modification/cancellation (§2):** VRChat may change eligibility, rates, or cancel at any time.
- **Credit withholding (§4):** credits blocked during disputes/suspected violations; negative balances offset future payments. Prepaid content pays out over time — cancellation forfeits the remainder.
- **Revenue-share framing (§5):** payouts are discretionary revenue shares, not currency conversion.
- **Ongoing service obligations (§6):** keep content compliant, reasonable efforts to update for security/platform/hardware compatibility, assist with **DMCA takedowns** (copyright policy: hello.vrchat.com/copyright). Opt-out via economy@vrchat.com.
- **Payout discretion (§7–8):** $100 minimum, two-week spacing, possible limits by account age/history, geographic restrictions.
- **Refunds/chargebacks:** refunds case-by-case, processed **by VRChat only** — sellers cannot issue refunds directly; refunds can push balances negative. Chargebacks flow back against credit balances.
- **Prohibited:** NSFW avatars, gambling/real-currency lotteries, dark patterns, false advertising, off-platform price steering, CE as a marketing funnel for external monetization, credit-for-cash exchange. Ownership: only sell content you fully own (ToS §8.4), including third-party assets — the anti-piracy clause, 10-day remediation window on violations.

## 7. Seller Workflows

- **Publishing pipeline:** create product → listing/subscription → stores/visibility (draft/publish) → moderation review → live. Edits to live avatar listings re-enter review.
- **Analytics:** Revenue/My Sales page (sales, credits earned); per-listing revocation for testing; buyer purchase history. No rich funnel/retention analytics comparable to Booth telemetry — sellers supplement with Discord signals.
- **Compatibility obligation:** §6 "reasonable efforts" = SDK/Unity upgrades and new hardware effectively require keeping listings buildable; broken avatars are takedown risk.
- **Support:** economy@vrchat.com, ask.vrchat.com feedback boards, avatar-seller Discord.

## 8. Case Patterns: What Successful Sellers Do

- **Bundle strategy:** avatar+Udon perks in one listing lifts average order value and satisfies the per-avatar floor elegantly; variants keep listings performant.
- **Cross-platform coverage:** PC + Quest + iOS with visual parity captures more buyers, more filter surfaces.
- **Marketing off-platform:** a 2024 academic study of Booth found X (Twitter) campaigns directly drove sales spikes (a major update tweet at 240k impressions took daily sales from <5 to 85; ~36% average like-to-purchase conversion). Same playbook for Marketplace: Discord update pings, X/Bluesky showcases, Booth presence for source-file buyers.
- **Update cadence as marketing:** regular visible improvements sustain the "Last Updated" sort.
- **Community-first subscriptions:** role (queue priority, member visibility, free-join) + Udon perks, discounted 6/12-month plans to fight no-auto-renew churn.
- **In-world point of sale:** Udon store prefabs at the moment of desire convert far better than hoping users visit the website store.

## Sources

- https://hello.vrchat.com/legal/economy · https://hello.vrchat.com/copyright
- https://creators.vrchat.com/economy/ (+ getting-started, listings, subscriptions, products, guidelines)
- https://help.vrchat.com/hc/en-us/articles/23255458227987 · https://help.vrchat.com/hc/en-us/articles/41810354825235
- https://wiki.vrchat.com/wiki/Creator_Economy · /wiki/Avatar_Marketplace · /wiki/Shop · /wiki/Hype_Train
- https://ask.vrchat.com/t/avatar-marketplace-faq-for-sellers/43555
- https://www.jstage.jst.go.jp/article/vconf/2024/0/2024_23/_pdf/-char/ja (Booth/X promotion study)
