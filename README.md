# wiki-security-passportjs-gated

**Upstream `wiki-security-passportjs` 0.14.0 with its login-to-view block swapped
for [wiki-view-gate](https://github.com/Hitchhikers-Guide-to-the-Galaxy/wiki-view-gate),
plus the hitchhiker client bootstrap.** OAuth login is upstream's, untouched.

Phase 5 of the [Gatekeeper Plan](http://plan.localhost/view/gatekeeper-plan):
the module the Hitchhiker public farms run, superseding
`wiki-security-hitchhiker-passport` (which kept upstream's server byte for byte
and so carried the gate fixes as hand patches on the Pi5 in
`~/wiki-patches/apply.py`). Those patches are now the module's own behaviour,
shared with `wiki-security-hitchhiker` (the friends-based private-farm module)
because both mount the same gate.

## What the gate swap changes for a restricted site

A `wikiDomains` entry with `"restricted": "true"` gets, instead of upstream's
fabricated "Login Required" paragraph on `.json`/`.html` requests only:

- a **splash page** for logged-out visitors on `/`, `/view/*` and `*.html` —
  the site's `login-required` wiki page rendered in the packaged shell (or
  `assets/splash/index.html` if the site authors one), with a `require_terms`
  page shown inline above the login;
- **blessed public pages** (`public_pages`): their json, `/view` lineups,
  asset folders and a filtered `/system/sitemap.json`, so a logged-out visitor
  can browse the pitch, the licence and the instructions in the real client;
- **everything else walled**, not just page json: `/assets/*`, the assets
  plugin's listings, the farm MCP door, the xml sitemap and the site index;
  matched on a canonical path so `/Secret.JSON/?x=1` is as closed as
  `/secret.json`;
- `"restricted": "terms"` — the terms-gated federation read mode, when
  `wiki-plugin-terms` is installed beside it.

**Who may view** is still upstream's rule, word for word: the site owner, or a
login that clears `allowed_ids` (`["*"]` = any login) / `allowed_domains`.
Those keys, `details`, `splash_page`, `public_pages` and `require_terms` are
what the [Gatekeeper](https://github.com/Hitchhikers-Guide-to-the-Galaxy/wiki-plugin-gatekeeper)
item edits in the farm config.

When `restricted` is unset the gate is a pass-through: an open site behaves
exactly as on upstream.

Two small guards ride along in `defineRoutes`, for every site: the assets
plugin's `/plugin/assets/list` error no longer names the server's disk path,
and `/plugin/assets/index` on a site with no assets folder answers `[]`
instead of a stack trace.

## The rules this fork keeps

1. **One server change, marked.** `server/social.coffee` is upstream's except
   the block headed `THE GATE SWAP` (and the two guards above it). Diff
   against upstream 0.14.0 to see exactly that.
2. **Append, never edit, on the client.** Everything above the marker comment
   in `client/security.js` is upstream, byte for byte. Below it is the
   bootstrap that loads `/plugins/hitchhiker/wiki-ux.js`, the chooser and the
   fish picker on every page. Only list a plugin that is installed: a missing
   one 404s on every page of every site.
3. **Track the upstream version.** `package.json` → `hitchhiker.upstreamVersion`.

## Rolling it out on a farm

The security module is chosen per farm by `security_type`, but a
`wikiDomains` entry may carry its own `security_type`, because the farm merges
that entry over the farm settings before it starts the site's server. So one
site can run this module while the rest still run the old one:

    "low-stakes.example": { "restricted": "true", "allowed_ids": ["*"],
                            "security_type": "passportjs-gated" }

Watch it, then set the farm-level `"security_type": "passportjs-gated"` and
drop the per-site key. Install into the wiki's package directory, derived not
typed:

    W=$(npm root -g)/wiki
    npm install --prefix "$W" wiki-security-passportjs-gated

then restart the farm. Login is unchanged; Keycloak redirect URIs are unaffected.

## To re-fork on a new upstream release

    UP=$(npm root -g)/wiki/node_modules/wiki-security-passportjs   # or the new tarball
    rsync -a --exclude node_modules --exclude client/security.js --exclude server/social.coffee "$UP/" .
    # client/security.js: copy upstream's over ours, re-append the marker block
    # server/social.coffee: copy upstream's over ours, re-apply THE GATE SWAP
    #   (the requires at the top, the two guards, allowedToView + app.use viewGate
    #    in place of the `if argv.restricted?` block)
    # bump to <upstream>-gated.N and set hitchhiker.upstreamVersion

Forked 2026-10-22 from wiki-security-hitchhiker-passport 0.14.0-hitchhiker.3
(itself upstream 0.14.0 + the bootstrap, forked 2026-08-18).
