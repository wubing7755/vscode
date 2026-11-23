# Hestia fork patches

Hestia is a rebranded Code OSS distribution built from `microsoft/vscode` tags.
This file records every intentional divergence from upstream so that syncing a new
upstream tag is a mechanical job instead of an archaeology exercise.

Rules for keeping this file useful:

- One entry per logical change, with the files it touches.
- Update it in the same commit that changes behaviour.
- When a change is retired, delete the entry instead of leaving it stale.

## Branch model

| Branch | Purpose |
| --- | --- |
| `main` | Read-only mirror of `microsoft/vscode` `main`. Never commit here. |
| `hestia` | Product line. All Hestia changes land here. |
| `sync/<tag>` | Temporary branch that merges a new upstream tag into `hestia`. |

`git rerere` is enabled; keep it that way.

## Patches

### 1. Product identity (`product.json`)

- Renamed the product to `Hestia` (`nameShort`/`nameLong`), `applicationName` `hestia`,
  data folders `.hestia` / `.hestia-shared`, `win32DirName`/`win32NameVersion` `Hestia`,
  `win32RegValueName` `Hestia`, `win32AppUserModelId` `HestiaLab.Hestia`,
  `win32ShellNameShort` `H&estia`, tunnel/server application names, `urlProtocol` `hestia`.
- Assigned Hestia-specific Inno Setup app IDs (`win32x64AppId`, `win32x64UserAppId`) so
  Hestia installs side by side with VS Code and Code OSS.
- Pointed `licenseUrl`/`serverLicenseUrl`/`reportIssueUrl` at the fork.
- Removed `defaultChatAgent`, `trustedExtensionAuthAccess`,
  `builtInExtensionsEnabledWithAutoUpdates`, `voiceWsUrl` and
  `webviewContentExternalBaseUrlTemplate`, which all referenced Microsoft services,
  the GitHub Copilot extension, or Microsoft's CDN.

Why: a rebranded product must not inherit another vendor's identity, telemetry
endpoints or CDN. `webviewContentExternalBaseUrlTemplate` is intentionally absent so
the workbench falls back to its local `vscode-webview://` endpoint instead of
`*.vscode-cdn.net`.

### 2. Windows branding assets (`resources/win32/**`)

- `code.ico`, `default.ico`, `sessions.ico`, `code_150x150.png`, `code_70x70.png`.
- Inno Setup wizard bitmaps (`inno-big-*.bmp`, `inno-small-*.bmp`).
- `VisualElementsManifest.xml`: `ShortDisplayName` → `Hestia`.
- `appx/AppxManifest.xml`: `PublisherDisplayName` → `HestiaLab`.

Keeping the upstream file names means the build scripts and `code.iss` need no changes
and future upstream asset additions keep working.

### 3. Installer text and vendor metadata

- `build/win32/code.iss`: `AppPublisher` → `HestiaLab`, publisher/support/update URLs
  point at the fork, and the "use the System Installer" prompt no longer links to
  `code.visualstudio.com`.
- `build/gulpfile.vscode.ts`, `build/gulpfile.reh.ts`: `CompanyName` → `HestiaLab`,
  `LegalCopyright` → `Copyright (C) 2026 HestiaLab` for the Windows binary metadata of
  native dependencies.

### 4. Copilot stays out of the product build

- `build/gulpfile.vscode.ts`: `prepareCopilotRipgrepShimTask` and
  `compileCopilotExtensionBuildTask` run only when `product.defaultChatAgent` is set.

Why: Hestia ships without a default chat agent, so the Copilot VSIX and its native
runtime shim have no consumer. Building them made packaging depend on downloading
`@github/copilot-*` tarballs, and shipping them would redistribute a proprietary
extension Hestia does not use.

### 5. Workbench survives a missing default chat agent

Everything below assumes `product.defaultChatAgent` exists; Hestia removed it, so each
site now degrades instead of throwing.

- `src/vs/workbench/contrib/welcomeOnboarding/browser/onboardingVariationA.ts`:
  dropped the module-level `assertDefined`, which turned a product configuration
  difference into an uncaught exception while loading the workbench bundle, and
  `show()` now returns early when no agent is configured so the internal command
  cannot render a half-initialised wizard.
- `src/vs/workbench/contrib/welcomeOnboarding/test/browser/onboardingVariationA.test.ts`:
  suite skipped (see "Consequence for upstream tests" below).
- `src/vs/workbench/contrib/welcomeGettingStarted/browser/startupPage.ts`:
  `tryShowOnboarding` returns early when no default chat agent is configured, so the
  AI-centric onboarding wizard is not offered.
- `src/vs/workbench/services/accounts/browser/defaultAccount.ts`:
  `toDefaultAccountConfig` accepts `undefined` and returns a dormant configuration
  instead of dereferencing it during service construction.

Follow-up: the same pattern may exist in other Copilot-adjacent surfaces. Any new
`Not Found`/`Cannot read properties of undefined` error mentioning `defaultChatAgent`
belongs in this section.

Consequence for upstream tests: `src/vs/workbench/contrib/welcomeOnboarding/test/browser/onboardingVariationA.test.ts`
exercises the AI sign-in wizard, which cannot render without a default chat agent, so the
suite is skipped on this fork. Re-enable it if Hestia ever configures its own agent.

## Known gaps

- The CLI `copilot-shim` binary is still copied into `bin/`. It is inert without a chat
  agent but should be dropped from the packaging task.
- Dictionaries/extension suggestions still use upstream URL defaults; revisit before a
  public release.
- The `Code OSS`-style welcome page strings that read product names come from
  `product.json`, so they update automatically; hard-coded "Code OSS" strings elsewhere
  are upstream bugs, not Hestia patches.
