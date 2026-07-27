# cloud-itonami-gtm-data

DataLad dataset holding the **generated go-to-market data for the
`cloud-itonami` blueprint fleet** — per-vertical sales lists, outreach
drafts and ad/marketing-strategy notes for every
`:maturity :implemented` `cloud-itonami-*` repo.

The content lives in **git-annex + Backblaze B2**, not in git history:
git carries only annex pointers. This is deliberate — each fleet-wide run
produces ~30MB of generated EDN, and committing that to
`gftdcojp/cloud-itonami` (which is cloned by CI, agents and every
operator) would bloat it permanently. See the superproject's
`large-binary-datalad` skill and `manifest/repos.edn` `:b2`.

```bash
datalad get 2026-07-25                 # fetch content from B2
datalad drop 2026-07-25                # release local copy, keep the pointer
```

## Layout

```
2026-07-25/
  fleet-sales-ads-store.edn      # the operational store: ONE langchain.db conn
                                 #   holding all 634 tenants' governed activities
  fleet-sales-ads-summary.edn    # human-readable projection, 634 entries:
                                 #   {:repo :sales-list :outreach :ad-strategy}
  per-repo/<repo>.edn            # 624 per-repo research outputs (one file per
                                 #   blueprint repo, as written by its agent)
  kv-tenant-chunks/chunk-NNN.json# 634 per-tenant conns, pre-shaped for
                                 #   `wrangler kv bulk put` (key store:{org}/{repo},
                                 #   the shape cloud-itonami.edge.workspace-store reads)

2026-07-27/
  per-repo/<repo>.edn            # incremental addendum, NOT a fleet run.
                                 #   Covers repos the 2026-07-25 run missed.
                                 #   Currently: cloud-itonami-iso3166-chn
```

## Provenance (2026-07-25)

Produced by real calls into `gftdcojp/cloud-itonami`'s existing governed
actor — `cloud-itonami.marketing/propose-outreach!` and
`propose-copy-suggestion!` — not hand-written. Every record is
`:status :proposed` / `:risk :external-send`: **nothing here has been
approved or sent.**

Prospect organisations were gathered by live web research, each with a
source URL. **No fabricated names** — agents were instructed to report
fewer targets rather than invent any. A 40-URL random sample checked
2026-07-25 returned 38×200 OK, 1 wrong TLD (`ibstock.com` →
`ibstock.co.uk`) and 1 real-but-down host (`mcra.gov.gh`).

⚠️ **A reachable URL is not a verified claim.** The *fit rationale* and
*facts attributed* to each organisation are unverified LLM output. Review
before any outreach.

Two verticals whose business *is* advertising (`cloud-itonami-isic-7310`,
`cloud-itonami-isic-6312`) additionally got a real advertiser
registration and a `cloud-itonami.adnetwork/propose-campaign-create!`
attempt. Both were **correctly held** by the real Campaign Governor for
lacking a verified USDC deposit — no funding was fabricated to force them
through (ADR-2607110100 defers real advertiser go-live deliberately).

## Provenance (2026-07-27 addendum)

Not a fleet run and not produced by the governed actor. One repo
(`cloud-itonami-iso3166-chn`) researched by hand because the 2026-07-25
run covered only **81 of 223** `iso3166-*` repos and China was not among
them — the 22 occurrences of "China" in the 2026-07-25 summary are
prospect *locations* in other verticals (Goertek, Danieli), not this
repo's own GTM.

Unlike the 2026-07-25 run, every `:source` here states **only what the
fetched page actually confirmed**, and a page that could not be fetched
is recorded as such rather than summarised anyway (the European
Chamber's public-procurement study returned HTTP 405 to an automated
fetch; nothing is claimed about its contents). The `:fit` rationale is
still argument, not fact — review before any outreach, same as above.

Written under superproject ADR-2607277000, alongside the two new CHN
marketing-vertical repos (`cloud-itonami-iso3166-chn-advertising`,
`cloud-itonami-iso3166-chn-market-research`).

## Known gaps

- **iso3166 coverage is 81/223** in the 2026-07-25 run. The 2026-07-27
  addendum closes exactly one of the 142 missing (CHN). The rest remain
  ungenerated.
- `:to` (a real recipient address) is **empty for all 634** — the sales
  lists carry organisation names and URLs, not contacts. Nothing can be
  sent as-is.
- Sending is additionally blocked on `RESEND_API_KEY` (owner action, see
  ADR-2607185000).
- Of the 634 per-tenant conns, only
  `store:cloud-itonami/cloud-itonami-cofog-04.5` has been loaded into the
  production KV (`ITONAMI_DATA`) as a verified canary; the remaining 633
  chunks are staged here, not loaded.

## License

Same terms as `gftdcojp/cloud-itonami`.
