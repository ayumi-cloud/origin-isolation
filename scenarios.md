# Origin isolation scenarios

This document considers various combinations of documents being loaded, with different relationships to each other and different origin isolation signals, to illustrate the consequences of the [proposed algorithm](./README.md#specification-plan).

Notes:

* We use `https://e.com` as our example domain for brevity. "e" stands for "example" but is short enough to fit into tables.
* All of these considerations apply for popups, not just frames, in the same ways.
* All of these considerations apply for workers, not just documents, in the same ways.
* We use the notation `Origin{}` and `Site{}` to denote the type of agent cluster key.
* We assume that "blocking" origin policies are used at all times, without async updates. (Async updates would just mean that we re-run these scenarios on a future load.)

## Two-document scenarios

| Main frame              | Subframe                 | Result                                                               |
| ------------------------|--------------------------|----------------------------------------------------------------------|
| `https://e.com` w/ OI   | `https://e.com` w/ OI    | Both in `Origin{https://e.com}`                                      |
| `https://e.com` w/ OI   | `https://e.com` w/o OI   | Both in `Origin{https://e.com}`                                      |
| `https://e.com` w/o OI  | `https://e.com` w/o OI   | Both in `Site{https://e.com}`                                        |
| `https://e.com` w/o OI  | `https://e.com` w/ OI    | Both in `Site{https://e.com}`                                        |
| `https://e.com` w/o OI  | `https://x.e.com` w/o OI | Both in `Site{https://e.com}`                                        |
| `https://e.com` w/o OI  | `https://x.e.com` w/ OI  | Main in `Site{https://e.com}`   <br>Sub in `Origin{https://x.e.com}` |
| `https://e.com` w/ OI   | `https://x.e.com` w/o OI | Main in `Origin{https://e.com}` <br>Sub in `Site{https://e.com}`     |
| `https://e.com` w/ OI   | `https://x.e.com` w/ OI  | Main in `Origin{https://e.com}` <br>Sub in `Origin{https://x.e.com}` |

## Three-document scenarios

| Main frame                 | Subframe A                  | Subframe B                 | Result                                                               |
| ---------------------------|-----------------------------|----------------------------|----------------------------------------------------------|
| `https://e.com`<br>w/ OI   | `https://x.e.com`<br>w/o OI | `https://x.e.com`<br>w/ OI | Main in `Origin{https://e.com}`<br>_If A loads first_: A and B both in `Site{https://e.com}` <br>If _B loads first_: A and B both in `Origin{https://x.e.com}` |
| `https://e.com`<br>w/o OI  | `https://x.e.com`<br>w/o OI | `https://x.e.com`<br>w/ OI | Main in `Site{https://e.com}`<br>_If A loads first_: A and B both in `Site{https://e.com}` <br>If _B loads first_: A and B both in `Origin{https://x.e.com}` |
| `https://e.com`<br>w/ OI   | `https://a.e.com`<br>w/o OI | `https://b.e.com`<br>w/ OI | Main in `Origin{https://e.com}`<br>A in `Site{https://e.com}`<br>B in `Origin{https://b.e.com}` |
| `https://e.com`<br>w/o OI  | `https://a.e.com`<br>w/o OI | `https://b.e.com`<br>w/ OI | Main and A in `Site{https://e.com}`<br>B in `Origin{https://b.e.com}` |


## Session history scenarios

The previous scenarios illustrate some cases where the proposed algorithm ignores the origin isolation request because a pre-existing site-keyed agent cluster exists for a same-origin URL in the current frame tree. The following scenario illustrates a case where the session history contributes to this calculation.

### Navigating a subframe

* Main frame: `https://e.com` w/ OI
* Subframe: `https://x.e.com` w/o OI
* Outcome (per above): Main in `Origin{https://e.com}` / Sub in `Site{https://e.com}`

The user clicks a link in the `https://x.e.com` subframe which takes them to `https://e.org`.

The user then clicks a button in the main frame which inserts a second subframe, pointing at `https://x.e.com`. But in the meantime the server operator has deployed origin isolation on `https://x.e.com`. Thus the scenario is now:

* Main frame: `https://e.com` w/ OI
* Subframe 1: `https://e.org` (with `https://x.e.com` in the session history)
* Subframe 2: `https://x.e.com` w/ OI

The outcome for subframe 2 depends now on whether the `https://x.e.com` document in session history was retained in the back–forward cache. (I.e., whether the relevant session history entry contains a non-null `Document`.)

If it was retained, then subframe 2 ends up keyed by `Site{https://e.com}`. This ensures that if the user navigates subframe 1 back, restoring the `Site{https://e.com}`-keyed instance of `https://x.e.com` in subframe 1, that subframe 1 and subframe 2 are still in the same `Site{https://e.com}` agent cluster, i.e. we have avoided isolating same-origin pages from each other.

If it was not retained, then subframe 2 ends up keyed by `Origin{https://x.e.com}`, as there is nothing keyed by `Site{https://e.com}` in the agent cluster map. Then, if the user navigates subframe 2 back, since there is no back–forward cache document stored for `https://x.e.com`, a new one will be created. This new one will go through the process of choosing an agent cluster key, and also end up with `Origin{https://x.e.com}`. So again, we have ensured that subframe 1 and subframe 2 are both in the same agent cluster (this time the `Origin{https://x.e.com}` one), and have avoided isolating same-origin pages from each other. And even better, this time we were able to respect the origin isolation request, since there was nothing in the back–forward cache to prevent us.

### Inserting iframes and saving JS references

* Main frame: `https://e.com` w/ OI
* Subframe: `https://e.org` w/o OI
* Outcome (per above): Main in `Origin{https://e.com}` / Sub in `Site{https://e.org}`.

JavaScript code in the main frame goes through a variety of contortions:

```js
// Save a reference to the sub-frame window.
window.savedFrame = frames[0];

// Now remove it from the DOM.
frames[0].frameElement.remove();
```

Some time later, JavaScript code inserts a new subframe, again pointing at `http://e.org`. But in the meantime the server operator has deployed origin isolation on `https://e.org`. Thus the scenario is now:

* Main frame: `https://e.com` w/ OI
* Subframe: `https://e.org` w/ OI

Will the subframe get site-keyed, or origin-keyed?

The answer is origin-keyed. The `window.savedFrame` variable does mean that the agent cluster map still contains an entry with key `Site{https://e.org}`, which itself contains the saved realm and corresponding `Window` object. However, because the iframe was removed from the DOM, the `Window` has  no browsing context, and thus no session history. This means there is no session history containing an `https://e.org`-origin `Document` within the agent cluster. Thus the request for origin isolation is respected, and we end up with `Origin{https://example.org/}` as the key.


## Worked-out nested scenario

The following is a more detailed scenario, involving more levels of nesting and dynamic insertion of new frames. It illustrates the same fundamental principles as the above examples, but it goes through them in more detail, which might be helpful.

We start out as follows:

* `https://example.org/` embeds an iframe for `https://a.example.com/` which embeds an iframe for `https://b.example.com/1`
* We open tab #1 to `https://example.org/`. Both `https://a.example.com/` and `https://b.example.com/1` responses point to origin policies with `"isolation": true` set.

Now, things get fun:

* While we have tab #1 open, the server operator updates both `https://a.example.com/.well-known/origin-policy` and `https://b.example.com/.well-known/origin-policy` to set `"isolation": false`.
* Then, in tab #1, `https://a.example.com/` inserts a new iframe, pointing to `https://b.example.com/2`. Since the `https://b.example.com/` policy on the server has been updated, the response used for creating this second child iframe is no longer requesting origin isolation.
* This `https://b.example.com/2` iframe inserts an iframe for `https://c.example.com/`, which has no origin policy. Then `https://b.example.com/2` tries to `postMessage()` a `SharedArrayBuffer` to `https://c.example.com/`.

What happens?

The answer that the [proposed algorithm](./README.md#specification-plan) gives is that the `postMessage()` fails. Within the browsing context group (i.e. tab #1), we find the origin `https://b.example.com/` used as an agent cluster key, so even though the `https://b.example.com/2` iframe was loaded with an origin policy saying `"isolation": false`, it still gets origin-isolated.

OK, let's go further.

* Now we open up a new tab to `https://example.org/`, tab #2. Because of the server update, the origin policies corresponding to the nested iframes for `https://a.example.com/` and `https://b.example.com/1` have `"isolation": false` set.
* The `https://a.example.com/` iframe tries to `postMessage()` a `SharedArrayBuffer` to the `https://b.example.com/1` iframe.

What happens this time?

This time, we are in a new browsing context group, with a new agent cluster map. So this time, the iframes are not origin-isolated (i.e. everything is using site agent cluster keys), and the sharing succeeds.

This means that, if you want to "un-isolate" your origin, you can to do so via a new, disconnected browsing context group, separate from the isolated one.
