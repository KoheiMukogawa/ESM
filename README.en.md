# ESM (Entry Sheet Manager)

> 日本語版は [README.md](README.md) を参照してください。

Applying for a job in Japan means writing essays. A company's application — its
*entry sheet* — is a set of open-ended questions, a few hundred to a thousand
characters each, and a student applies to dozens of companies in a single season.
The services that exist to manage this are free because they sell your attention:
banner ads, push notifications, recommended listings. I burned out on them and
built the opposite.

- **Live**: https://koheimukogawa.github.io/ESM/ — shared with 10+ job-hunting
  students; my brother and I use it for our own applications. Not a demo or a mockup.
- **Landing page**: https://koheimukogawa.github.io/ESM/landing.en.html
- **Frontend**: one `index.html`, ~2,500 lines, vanilla JS, no build step, zero dependencies
- **Auth / DB**: Firebase Auth (Google sign-in) + Cloud Firestore, all of a user's data in one document
- **AI**: a Cloud Function verifies the Firebase ID token and relays to the Claude API over SSE

The interface is in Japanese, because its users are.

## Architecture

```
Browser (index.html)
  ├─ Firebase Auth (Google sign-in)
  ├─ Cloud Firestore ── one document: users/{uid}
  │     companies[] (company → question → draft) / materials[] / guidelines / aiUsage
  └─ fetch → Cloud Functions `ai` (gen 2, Node 20, asia-northeast1)
                ├─ verify Firebase ID token
                ├─ UID allowlist (Secret Manager `ALLOWED_UIDS`, fail-closed)
                └─ Claude API (key from Secret Manager) → relayed back as SSE
```

Hosting is GitHub Pages, served from `main`. The free tier requires a public
repository, so making this private would take the site down — which is why the
security model below had to be answered precisely rather than by instinct.

The last five generations of a user's data are also kept in `localStorage`, on a
path independent of the cloud one. Cloud and local storage fail for different
reasons; that independence is the point, and is why this is a backup rather than
a cache.

## Design decisions

Each decision below names the alternative I rejected. The rejected option is
where the reasoning actually lives.

### 1. Why a single HTML file, with no build tooling

`index.html` contains all of the HTML, CSS and JavaScript. I did not reach for
React/Vue on Vite.

The product's premise is that it does not steal your attention, and I wanted the
same property in the codebase. Maintaining a build pipeline — dependency
upgrades, build configuration, a CI job to keep green — is a cost a personal
project at this scale pays forever against value it never collects. Local
development is `python3 -m http.server`. Deployment is a commit.

**Rejected:** a modern frontend stack (bundler plus framework). At ~2,500 lines
the overhead exceeds what the component model returns, and `node_modules` would
be a security surface I then have to watch. I would choose differently at 20,000
lines; I would rather be accurate about the size I am actually at.

### 2. Why all of a user's data lives in one Firestore document

There are exactly two Firestore calls in the entire implementation:
`db.collection('users').doc(uid).set(payload)` and `.get()`.

After sign-in, `loadFromCloud()` finishes in a single `get()`. From then on every
save goes through `scheduleSave()`, which debounces 1.5 seconds before
`saveToCloud()` writes `companies` (the whole company → question → draft tree),
`materials`, `guidelines` and `aiUsage` in one `set()`.

Had this been split into `companies/{id}` or `questions/{id}` subcollections,
changing one field deep in the tree would mean writing to several documents — and
some of those writes can succeed while others fail, leaving the tree half
updated. A single-document `set()` is atomic in Firestore, so that class of
inconsistency **cannot occur by construction**, and I never have to write
reconciliation logic for it. It also minimises the operations themselves: one
read per session, and at most one write per 1.5 seconds.

**Rejected:** splitting into collections and subcollections. At a few dozen
companies and questions per user, the query efficiency that buys does not justify
the complexity.

### 3. Why the browser never calls the Claude API directly

The `ai` function in `functions/index.js` verifies the Firebase ID token with
`admin.auth().verifyIdToken()`, and relays to the Claude API only if the caller
also matches the allowlist. The Claude API key (`sk-ant-…`) exists solely in
Google Secret Manager and is read inside the function as
`process.env.ANTHROPIC_API_KEY`. It is never in the client and never in the
repository.

**Rejected:** calling the Anthropic API from frontend JavaScript. This was not
genuinely a candidate — anything embedded in browser JavaScript can be extracted,
and an extracted key is a stranger spending my credits until I notice. Putting a
Cloud Function in between gives three independent layers — secret isolation,
identity (ID token), authorization (UID allowlist) — without the frontend needing
to know anything.

Because this repository is public, I also avoided hardcoding UIDs that uniquely
identify two real people. They live in Secret Manager as `ALLOWED_UIDS` (a
comma-separated string, injected via the same `secrets: [...]` as the API key). A
UID is not usable without a signed ID token, so it is not secret; it went there to
keep one operational path for values that leave the codebase rather than two.

The check is **fail-closed**: if the allowlist is unset, parses to empty, or fails
to load, every caller is denied — including one holding a valid ID token. An
earlier version skipped the check entirely when the list came back empty. I
changed it deliberately, because "misconfigured, so nobody gets in" is
recoverable and "misconfigured, so anybody gets in" is not.

Note that this allowlist gates **the AI features only**. Access to the app itself
is governed by the Firestore rules below, so it is unrelated to how many people
the app has been shared with.

## About the Firebase Web API key

`index.html` contains a Firebase `apiKey` (`AIzaSy...`) in plain sight. This is
correct, not an oversight. That key identifies the project; it is not a
credential, and Firebase's own documentation states it is safe to expose.

Treating it as a secret would be cargo-cult security: it would buy nothing, and it
would imply the real access control lives in hiding that string. It does not.
Access control is `firestore.rules` (`request.auth.uid == userId`, owner only) and
the ID token verification plus UID allowlist in `functions/index.js`. The value
that genuinely must stay secret is the Claude API key, and that one is in Secret
Manager.

## Firestore security rules

`firestore.rules` is the rule set currently deployed to the production project
(`escounter-d9db7`), retrieved with Firebase's read-only
`firebase_get_security_rules` and synced into this repository. It is what is live,
not what I remember writing — moving it out of console-only management and under
version control is what keeps those two from drifting apart.

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

## What I decided not to build

A product whose promise is "does not steal your focus" is defined mostly by its
refusals. Each of these was designed far enough to be rejected on the merits:

| Not built | Why |
|---|---|
| Push notifications | The exact mechanism I built this to escape |
| Calendar view | A second place for dates to disagree with the first |
| "Generate three full drafts" | Litters the workspace; one suggested angle beats three finished essays |
| Command palette (`Ctrl+K`) | Discovery cost too high for a tool this small |
| Scroll-to-advance between questions | Cheap to build, and nobody would ever find it |
| Duplicate-draft button | Shipped, then removed — it produced more drafts, not better ones |

**Next:** company research grounded in web search. The obvious version — asking
the model what it knows about a firm — is the wrong one: a knowledge cutoff plus a
confident tone is how a student walks into an interview citing something that was
never true. If it ships, every claim carries a source and it is framed as
reference material, never as fact.

## Repository layout

| Path | What it is |
|---|---|
| `index.html` | The entire application |
| `landing.html` / `landing.en.html` | Landing pages (Japanese / English) |
| `functions/` | Cloud Functions source — the `ai` proxy |
| `firestore.rules` | Deployed Firestore security rules |
| `CLAUDE.md` | Project guide: north star, architecture, conventions (Japanese) |
| `PROGRESS.md` | Development log (Japanese) |
| `docs/` | Design specs for individual features (Japanese) |

## Background

The Firestore rules and the Functions secrets were originally managed only
through the Firebase console and were not part of this repository. The north star
("Everything for the job hunt, in one place, without stealing your focus") and the
full design principles are documented in `CLAUDE.md`.

---

© 2026 Kohei Mukogawa · [MIT License](LICENSE)
