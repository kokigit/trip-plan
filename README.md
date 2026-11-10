# Trip Picker

A single-page app for a group to choose a weekend for a trip. It has no backend: the trip state is an encrypted JSON file in this repo, and every vote is a commit.

**Live:** https://kokigit.github.io/trip-plan/

## How it works
- The organiser creates a trip with a name, a date range and a trip length (1–4 days). Every Saturday and Sunday in the range starts out as a candidate date, and the organiser can add or remove days.
- The app generates a random trip id and a 256-bit AES-GCM key. The share link is `…/trip-plan/#t=<id>&k=<key>`. Because the key is in the URL fragment, the browser never sends it to GitHub.
- The trip is stored on the **`data` branch** as `trips/<id>.json` = `{ v, iv, ct }`, which is ciphertext only. Pages serves `main`, so votes never trigger site rebuilds.
- Friends open the link, join with a name and PIN, and tap the days they're free. The app ranks the best consecutive blocks.
- Sync works like this:
  - Each change re-encrypts the trip and commits it through the GitHub Contents API.
  - Concurrent edits are merged: on a sha conflict the app re-reads the latest state and re-applies the change.
  - Open pages poll every 15 seconds.
- The organiser can lock the final date, reopen voting, edit candidate dates, remove people, replace the participant token, or delete the trip.

## Setup
1. **Enable GitHub Pages:** Settings → Pages → Deploy from branch → `main` / root. This is already done for this repo.
2. **Organiser token:** create a [fine-grained token](https://github.com/settings/personal-access-tokens/new) with these settings:
   - Repository access: *Only select repositories* → `trip-plan`
   - Permissions: *Contents → Read and write*
3. **Participant token:** create a second fine-grained token with the same settings and a short expiry, e.g. 30–60 days.
4. Open the site, go to **Create a trip**, and paste the organiser token. It is saved only in your browser. Then fill in the trip details, paste the participant token, and share the link.

The app also works when you fork it. It reads the owner and repo from the `*.github.io` URL, and it creates the `data` branch the first time you save a token.

Local dev: run `python3 -m http.server` and open `http://localhost:8000`. It uses `CONFIG` in `index.html`, which defaults to `kokigit/trip-plan`.

## Security notes
- Anyone with the link can decrypt the trip, **including the participant token**. That token can only write *contents* of this one repo, so keep this repo dedicated to the app and give the token a short expiry. If it leaks, revoke it and use **Replace participant token**. Git history makes any vandalism reversible.
- The PIN protects your votes from casual edits by other people in the group. It is checked in the browser, so it is not a cryptographic guarantee.
- Commit messages contain only the trip id and an action (`vote`, `join`, …). Names, dates and titles are only ever inside the ciphertext.
- If the link is lost, the trip can't be recovered. There is no way to recover a key.
