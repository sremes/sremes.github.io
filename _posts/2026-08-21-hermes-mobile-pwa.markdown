---
layout: post
title:  "Forking Hermes Desktop into an Android PWA"
date:   2026-08-21 12:00:00 +0300
categories: pwa android
---

I run a Hermes agent daily. The desktop app that talks to it is
Electron, and I wanted the same interface on my Android phone. Building a
native client was out of scope for me. Instead, we forked the desktop
renderer and shipped it as a browser PWA. This post covers why, how the fork
works, and the bridge layer that made it work.

## Why not use the mobile site or a native app

Hermes Desktop is one renderer with two shells: an Electron shell and a web
renderer. The web renderer already speaks to the Hermes gateway over REST
and a WebSocket. The Electron part exists for local files, native menus, and
auto-update. None of that matters on a phone where the gateway runs the
agent remotely.

A thin wrapper around the existing renderer would give us:

- the full chat UI, session list, model picker, and settings, without
  maintaining a second frontend;
- upstream changes merged in instead of reimplemented;
- install-to-home-screen through a standard PWA manifest.

So we stripped Electron and kept the renderer.

## The fork

The repo is a monorepo. We kept only `apps/desktop` and `apps/shared`, then
removed every file that needed Node or Electron APIs. What remains is a Vite
React app that talks to the gateway only through HTTP.

Upstream moves fast, so raw merges are useless: most of their commits touch
paths we deleted. Our sync process splits their tree down to those two
directories first, then merges that split. Conflicts stay in a small,
known set of files. A dependency-drift check after each merge catches build
graph changes outside our subtree. The full procedure lives in
`UPSTREAM-SYNC.md` in the repo.

## The bridge

`window.hermesDesktop` is the seam between the renderer and the world. On
desktop, Electron fills it. In the fork, `src/bridge/browser-bridge.ts`
fills it with web API implementations backed by gateway REST calls:

| Bridge call | Browser implementation |
|---|---|
| File reads | `GET /api/fs/read-data-url`, `GET /api/fs/read-text` |
| Uploads | `POST /api/files/upload` (any file), `/api/chat/image-upload` (images) |
| Git review | `/api/git/*` endpoints |
| Attachments | upload first, attach the returned host path |

Members with no browser equivalent stay `undefined`. Call sites feature-detect
with `window.hermesDesktop?.member` and fall back. This rule came from a bug:
an early stub returned empty arrays, the feature check passed, and the dead
code path ran silently. Absent means absent; no fake values.

## Making the gateway same-origin

The gateway's session cookie is host-only, and its CORS never allows
credentials. Cross-origin calls from the phone cannot authenticate. The fix
is routing, not code: nginx serves the static build **and** proxies `/api`,
`/auth`, `/login`, and `/fonts` to the gateway under one origin. The cookie
lands on the app's own origin and every request just works.

The WebSocket leg needs care. The `/api` location must set
`proxy_http_version 1.1` plus `Upgrade` and `Connection` headers. Without
them, the connection test passes but the live stream never opens — a failure
mode worth its own debug log.

Auth is basic-auth against the gateway: `POST /auth/password-login` sets the
cookie, the app polls `GET /api/auth/me`, and the WebSocket dials with a
single-use ticket from `POST /api/auth/ws-ticket`.

## Mobile work the desktop never needed

The renderer assumed a mouse and a big window. Changes that mattered:

- Viewport: `100vh` lies on Android. Shell roots use `dvh` units and safe-area
  insets so the keyboard and address bar do not clip the composer.
- Drawers: below a 768px breakpoint, panes collapse into edge drawers. Touch
  needed a real open event and a tap-outside backdrop, since hover strips do
  nothing on touch.
- Keyboards: hotkey badges and shortcut hints are noise on a phone. They hide
  behind a coarse-pointer media query. Physical keyboards still work.
- Files: Android Chrome fires no `change` event from a detached file input.
  Inputs must be attached to the document before `click()`.
- Images: Android cameras default to HEIC, which no Chromium build decodes.
  Unsupported uploads go through a decode ladder ending in a WASM decoder and
  re-encode to JPEG.
- Share target: the manifest registers the installed app as an Android share
  target. A service worker captures the POST body (pages cannot read
  navigation bodies), stashes the payload in the Cache API, and redirects.
  The intake dialog stages a draft into the composer; it never sends by
  itself. You review, pick a model, then press Send.

One process rule proved itself repeatedly: headless tests pass things a phone
fails. Every touch change got verified on the actual device.

## Install story

Service worker rules are boring on purpose: hashed assets cache-first,
navigation network-first with a cached offline fallback, and never intercept
`/api` or auth paths. Installability, share target, and notifications need a
secure context, so plain-HTTP testing cannot exercise them — the
production deploy sits behind HTTPS.

## Result

The phone gets the full desktop chat experience against the same gateway as
the desktop app: sessions, tools, streaming, attachments, git review, and
Android share-sheet integration. One codebase, upstream merges stay cheap,
and there is no separate mobile stack to rot.
