# Security Policy

BloodHound Legacy (v4) is an **Electron desktop application** that ingests and
renders data collected from Active Directory / Azure Active Directory
environments. The upstream project is deprecated; this fork exists to keep the
v4 application **safe to run** for people who still depend on it.

This document describes the current security posture, the threat model, and the
hardening roadmap.

## Threat model

The most important thing to understand about this app is that the data it loads
is **untrusted**. BloodHound ingests JSON produced by the SharpHound / AzureHound
collectors (and any JSON a user is handed), then renders attacker-controllable
fields (node names, descriptions, etc.) in the UI. That means:

- A malicious or tampered collection file is an attacker-controlled input.
- The renderer process runs with **Node integration enabled** (see below), so
  any path that turns rendered data into code execution, or that navigates the
  window to remote content, is effectively **remote code execution** on the
  analyst's machine.

The analyst running BloodHound is typically a privileged user (red/blue teamer),
which makes their workstation a high-value target. Treat collection files from
untrusted sources accordingly.

## Current posture

### Renderer privileges (`main.js`)

The renderer currently runs with `nodeIntegration: true` and
`enableRemoteModule: true`. This is **not** Electron's recommended
configuration, but it cannot simply be flipped off: the renderer
(`src/index.js` and several components) imports the Electron `remote` module,
`fs`, and `path` directly. Disabling Node integration without first introducing
a preload script + IPC bridge would break the app.

To mitigate the worst consequence of running with Node integration on, the main
process now **locks down navigation**:

- `will-navigate` is blocked for any non-`file://` URL, so the window cannot be
  driven to a remote origin (which would otherwise mean RCE).
- `new-window` is denied; external `http(s)` links are handed off to the user's
  default browser via `shell.openExternal` instead of opening inside the app.

### Dependencies

Dependency vulnerabilities are tracked with `npm audit` and Dependabot.
Non-breaking fixes are applied via `npm audit fix`. Fixes that require major
version bumps (for example, upgrading Electron or Bootstrap) are evaluated
separately because they can break the build or the UI — see the roadmap below.

You can reproduce the current state with:

```bash
npm audit
```

## Hardening roadmap

These are known, intentionally-deferred improvements, roughly in priority order:

1. **Context isolation.** Introduce a `preload.js` that exposes a minimal,
   explicit API over `contextBridge`, move all `fs` / `path` / `remote` usage in
   the renderer behind IPC, then set `contextIsolation: true`,
   `nodeIntegration: false`, and `sandbox: true`. This is the single biggest
   security win and removes the need for the navigation work-arounds above.
2. **Drop `@electron/remote` / `enableRemoteModule`.** The `remote` module is
   deprecated and a known security foot-gun; replace its uses with IPC.
3. **Upgrade Electron.** The pinned major (Electron 11) is long out of support
   and carries known CVEs. Upgrading is a breaking change and should be done on
   its own branch with a full smoke test.
4. **Add a Content-Security-Policy** to `index.html` to constrain what the
   renderer can load and execute.
5. **Sanitize rendered AD data** consistently to prevent injection from
   attacker-controlled collection files.

## Reporting a vulnerability

If you find a security issue in this fork, please open a GitHub issue (or, for
sensitive reports, contact the repository owner directly) with:

- a description of the issue and its impact,
- steps to reproduce, and
- the affected version / commit.

Because this is a volunteer-maintained fork of a deprecated project, there is no
formal SLA, but reports will be reviewed on a best-effort basis.

## Looking for a maintained tool?

If you do not specifically need BloodHound v4, use the actively maintained
[BloodHound Community Edition](https://github.com/SpecterOps/BloodHound)
instead. It receives regular security updates.
