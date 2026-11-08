---
title: "Porting a 48,000-install Jira app off Atlassian Connect: what moved, what didn't, and the OAuth gap nobody warns you about"
published: false
tags: atlassian, jira, forge, typescript
canonical_url: https://tosborne89.github.io/figma-for-jira-forge-port/
---

Atlassian Connect reaches end of support on 31 January 2027. Connect apps keep running after that date, but Atlassian "will only address critical security patches", and some changes "will cause apps to stop functioning without advance notice" ([Atlassian](https://www.atlassian.com/blog/development/getting-ready-for-connect-end-of-support)).

To see what a real migration involves, I took an open-source Connect app and ported it to Forge. I chose [atlassian-labs/figma-for-jira](https://github.com/atlassian-labs/figma-for-jira) because it is Apache-2.0 licensed and widely used: the "Figma for Jira" Marketplace listing has 48,052 installs and still has no Forge manifest (Marketplace and forge-apps.com APIs, 7 November 2026). It is also a hard case, which makes it a useful one.

The port runs on a live Jira Cloud site. It is a case study, with no connection to Figma or Atlassian. Code: [github.com/tosborne89/figma-forge-port](https://github.com/tosborne89/figma-forge-port).

## What moved cleanly

| Connect | Forge |
|---|---|
| Admin page served from the app's own server in an iframe | `jira:adminPage` in UI Kit, rendered natively by Jira |
| `installed` webhook storing `clientKey` and `sharedSecret` | An install trigger with nothing to store, because Forge owns installation identity |
| `uninstalled` webhook | `preUninstall`, which deletes the app's Figma webhooks and then its data |
| Two Express routes for Figma webhooks, with a passcode check | One web trigger routed by path, with a constant-time passcode check |
| Background sync via `setImmediate` with no retry | A Forge Async Events queue (up to 900 s) that retries on failure |
| PostgreSQL tables for teams and webhooks | Forge key-value storage, with passcodes in secret storage |
| Connect JWT middleware and an installations table | Deleted. Forge authenticates requests itself. |

That last row is the pattern for the whole port: a lot of code simply disappears, because Forge takes over hosting, installation identity and request auth.

## What did not move: the Designs panel

The app's core feature is Jira's native Designs panel, provided through the Connect module `jiraDesignInfoProvider`. Forge has no equivalent module, and its scope list has no design scope ([Forge scopes](https://developer.atlassian.com/cloud/jira/software/scopes-for-oauth-2-3LO-and-forge-apps/)).

Keeping that one module as Connect inside a Forge manifest doesn't escape the deadline either: such an app "is considered a Connect app until all Connect modules are removed" ([developer community](https://community.developer.atlassian.com/t/how-long-forge-connectmodules-will-be-working/98926)).

For a client in this position, I'd recommend moving everything else now and raising the gap with Atlassian. In parallel, prototype a pure-Forge fallback, the route [Figma for Jira Pro](https://marketplace.atlassian.com/apps/1221981) already takes.

## The OAuth gap

Forge has built-in external auth (`withProvider`), and the obvious plan is to delete the app's own Figma OAuth code and use it. That plan fails.

`withProvider` only works in functions a user invokes from the UI. Atlassian staff say providers "only support online asUser which requires user context" ([developer community](https://community.developer.atlassian.com/t/can-forge-webtriggers-access-provider-authentication-like-google/96357)). This app calls Figma in two places where no user is present:

- the queue consumer that reacts to Figma webhooks;
- `preUninstall`, which deletes the app's Figma webhooks.

So the app has to keep its own OAuth code. The port does it like this:

- **Start:** the admin clicks Connect Figma. A resolver signs a short-lived state with HMAC and stores a one-time PKCE S256 verifier. It then returns Figma's authorize URL, which opens in a new tab because Figma blocks its sign-in inside WebViews ([Figma OAuth apps](https://developers.figma.com/docs/rest-api/oauth-apps/)).
- **Callback:** a dedicated web trigger checks the state and consumes the verifier, so a replayed callback fails. It then exchanges the code straight away, since Figma's codes expire after 30 seconds. Tokens go into Forge secret storage under the admin's account ID.
- **Use with no user present:** the consumer looks up who connected the team, loads that account's token, and calls Figma through Forge `fetch`, with `api.figma.com` as the only declared egress host.
- **Refresh:** within an hour of expiry. Figma's refresh response contains no new refresh token, so the old one is kept ([Figma OAuth apps](https://developers.figma.com/docs/rest-api/oauth-apps/)).

This is stricter than the original, which used a signed state only. The port adds PKCE and one-time state.

## A finding that matters for Marketplace apps

Forge web trigger URLs are stable per app, site and environment ([web trigger API](https://developer.atlassian.com/platform/forge/runtime-reference/web-trigger-api/)). On Connect, the app had one base URL, so it could register one OAuth redirect URI with Figma. On Forge, every customer site gets a different callback URL.

For a private app on one site, that doesn't matter. A Marketplace app needs one of these:

- a small fixed relay that forwards the callback to the per-site URL carried in the signed state;
- or one OAuth app per customer.

Plan for this before you start, not after.

## Proof

On a live Jira Cloud site, on 8 November 2026:

- **Sign-in works.** The admin page shows "Connected as" with the Figma account's email, read from Figma's `/v1/me` with the stored token.

  ![The Forge admin page in Jira settings, showing "Connected as" with the Figma account](https://tosborne89.github.io/assets/images/figma-forge-admin-connected.png)
- **Figma's paid-plan rule is handled.** Figma answers webhook creation on free plans with 400 "Access Denied". The port maps that to a clear message, as the original did.
- **Background calls work with no user present.** A queued job, running in the consumer with no user context, read a Figma file with the stored token and logged `figma-sync read file { name: 'Forge port sync test' }`.

  ![forge logs output showing the queue consumer reading the Figma file with no user present](https://tosborne89.github.io/assets/images/figma-forge-sync-log.png)
- **Tests:** 60 unit tests covering state tampering and expiry, one-time verifiers, redirect mismatches, refresh behaviour, and the callback never echoing input. `npm audit` reports 0 vulnerabilities.

## Practical traps

- Adding an egress host forces a major version, a `forge install --upgrade`, and loss of "Runs on Atlassian" eligibility.
- The string form of an egress entry (`'https://api.figma.com'`) now gives a lint deprecation warning. Use `address: api.figma.com`.
- A `jira:adminPage` with `useAsConfig: true` lives at `/jira/settings/apps/configure/{appId}/{envId}`. The URL without `configure` doesn't work.
- Store client secrets with `forge variables set --encrypt`, and type them in your own terminal, never in a shared session.

## If you own a Connect app

If your app still shows LEGACY, I offer a free audit. You get a component map like the one above, with an honest list of what Forge can't do yet. Details are on the [offer page](https://tosborne89.github.io/), email [osborne.tim89@gmail.com](mailto:osborne.tim89@gmail.com?subject=Forge%20migration), or [message me on Upwork](https://www.upwork.com/freelancers/~01c351fe04e5d1ae40).
