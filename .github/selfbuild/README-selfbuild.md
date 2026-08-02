# Self-build Bitwarden (Windows passkey provider) — CI

Auto-tracks upstream, rebuilds the Windows passkey-provider desktop client, signs it with your
FHYQ-Software CA, and publishes an auto-updating MSIX to a GitHub Release.

## One-time setup

1. **Secret** — add repo secret `SIGNING_PFX_BASE64` = contents of
   `…\FHYQ-Software\key\Bitwarden-SelfBuild-CI\ci.pfx.base64.txt`
   (1-year CI cert, subject `CN=Bitwarden-SelfBuild …`, signed by FHYQ-Software CA, passwordless).
   ```
   gh secret set SIGNING_PFX_BASE64 --repo FHYQ-Dong/bitwarden-clients < ci.pfx.base64.txt
   ```
2. **Trust the CA on each machine** (once): import `FHYQ-Dong-CAroot` into LocalMachine\Root and
   `FHYQ-Software-CA` into LocalMachine\CA. (Already done on the dev machine.)
3. **Run once**: Actions → `selfbuild-release` → *Run workflow*. It creates the `selfbuild-latest`
   release with the `.appx` + `Bitwarden.appinstaller`.

## Install so it auto-updates

```powershell
Add-AppxPackage -AppInstallerFile `
  https://github.com/FHYQ-Dong/bitwarden-clients/releases/download/selfbuild-latest/Bitwarden.appinstaller
```
App Installer then re-checks that URL on launch (every ~8h) and updates when the version bumps.
The build enables `WindowsNativeCredentialSync` at build time; you can also enable it on your
self-hosted server and delete that overlay step.

## Lifecycle (this is a temporary bridge)

- **Now** — upstream release tags don't contain PM-29791 (native OS user verification, needed for
  the passkey *assertion*). So we build from `upstream/main` + the PM-29791 branches. That merge can
  conflict as those branches move; on conflict the workflow opens an issue and you resolve/update
  `PM_BRANCHES`.
- **When PM-29791 is in a release tag** — set `BUILD_FROM_TAG: "true"` and empty `PM_BRANCHES` in the
  workflow env. Now it's just "upstream tag + your appx overlay" (robust, no feature-branch merge).
- **When Bitwarden ships it enabled in the official signed appx** — delete this workflow and install
  official Bitwarden. Nothing here is needed anymore.

## What the fork actually changes (the whole overlay)

Only the appx identity/signing so *you* can sign it (applied in-CI, not committed to upstream code):
`appx.identityName`, `appx.publisher`, `appx.publisherDisplayName`, `appx.customManifestPath`
(pulls in the `com:Extension`), `win.publisherName`. Plus the optional build-time feature-flag flip.
