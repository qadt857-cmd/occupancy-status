# Occupancy Status — GitHub Pages version

This is the same dashboard, rebuilt to run on GitHub Pages with Firebase as
the shared backend (instead of claude.ai). Google Drive auto-sync is not
included — the admin uploads the Excel file manually, same as before that
feature existed.

## Part 1 — Create the Firebase project (free)

1. Go to https://console.firebase.google.com and sign in with your Gmail account.
2. Click **Add project** → name it (e.g. "occupancy-status") → finish the wizard
   (you can turn off Google Analytics, it's not needed).
3. In the left sidebar: **Build → Firestore Database → Create database**.
   - Choose **Start in production mode**.
   - Pick any location close to you.
4. In the left sidebar: **Build → Authentication → Get started**.
   - Under **Sign-in method**, enable **Google**.
   - Under **Settings → Authorized domains**, add the domain your GitHub
     Pages site will use (e.g. `yourusername.github.io`) — you can add this
     after Part 3 once you know the exact address.
5. Click the gear icon (top left) → **Project settings** → scroll to
   **Your apps** → click the **</>** (web) icon → register an app (any
   nickname) → it will show a `firebaseConfig` object. Copy it.

## Part 2 — Set the security rules

Still in Firebase console: **Build → Firestore Database → Rules**, replace
the contents with this, then click **Publish**:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function isAdmin() {
      return request.auth != null && request.auth.token.email == "qadt857@gmail.com";
    }
    function isApprovedViewer() {
      return request.auth != null &&
        request.auth.token.email in
          get(/databases/$(database)/documents/config/access).data.viewers;
    }

    match /occupancy/summary {
      allow read: if isAdmin() || isApprovedViewer();
      allow write: if isAdmin();
    }
    match /occupancy/units {
      allow read, write: if isAdmin();
    }
    match /config/access {
      allow read, write: if isAdmin();
    }
  }
}
```

(Already filled in with your admin email — just copy this block as-is. This
version adds the approved-viewer list: nobody can even read the summary
unless they're the admin, or their signed-in email is on the `viewers` list
in `config/access` — which you manage from the "Manage access" button in
the page itself, no need to touch Firestore's console.)

This is what actually enforces "only I can upload, everyone else can only
view the summary" — the same job the claude.ai `db` rules were doing before.

## Part 3 — Edit index.html

Open `index.html` in Visual Studio / VS Code and edit the block near the
top of the `<script>` section:

```js
var firebaseConfig = {
  apiKey: "PASTE_YOUR_API_KEY",
  authDomain: "PASTE_YOUR_PROJECT.firebaseapp.com",
  ...
};
var ADMIN_EMAIL = "youradminemail@gmail.com";
```

- Paste in the `firebaseConfig` object you copied in Part 1, step 5.
- Set `ADMIN_EMAIL` to the same Gmail address you used in the security rules.

## Part 4 — Push to GitHub

1. Create a new **public** repository on GitHub (e.g. `occupancy-status`).
2. Upload `index.html` to it (via GitHub's web "Add file → Upload files", or
   with git from your PC — either works).
3. In the repo: **Settings → Pages** → under "Build and deployment", set
   **Source** to "Deploy from a branch", branch `main`, folder `/ (root)` →
   **Save**.
4. GitHub will give you a live URL after a minute or two, typically:
   `https://yourusername.github.io/occupancy-status/`

## Part 5 — Finish the Firebase authorized domain

Go back to Firebase console → **Authentication → Settings → Authorized
domains** → add `yourusername.github.io` (the domain from step 4, without
the path). Without this step, Google sign-in will fail on the live site.

## Using it

- Anyone who opens the link is asked to **sign in with Google** first — no
  one sees any data without signing in.
- Only two kinds of signed-in accounts get in:
  1. **You (admin)** — full access: Upload, Export, and the "Manage access" list.
  2. **Approved viewers** — anyone whose email you've added to the approved
     list can view the dashboard, nothing else.
  Everyone else who signs in sees an "Access pending" screen and can't see
  any data — this is enforced by the Firestore rules, not just hidden
  buttons.
- To approve someone: sign in as admin, click **"Manage access"** (top
  right), type their Gmail address, click Add. They can then sign in and
  view immediately (or click Refresh if they were already on the pending
  screen). Remove access the same way, any time.
- Upload the Excel file the same way as before (button or drag-and-drop).
  Everyone approved to view updates live, same as on claude.ai.

## What's different from the claude.ai version

- No Google Drive auto-sync — manual upload only.
- Viewing now requires sign-in plus admin approval — stricter than the
  claude.ai version, by request, since the repo (and therefore the link) is
  public.
- You now own and control the whole stack — Firebase's free tier comfortably
  covers a single small internal dashboard like this.
