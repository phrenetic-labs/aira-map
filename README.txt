SYSTEM-1 PATCH MAP — PWA SETUP
==============================

WHAT'S IN THIS FOLDER
  index.html            the whole app (no build step, no dependencies)
  manifest.webmanifest  makes it installable to your home screen
  sw.js                 service worker — makes it work offline
  icon-192.png / icon-512.png   home screen icons
  README.md             engineering handoff doc (architecture, design decisions)
  README.txt            this file — end-user setup

STEP 1 — HOST IT
  Recommended: GitHub Pages (see README.md §9, or the walkthrough
  already provided in chat) — push this folder to a repo, enable
  Pages serving from the root of main.
  Alternative: Netlify Drop — drag this folder onto app.netlify.com/drop
  for an instant free URL, no account needed.

STEP 2 — INSTALL ON YOUR PHONE
  iPhone:  open the URL in Safari > Share button > Add to Home Screen
  Android: open the URL in Chrome > "Install app" prompt, or
           menu (three dots) > Add to Home screen

STEP 3 — USE IT
  After the first load it works fully offline. It reopens exactly
  where you left off: same knob positions, same loaded patch, same
  genre tab.

YOUR DATA
  Saved patches live on your phone (localStorage), not on a server.
  EXPORT ALL / EXP on any saved patch gives you a portable .json
  backup — keep one, since clearing browser site data wipes local
  storage.
