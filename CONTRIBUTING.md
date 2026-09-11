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

The short version: **`v1.1.0` is a fact, `v1` is a subscription.** `v1.1.0`
names one exact commit permanently, so a caller can freeze and so you can say
what shipped. `v1` is a pointer you re-aim at the newest compatible `v1.x.y`,
so callers pick up fixes without editing anything.

`workflow_call` resolves the ref at run time, which is why moving `v1` takes
effect everywhere on the next run — and why the v1/v2 line has to be drawn on
the caller's contract rather than on how much code changed.

Note what follows from that: **merging to `main` releases nothing.** `v1` moves
only when you move it, so callers are unaffected by anything on `main` until
you do. Docs-only changes often never get a tag at all; they ride along with
the next real release.

### Who pins what

| Pin | Who | Why |
|---|---|---|
| `@v1` | the default for DesignBuilder repos | Fixes arrive automatically; the caller is never edited |
| `@v1.1.0` | a repository that must not change under it | Release-critical, or reproducing an old build |
| `@<40-char sha>` | supply-chain hardening | Immune even to a tag being moved maliciously |

`@v1` is the right default while both ends are ours. Dependabot's
`github-actions` ecosystem can bump pinned refs, which is what makes SHA
pinning practical rather than a maintenance burden, if a consumer ever wants
that.

## Deciding between a minor bump and a new major

The contract is whatever a calling repository has written in **its** YAML.
Break that and it is a new major. Everything else is a minor or patch, and
`v1` moves.

| Change | Tag | Pointer |
|---|---|---|
| Bug fix, no interface change | `v1.1.1` | move `v1` |
| New optional input, new output | `v1.2.0` | move `v1` |
| Breaks the caller's contract | `v2.0.0` | create `v2`; **`v1` stops moving** |

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

`v1` is not deleted when `v2` arrives. It freezes at the last `v1.x.y`, and
anyone pinned to it keeps working on that code indefinitely; it moves again
only if you cut a maintenance release. That is the point of the pointer — a
caller that never upgrades never breaks.

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
you want to try the pinned version somewhere first. Point one repository at
`@v1.2.0`, watch a real release go through, then move the pointer.

Two conventions worth stating, because both are choices rather than
requirements:

- **Only `vN` pointers.** No moving `v1.1`. `actions/checkout` and friends
  publish only the major, and each extra tier is another thing to move
  correctly.
- **Tags, not GitHub Releases.** Nothing here publishes a Release object at
  present. Adding `gh release create v1.2.0 --generate-notes` to step 3 would
  give a changelog answerable from the UI; skipping it keeps the ritual to two
  commands. Either is fine, but do it consistently.

## Once someone pins

The rules above are advisory while nothing references this repository. From the
first real consumer they are not:

- Never move or delete a `vX.Y.Z` tag. The tags were reset once, early on, when
  nothing referenced them; that is no longer available.
- `v1` is *meant* to move — that is why step 4 needs `--force`. Moving it is
  routine, moving `v1.2.0` is not.
- Do not retroactively renumber. A wrong-but-shipped version number is far
  cheaper than a moved one.

## Known duplication

Version-resolution logic is deliberately duplicated between
[`release.yml`](.github/workflows/release.yml) and
[`actions/read-version`](actions/read-version/action.yml). A reusable workflow
cannot `uses: ./actions/...` — when called from elsewhere, `./` resolves
against the *caller's* checkout, not this repository — and pointing it at a
pinned tag of itself creates a bootstrapping problem. Change one, change the
other; the self-test covers both.

## Leave the repository root empty

`owner/repo@ref` resolves to `action.yml` at the repository root, and there is
exactly one root. Putting a building block there would privilege it: the
repository's own name would become the reference for that one action while
everything else kept a path, and `db-github-workflows@v1` would read as though
the repository *is* that action.

This is a collection, and it expects to grow more workflows and more actions.
Keep every block under its own path — `.github/workflows/<name>.yml` for
workflows, `actions/<name>/` for actions — so they all read the same way and
adding the next one changes nothing about the existing ones.

If a single building block ever outgrows this repository enough to deserve the
short form, give it its own repository rather than the root slot here.

## Shared tag space

Everything in this repository shares one set of tags. A breaking change to the
workflow drags the actions to the next major too, even untouched: callers of
`read-version@v1` keep working, but sit on an ageing line until someone cuts
`v1.x` fixes for it. That is acceptable while the repo is small. If components
genuinely diverge, the way out is per-component tags such as
`read-version/v1`, not splitting the repository.
