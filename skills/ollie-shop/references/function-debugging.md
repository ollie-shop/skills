# Hub Functions — Failure Modes & Debugging

The diagnosis counterpart to [hub-functions.md](hub-functions.md) (authoring). A function fails
differently from a component: there is no browser console, no stack in the UI, and **several
platform limits fail silently** — a write that does not persist, an invocation that dies
mid-handler. A failing function can look identical to a function that was never invoked.

Use this file when a function is deployed and not doing what it should. For *writing* one, stay
in `hub-functions.md`.

---

## 1. Triage order

Each step rules out a whole class of cause before the next one costs time. Most investigations
end at step 1 or 2.

1. **Was it invoked at all?** Check `active`, that the trigger URL matches, and that the
   expression evaluates true against the real payload. A too-narrow expression means the Lambda
   never ran — silence is then the *correct* output, not a bug.
2. **Did it run to completion?** A timeout kills the handler where it stands, so a function that
   died mid-run and one that returned cleanly can look the same from outside. Confirm by whether
   the **effect** landed — the response was rewritten, the value persisted — not by the absence
   of an error.
3. **Did what it wrote persist?** Anything written to `sessionStorage` needs a read-back to be
   trusted; the write reports success either way (§3).
4. **Is the input what you think it is?** Env values, header casing, and body shape after the
   previous function in the chain — a function that reads a value the admin never delivered
   fails in a way that points at the value, not at the plumbing.
5. **Only then read the handler.**

---

## 2. Symptom → cause → how to confirm

| Symptom | Likely cause | Confirm it |
|---|---|---|
| Consumer renders empty; response is `200`; no error anywhere | `sessionStorage.setItem` silently discarded the value — payload over the ceiling (§3). It does **not** throw, so the write appears to succeed. | Read the key back immediately after writing and compare lengths. That is the only detector. |
| `TimeoutError: [AbortError]: The operation was aborted` at `timeoutEarlyResponse` | The invocation exhausted the platform time budget. Everything the handler had not yet emitted is lost. | Compare elapsed time against the ceiling in §3. Landing on the ceiling means the budget, not your code. |
| A signal you expect is missing, so "that branch did not run" | Indexing lag in the log backend — a line can surface **~10s after** it was emitted, and concurrent invocations interleave with nothing to regroup them. | Re-query before concluding. **Absence does not prove non-execution.** |
| Upstream rejects an id that looks correct; the error message shows a doubled space | An env var saved with leading or trailing whitespace. It is invisible in the admin and survives copy/paste. | Compare the value's length against the expected length. `.trim()` on read. |
| Two functions behave as if running different versions of the same shared file | A deploy bundles the **whole** functions tree and only selects the entrypoint (§3). Deploying one leaves the others on the old shared code. | Compare `bundleSizeBytes` across functions — near-identical sizes confirm it. Redeploy every function that imports the changed file. |
| Chained functions: the second one sees an unexpected body | A higher-priority function returned a `Response` and stopped the chain, or rewrote the body it forwards. | Check `priority` across every function matching that endpoint — see [hub-functions.md](hub-functions.md) § Priority and chaining. |
| CLI: `Unexpected token 'U', "Unauthorized" is not valid JSON` | The session expired; the CLI parses a text body as JSON. | `ollieshop whoami`. Re-login, then **re-verify every step of an interrupted deploy sequence** — a sequence is not atomic and may be half-applied. |
| Deployed, but the target environment shows no change | The command ran without `--stage`, so it resolved `ollie.json` — production. | `ollieshop whoami --stage <stage>` before and after. Treat a missing `--stage` as a production deploy until proven otherwise. |
| `Node.js 20 detected without native WebSocket support` | The CLI needs native `WebSocket`. It installs fine on Node 20 and fails only at command time. | Check `node -v` before blaming the command. |

---

## 3. Runtime limits — measured, not specified

None of the first four are returned by any API or documented in a spec; they were measured in the
field. **Re-measure before trusting them** — the time budget has already changed once with no
release note.

| Limit | Value | Behavior at the limit |
|---|---|---|
| `sessionStorage.setItem` payload | **~2 KB** | **Silent.** No throw, no error, no signal. The value simply does not persist, and the next reader sees an empty cache. |
| Invocation time budget | **~10s** | `TimeoutError` at `timeoutEarlyResponse`. The handler dies where it stands. Not returned by `function list` — there is no way to query it. |
| Timeout visibility | none | The runtime's own timeout and duration records are not forwarded to the log backend. A timeout cannot be confirmed from the dashboards — only inferred. |
| Deploy bundle | the whole functions tree | `--name` selects the entrypoint, not the bundle contents. |
| Admin zip upload | 10 MB | Rejected above. |

**Sizing an internal timeout against an unqueryable ceiling.** Both naive choices fail: a cap
equal to the platform budget never fires (the runtime always wins the race), and a cap inside the
call's normal range cuts healthy traffic. Pick one **above the call's p99 and below the platform
ceiling**, and record the elapsed time on every abort so the cap gets re-tuned from data rather
than re-guessed.

**Staying under the storage ceiling** is a design problem, not a debugging one — what to put in
the payload, what never to spread into it, and how to compress it:
[patterns/session-storage-payload-budget.md](patterns/session-storage-payload-budget.md).

---

## Related

- [hub-functions.md](hub-functions.md) — authoring: invocation phases, triggers, chaining, README contract
- [patterns/session-storage-payload-budget.md](patterns/session-storage-payload-budget.md) — keeping a shared cache under the ceiling
- [patterns/resolver-enricher-enforce-with-cookies.md](patterns/resolver-enricher-enforce-with-cookies.md) — the multi-function chain most of these failures surface in
- [cli-reference.md](cli-reference.md) — `--stage`, deploy flows, `.ollieignore`
