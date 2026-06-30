# Firestore rules and setup notes — INFOK-ChessControl

## Why this needs explaining

This app uses your own username/PIN login, checked in the browser against the
`users` collection, rather than Firebase Auth. That's the fastest path to a
working app by Thursday, but it means Firestore's built-in security rules
(which key off `request.auth`) can't see who's "logged in" the way they would
with Google sign-in — to Firestore, every request looks anonymous regardless
of which arbiter is using the device.

Practical implication: with this approach, Firestore rules can restrict
*what shape* of write is allowed (e.g. "only touch result/remarks/updatedBy
fields, never touch white/black/board"), but they cannot enforce "arbiter03
may only write to Juniors" — that enforcement lives in the app's UI (it only
shows you your category) rather than the database. For a one-day, trusted,
12-person event this is a reasonable trade. It would not be appropriate for
a public-facing app.

## Rules to paste into Firebase Console → Firestore → Rules

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    match /users/{userId} {
      allow read: if true;
      allow write: if false; // users are seeded once via seed.html, not edited from the app
    }

    match /tournaments/{tCode} {
      allow read: if true;
      allow write: if false;
    }

    match /games/{gameId} {
      allow read: if true;
      allow update: if request.resource.data.diff(resource.data).affectedKeys()
                      .hasOnly(['result', 'remarks', 'updatedBy', 'updatedTime']);
      allow create, delete: if false; // games are seeded once, never created/deleted from the app
    }
  }
}
```

This is the one piece of real protection available without Firebase Auth: even
if someone tampers with the app's JavaScript, Firestore itself will reject any
write that touches `white`, `black`, `board`, `tournament`, or `round` — those
fields are locked after seeding, no matter what the client sends.

## What this does NOT protect against

Anyone with the app URL and a valid username/PIN can write a result to ANY
board, not just their assigned category — the category restriction is a UI
convenience, not a database rule. Given it's 12 known people for one event,
this is an acceptable risk; just don't publish this URL anywhere public, and
treat the PINs as you would a shared door code.

## Deploy checklist

1. Firebase Console → create/open your project → Firestore Database → Create
   database (production mode is fine; the rules above replace the defaults).
2. Paste the rules above, click Publish.
3. Project Settings → General → scroll to "Your apps" → add a Web app (the
   `</>` icon) if you haven't already → copy the firebaseConfig object.
4. Paste that config into BOTH `seed.html` and `index.html` where marked.
5. Open `seed.html` in a browser, click "Seed Firestore", confirm it logs
   "SEED COMPLETE."
6. Open `index.html`, log in as `anitha` / `9001` to confirm the Chief view
   and live overview work.
7. Log in as `arbiter01` / `1001` on a second device/browser to confirm a
   floor arbiter only sees Sub Juniors and the result write lands instantly
   on the Chief's screen.
8. Host both files: easiest is Firebase Hosting (`firebase deploy`) if you
   have the CLI set up already from the Powathikunnel project, or even just
   GitHub Pages — either way, only share the index.html link with arbiters,
   never the seed.html link (no reason for it to exist after step 5).

## Default login credentials (from your existing Users sheet)

| Username | PIN | Role | Access |
|---|---|---|---|
| anitha | 9001 | Chief | ALL |
| jacob | 9002 | Deputy | ALL |
| arbiter01 | 1001 | Floor | SJ |
| arbiter02 | 1002 | Floor | SJ |
| arbiter03 | 1003 | Floor | J |
| arbiter04 | 1004 | Floor | J |
| arbiter05 | 1005 | Floor | J |
| arbiter06 | 1006 | Floor | S |
| arbiter07 | 1007 | Floor | S |
| arbiter08 | 1008 | Floor | SS |
| arbiter09 | 1009 | Floor | SS |
| arbiter10 | 1010 | Floor | ALL (floater) |

These are placeholder PINs generated for you — change them directly in
Firestore Console under the `users` collection any time before Thursday if
you want something less guessable. No code changes needed to change a PIN.

## What still needs the pairing import

This app reads White/Black from the `games` collection, same as the AppSheet
version did. Once Thursday's Swiss-Manager export is ready, the import logic
needs to move from the Google Apps Script (which wrote to a Sheet) to a small
Firestore version — say the word and I'll adapt it now, before you're testing
this under time pressure.
