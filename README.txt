SYSTEM-1 PATCH MAP — PWA SETUP
==============================

WHAT'S IN THIS FOLDER
  index.html            the whole app (no build step, no dependencies)
  manifest.webmanifest  makes it installable to your home screen
  sw.js                 service worker — makes it work offline
  icon-192.png / icon-512.png   home screen icons

STEP 1 — HOST IT (one time, ~2 minutes, free)
  Easiest: Netlify Drop
    1. Go to https://app.netlify.com/drop
    2. Drag this whole folder onto the page
    3. It gives you a URL like https://something.netlify.app
  Alternative: GitHub Pages — push these files to a repo,
  enable Pages in repo Settings, use the URL it gives you.

  (Hosting is required — PWAs need HTTPS. Opening index.html
  directly from a file works as a normal page but won't install
  or run offline.)

STEP 2 — INSTALL ON YOUR PHONE
  iPhone:  open the URL in Safari > Share button > Add to Home Screen
  Android: open the URL in Chrome > you'll see an "Install app"
           prompt, or menu (three dots) > Add to Home screen

STEP 3 — USE IT
  After the first load it works fully offline — airplane mode in
  the studio is fine. It reopens exactly where you left off:
  same knob positions, same loaded patch, same genre tab.

YOUR DATA
  Saved patches live on your phone (localStorage), not on a server.
  They survive closing the app and rebooting the phone. They will
  be lost if you clear the browser's site data or delete the app,
  so screenshot anything precious.

BACKUP & SHARING
  EXPORT ALL saves your whole collection as a .json file — keep it
  anywhere (cloud drive, email it to yourself). EXP on a single row
  exports just that patch. IMPORT reads those files back in, on this
  phone or anyone else's copy of the app: exact duplicates are
  skipped, and a name clash comes in as NAME (2). The same files
  work in the Claude version of the app too, and vice versa.
