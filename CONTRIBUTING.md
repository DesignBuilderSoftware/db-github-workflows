# Contributing

Everything here is consumed by other repositories at run time. A bad tag does
not break a build in this repo — it breaks whichever repo happens to release
next. That single fact drives all of the conventions below.

## Refs, and what each one is for

```
main                    development; always the newest major
release/v1              maintenance; created lazily, only once v2 exists
v1.0.0, v1.1.0, v2.0.0  immutable release tags; never move
v1, v2                  moving pointers; this is what callers pin
```

Callers pin `@v1`. You re-point `v1` at each backwards-compatible release, so
they pick up fixes without editing anything. `workflow_call` resolves the ref
at run time, which is why moving `v1` takes effect everywhere on the next run —
and why the v1/v2 line has to be drawn on the caller's contract rather than on
how much code changed.

## Deciding between a minor bump and a new major

The contract is whatever a calling repository has written in **its** YAML.
Break that and it is a new major. Everything else is a minor or patch, and
`v1` moves.

Forces a new major:

- Renaming or removing an input, or making an optional input required.
- Changing an input's default such that behaviour changes — e.g. `tag-prefix`
  from `v` to `''`.
- Renaming an output, or changing what it means.
- Requiring a new secret.
- Requiring broader `permissions:` on the caller's job.
- Renaming the internal `release:` job id.
- Raising the runner floor, e.g. now needing a newer Python than `tomllib`
  currently requires.

Stays on the current major:

- Adding an optional input whose default preserves existing behaviour.
- Adding an output.
- Fixing a bug, internal refactors, better logging, step-summary changes.
- Bumping a pinned action, e.g. `actions/checkout@v4` to `@v5`, as long as
  runner requirements do not change.

Two of the breaking cases are easy to miss because the diff looks additive:

- **Caller permissions.** Add a step needing `packages: write` and every caller
  must edit their workflow before it works. A called workflow can never hold
  more permission than the caller's token grants.
- **The job id.** The check name a caller sees is `<their job> / release`.
  Renaming `release:` silently breaks anyone's branch-protection required
  checks.

The grey area is a bug fix somebody might be relying on. Rule of thumb: if the
current behaviour is defensible as intentional, treat changing it as breaking;
if nobody could sanely want it, patch it on the current major.

## How the branches split

Nothing about `main` changes when a new major arrives — `main` is simply always
the newest major. The asymmetry is on the old line.

**Before v2 exists.** Branch off `main`, merge, tag `v1.x.y`, move `v1`. One
line, no maintenance branch.

**The moment `v2.0.0` is tagged from `main`.** `main` now holds v2 code, so a
v1 patch can no longer be cut from it. Create `release/v1` from the last v1
commit — do this lazily, at the first v1 fix you actually need, not
pre-emptively.

**After that.** v1 fixes land on `release/v1`, tag `v1.x.y`, move `v1`. v2 work
lands on `main`, tag `v2.x.y`, move `v2`. Fixes affecting both get
cherry-picked, normally main first.

## Testing a change before tagging it

[`.github/workflows/self-test.yml`](.github/workflows/self-test.yml) exercises
the building blocks against [`tests/fixtures`](tests/fixtures). Within a
repository a workflow may reference a sibling with a local path, and `./`
resolves against the current checkout, so on a branch the self-test runs
*that branch's* definitions. Open a pull request, or dispatch it manually:

```bash
gh workflow run self-test.yml --ref my-branch
```

It covers version resolution for all four file shapes, asserts that unusable
inputs are refused rather than silently resolving to nothing, and runs
`release.yml` end to end down the path that creates nothing — `v1.0.0` is
already tagged here, so it must skip. That job is granted only
`contents: read` on purpose: if the skip logic ever regresses, creating a
release fails loudly instead of publishing something unintended.

What the self-test does **not** cover:

- The release-creation path itself. Nothing verifies a real `gh release create`
  without publishing a real release, so that stays manual.
- The `token` secret fallback. `${{ secrets.token || secrets.GITHUB_TOKEN }}`
  is evaluated on the create step, which the skip path never reaches.
- `tag-prefix: ''`. Whether an explicitly empty `with:` value overrides an
  action's declared default or falls back to it is an Actions sharp edge;
  confirm the behaviour before relying on bare `1.2.3` tags.

For anything beyond that, point a scratch repository at a branch or a full
SHA — callers can pin any ref, not just tags:

```yaml
uses: DesignBuilderSoftware/db-github-workflows/.github/workflows/release.yml@my-branch
```

## Cutting a release

1. Merge to `main` (or to `release/vN` for a maintenance fix).
2. Confirm the self-test is green on that commit.
3. Tag the immutable version and push it:

```bash
git tag -a v1.1.0 -m "v1.1.0" && git push origin v1.1.0
```

4. Move the major pointer:

```bash
git tag -f v1 v1.1.0 && git push origin --force v1
```

Step 4 is the one that reaches every caller, so leave a gap between 3 and 4 if
you want to try the pinned version somewhere first.

## Known duplication

Version-resolution logic is deliberately duplicated between
[`release.yml`](.github/workflows/release.yml) and
[`actions/read-version`](actions/read-version/action.yml). A reusable workflow
cannot `uses: ./actions/...` — when called from elsewhere, `./` resolves
against the *caller's* checkout, not this repository — and pointing it at a
pinned tag of itself creates a bootstrapping problem. Change one, change the
other; the self-test covers both.

## Shared tag space

`release.yml` and `actions/read-version` share one set of tags. A breaking
change to the workflow drags the action to the next major too, even untouched:
callers of `read-version@v1` keep working, but sit on an ageing line until
someone cuts `v1.x` fixes for it. That is acceptable while the repo is small.
If the two genuinely diverge, the way out is per-component tags such as
`read-version/v1`, not splitting the repository.
