# Verification rubric and false-positive catalog

## Confidence rubric (give to each verifier verbatim)

Score the candidate issue from 0 to 100 for how confident you are that it is a real, reportable
problem. Double-check the claim against the diff excerpt before scoring — the finder was rewarded
for recall, you are rewarded for precision.

- **0 — Not confident at all.** A false positive that doesn't stand up to light scrutiny, or a
  pre-existing issue not introduced by this change.
- **25 — Somewhat confident.** Might be real, but you could not verify it from the evidence. If
  the issue is stylistic, it is one no documented standard calls out.
- **50 — Moderately confident.** Verified real, but it's a nitpick or unlikely to happen in
  practice; relative to the rest of the change it is unimportant.
- **75 — Highly confident.** Double-checked and very likely real, will be hit in practice, and
  the change's current approach is insufficient — or the issue is explicitly named in a
  documented repo standard.
- **100 — Absolutely certain.** Double-checked, definitely real, will happen frequently; the
  evidence directly confirms it.

Return the score and a one-line justification. **Only findings scoring ≥80 survive.**

## False-positive catalog (give to the Bugs finder AND the verifiers)

These do not count as findings:

- Pre-existing issues — problems on lines the change did not modify, or that existed before the
  fixed point.
- Anything a linter, typechecker, compiler, or test suite would catch (missing imports, type
  errors, formatting). Gates run separately; do not run them yourself.
- Pedantic nitpicks a senior engineer wouldn't raise in review.
- General quality wishes (more test coverage, better docs, generic security posture) unless a
  documented repo standard explicitly requires them.
- Issues a documented standard flags but the code explicitly silences (e.g. a lint-ignore
  comment) — the silencing was a decision.
- Behaviour changes that are clearly intentional and within the scope of the broader change.
- Speculative "what if" scenarios with no concrete path from the evidence in the diff.
