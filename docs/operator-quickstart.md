# Operator quickstart

From a fresh clone to a verified archive. Every step below was run end to end
on 2026-09-25 (macOS, `kbb` on PATH, `git-annex` + `datalad` from Homebrew,
the superproject checked out at `~/github/com-junkawasaki`). The expected
output shown is what those runs printed.

## 1. Clone, and check the checks before touching any content

```bash
git clone git@github.com:cloud-itonami/cloud-itonami-gtm-data.git
cd cloud-itonami-gtm-data
kbb --backend sci --classpath src:test test/gtm_data_verify_test.cljk
```

Expected: `Ran 19 tests containing 34 assertions.` / `0 failures, 0 errors.`
and exit 0. This needs no annex content — it feeds each check a synthetic
violation and asserts the check's own name comes back.

If it dies with `Could not find namespace: gtm-data-verify`, the sources are
not `.cljk` (or `cljk-origin.edn` is missing): the kbb engine resolves `.cljk`
namespaces and nothing else.

## 2. Watch the verifier refuse

```bash
kbb --backend sci --classpath src bin/verify.cljk .
echo $?
```

Expected: exit **2** and

```
  missing content: 2026-07-25/fleet-sales-ads-summary.edn
Fetch it first:  datalad get 2026-07-25
Refusing to report a pass on content this run could not read.
```

A fresh clone holds annex pointers, not bytes. Exit 2 is the correct answer
here; a `0` at this step would mean the evidence floor is broken.

## 3. Fetch the content from B2

The `b2` special remote is `encryption=none`, so the only thing needed is the
B2 key. Resolve it from the superproject (env → 1Password → Keychain; the key
is never written to disk by this step), then enable the remote once:

```bash
eval "$(cd ~/github/com-junkawasaki && kbb --backend sci \
  --classpath '.:orgs/kotoba-lang/secret-resolve/src:orgs/kotoba-lang/text/src' \
  scripts/b2-creds.cljk)"
git annex enableremote b2
datalad get 2026-07-25/fleet-sales-ads-store.edn \
            2026-07-25/fleet-sales-ads-summary.edn \
            2026-07-25/kv-tenant-chunks
```

Expected: `enableremote b2 ok`, then `get (ok: 11)`.

The `eval` and the `get` must run in the **same shell** — without
`AWS_ACCESS_KEY_ID` in the environment `git annex get` fails with
`Unable to access these remotes: b2`. Both classpath entries are required;
leaving either out fails with `Could not find namespace:
secret-resolve.resolver` or `... kotoba.lang.text`, and `eval` of an empty
string then silently sets nothing.

`datalad get 2026-07-25` fetches all ~31 MB including the 625 per-repo
shards; the three paths above are the minimum the verifier needs.

## 4. Verify the archive

```bash
kbb --backend sci --classpath src bin/verify.cljk .
echo $?
```

Expected: exit 0 and

```
SCANNED	store-effects=1270	summary-entries=634	kv-entries=634 (8 chunk files)
COVERAGE	per-repo shards read 0/625 -- 625 shard(s) not fetched; their invariants are UNMEASURED, not clean
RESULT: 0 violation(s)
```

The `COVERAGE` line is not noise: shards you did not fetch were not checked.
Fetch `2026-07-25/per-repo` to raise it to 625/625.

## 5. Release the local copy

```bash
datalad drop 2026-07-25
```

Expected: `drop (ok: 11)`. B2 keeps its copy; only the local bytes go.

This also needs the step-3 `eval` in the **same shell**: `drop` confirms the
B2 copy before deleting the local one, and without the key every file fails
with `No publicurl is configured for this remote` (exit 1, nothing dropped —
safe, but not done).

## Before shipping anything from this archive

Read the README's *Known gaps*: `:to` is empty for all 634 entries, every
`:external-send` effect is still `:proposed`, and sending is blocked on
`RESEND_API_KEY`. Nothing in this dataset is ready to send as-is.
