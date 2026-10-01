# Browser State Instrumentation — SPA / page-global bug reproduction

Use when a bug depends on page-global state (`window.*`), instance identity, or only appears after in-app (SPA) navigation. Goal: turn "it broke after navigating" into an ordered, evidenced timeline.

## Hook before you navigate

- **State writes, not just reads.** For each suspect global:
  ```js
  let _v = window.someGlobal;
  Object.defineProperty(window, 'someGlobal', {
    configurable: true,
    set(v) { record('assign', { value: v, connected: v?.getContainer?.().isConnected }); _v = v; },
    get() { return _v; },
  });
  ```
  A `delete window.someGlobal` later removes the accessor — subsequent reads still behave correctly, and reads going undefined after a set is itself timeline evidence.
- **Factories/functions** — wrap to capture WHO created state and the world-state at that moment:
  ```js
  const orig = L.layerGroup;
  L.layerGroup = (...a) => {
    record('create', { stack: new Error().stack, mapConnected: window.map?.getContainer().isConnected });
    return orig(...a);
  };
  ```
- **Element identity** — before navigating: `el.setAttribute('data-probe', '1')`. After: compare the attribute (node reused?) and `el.isConnected` (node replaced/disconnected?). This distinguishes "same DOM" from "morph created a new node".
- **Coarse sampling** — a ~100ms `setInterval` recording globals + key DOM counts, installed before navigation, for context around the hooked events. Dump with change-compression: keep only the first sample of each unique state so the interleaving is readable.

## Why hooks beat polling

An assign-then-delete can happen entirely inside one sampling interval — polling records only "before" and "after" (both "undefined") and misses that a value ever existed. Setters record order; polling records state. Use both: setters for ordering, polling for coarse context.

## Assert behavior, not just presence

```js
const rects = () => [...document.querySelectorAll('.target')].map(el => {
  const r = el.getBoundingClientRect(); return [Math.round(r.x), Math.round(r.y)];
});
const before = await rects();
await page.evaluate(() => window.map?.panBy([120, 40], { animate: false }));
await page.waitForTimeout(400);
const after = await rects();   // compare: did every element move as expected?
```

Same pattern for zoom (`zoomIn()`), drag, form re-submit — anything that should move or update the elements under test. Presence-only assertions pass on frozen, duplicated, and wrongly-bound elements.

## Read the interleaving

Combine: navigation timestamp + hooked events (with captured state) + coarse samples. Root cause = ordering statement: "callback X ran against the detached instance (isConnected=false) at t1; init Y reassigned the global and deleted the layers at t2 > t1". If two different orderings would produce the same final state, the hooks are what disambiguate — state the captured state at each event, never infer it from the end.

## Output shape

1. Scenario matrix: entry path × outcome (red/green), with URLs logged.
2. Timeline table: t-offset, event, source `file:line`, captured state.
3. Console snippet for the reporter to classify their variant on their own environment.
4. Explicit reproduced-vs-inferred labels.
