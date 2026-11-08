---
layout: page
title: Connect to Forge migrations
permalink: /
---

## Your Jira or Confluence app has a LEGACY tag. I'll move it to Forge before it breaks.

Atlassian Connect reaches end of support on 31 January 2027. Connect apps keep running after that date, but Atlassian "will only address critical security patches", and some changes "will cause apps to stop functioning without advance notice" ([Atlassian](https://www.atlassian.com/blog/development/getting-ready-for-connect-end-of-support)). Apps your company built in-house are your responsibility, not Atlassian's.

### What you get

1. **A free audit (async, about 30 minutes of my time).** Send me the list of apps your site shows as LEGACY. I'll tell you which ones:
   - can be deleted
   - can be replaced by a native feature or a Marketplace app
   - need migrating
2. **A fixed-price migration per app:**
   - **Small:** one or two modules, no external services. USD 1,500.
   - **Medium:** up to five modules, or a backend with its own database. USD 3,500.
   - **Large:** quoted after the audit.
3. **What's included:**
   - The Forge app, deployed to your site.
   - Your existing data, carried over to Forge storage.
   - A handover README.
   - 30 days of fixes.

### What I need from you

- The app's source code.
- A test site, or a sandbox on your site.

### Proof

I ported [Figma for Jira](https://github.com/atlassian-labs/figma-for-jira), an open-source Connect app with 48,052 installs, to Forge, including the Figma OAuth flow that Forge's built-in auth can't handle. Read the [case study]({{ "/figma-for-jira-forge-port/" | relative_url }}) or see the [code](https://github.com/tosborne89/figma-forge-port).

### Get in touch

Email me at [osborne.tim89@gmail.com](mailto:osborne.tim89@gmail.com?subject=Forge%20migration), or [message me on Upwork]({{ site.upwork_url }}) if you'd rather Upwork handle the contract and payment.

I don't take projects from banks, payment companies, or identity or KYC providers.
