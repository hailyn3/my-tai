# Failure Signatures → Root Causes

Classify the FULL failure list against this table before editing anything.
Strip ANSI (`sed 's/\x1b\[[0-9;]*m//g'`) first — colored output hides the
patterns.

| Log signature | Likely root cause | Fix at the source |
|---|---|---|
| FK violation (`SQLSTATE 23503`) on a lookup table the test never mentions | Sequence drift: factory-seeded rows landed at id≥2 while the test hardcodes id=1 | Explicit ids + `setval(...)` resync in shared setUp |
| Only the first test in the run fails, rest green | Same sequence drift (later tests inherit the resynced state) | Same as above |
| `Undefined array key 0` / `array given` fed to a typed call on a helper result | Helper returns an assoc array; call site uses positional destructure or plain assignment | Named destructure at every call site, or keep both key styles |
| Test flips green/red across runs of the same code | Order-dependent test: random execution order + magic-id or sequence-state assumption | Loop-run to confirm, pin sequence in setUp past the magic id |
| `assertSee`/`assertEquals` fails with a factory-generated value as actual | Assertion hardcodes data the factory randomizes (names, coords) | Assert the created model's real attribute |
| Count/feature query returns 0 with rows clearly present | Randomized data falls outside a fixed filter window (bbox, range) | Pin values inside the window at creation |
| Guard/regression test passes on first run | Not yet proven — it has never observed its own failure | Mutate the implementation to a no-op, watch the guard fail, revert |
| Local green, remote red | Base moved under the PR (merge-ref CI) or CI-only flags (`--parallel`, `--coverage`, runtime version) | Sync base first, then reproduce with CI's flags |

## Flake confirmation loop

```bash
for i in 1 2 3 4 5; do
  XDEBUG_MODE=off php artisan test tests/Feature/SuspectTest.php 2>&1 | grep -a "Tests:"
done
```

Any variance across runs = order dependence, not a logic bug. A file that
passes alone but fails in the full suite is consuming state left by an
earlier test (or an un-reset sequence).
