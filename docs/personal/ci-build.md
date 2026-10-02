# SMS Backup+ cloud build (GitHub Actions) reference

Written 2026-10-02. Companion to `setup.md` (same folder) (local build, app settings, Tasker).
No passwords or keys are in this file.

## 1. What this gives you
Your fork `https://github.com/<your-github-user>/sms-backup-plus` builds a **signed release APK** on GitHub's servers.
You can start it from a phone, download the APK, and install it over the existing app (same key, so data and settings stay).
A monthly job merges new upstream commits and builds automatically.

## 2. What is in the fork
| Item | Where | Purpose |
|---|---|---|
| Build workflow | `.github/workflows/build-apk.yml` | Builds, signs, verifies and uploads the APK |
| Sync workflow | `.github/workflows/sync-upstream.yml` | Monthly merge of `jberkel/sms-backup-plus` master, then a build |
| Skip-crash fix | `LimitedCursor.java` and related files (PR #1121) | Merged into the fork's `master` so builds include it |
| Secrets | Fork, then Settings, Secrets and variables, Actions | `KEYSTORE_BASE64`, `KEYSTORE_PASSWORD`, `KEY_PASSWORD` (alias is fixed as `smsbackup` in the workflow) |

The fork is public (a fork of a public repo cannot be private). Secrets stay hidden from logs and are not given to pull requests from other forks.

## 3. Build an APK
1. GitHub (web or mobile app), open `<your-github-user>/sms-backup-plus`, then **Actions**, **Build APK**, **Run workflow**, pick branch `master`.
2. Options (both off by default):
   - **Publish release**: also creates a GitHub Release with a link anyone can download.
   - **Run tests**: runs unit tests first (adds a few minutes).
3. Takes about 2 minutes. Open the finished run. The **Summary** shows the signing certificate SHA-256.
   It should be the SHA-256 you noted from your first build (record it privately). A different value means a different key and the phone will refuse to upgrade.
4. Download the artifact `sms-backup-plus-<version>-<code>.apk` from the run (a zip containing the APK; needs GitHub login; kept 90 days).
5. Install: open the APK on the phone and allow "install unknown apps" for the browser or file manager. It installs over the existing app.
6. A push to `master` (including **Sync fork**) also starts a build by itself.

From the laptop:
```bash
gh workflow run build-apk.yml -R <your-github-user>/sms-backup-plus --ref master
gh run watch -R <your-github-user>/sms-backup-plus
gh run download <run-id> -R <your-github-user>/sms-backup-plus -D /tmp/apk
adb -s <ip:port> install -r /tmp/apk/*/*.apk
```

## 4. Staying current with upstream
- **Automatic**: Sync upstream runs at 06:00 UTC on the 1st of every month. It merges `jberkel/sms-backup-plus` master into the fork's `master`, pushes, and starts a build. If nothing is new it does nothing.
- **Manual (web)**: fork page, **Sync fork**, **Update branch**. This starts a build by itself. You can also run Sync upstream from the Actions tab.
- **Manual (laptop)**:
  ```bash
  git fetch origin && git checkout master && git merge origin/master && git push fork master
  ```
- A merge conflict makes the sync run fail, and GitHub emails you. Merge by hand on the laptop and push.
- GitHub pauses scheduled workflows after 60 days with no repo activity. If the monthly sync stops, re-enable it in the Actions tab.
- Once PR #1121 is merged upstream, the fix arrives through the sync and nothing else is needed.

## 5. If the signing key or secrets are lost or must be reset
- The key lives in `~/.config/sms-backup-plus/release.keystore`, with passwords in `keystore.properties` (repo root, gitignored). Keep your offline backups.
- Set the secrets again (this prints no values; run it from the repo root):
  ```bash
  get() { grep -E "^$1=" keystore.properties | head -1 | cut -d= -f2-; }
  base64 -w0 "$(get storeFile)" | gh secret set KEYSTORE_BASE64 -R <your-github-user>/sms-backup-plus
  get storePassword | tr -d '\n' | gh secret set KEYSTORE_PASSWORD -R <your-github-user>/sms-backup-plus
  get keyPassword   | tr -d '\n' | gh secret set KEY_PASSWORD   -R <your-github-user>/sms-backup-plus
  gh secret list -R <your-github-user>/sms-backup-plus
  ```
  Or paste them in the GitHub web UI (for `KEYSTORE_BASE64` use `base64 -w0 release.keystore`).
- If the key is gone for good, the new key cannot upgrade the installed app. You would uninstall (losing app settings) and reconfigure.

## 6. How the build works
- Runner `ubuntu-latest`, Temurin **JDK 17**. JDK 21 fails because `-Werror` turns a Java 8 deprecation warning into an error.
- Guard step: fails if `LimitedCursor.java` is missing (it means the Skip-crash fix is absent from the branch).
- Signing step: decodes `KEYSTORE_BASE64` into the runner's temp directory and writes `keystore.properties`. `app/build.gradle` signs the release when that file has `storePassword`.
- Build: `./gradlew assembleRelease`. Then `apksigner verify --print-certs` and the file is renamed with the version and code.
- Cleanup: the key and `keystore.properties` are deleted at the end, even if the build failed.
- Action versions: checkout v7, setup-java v6, setup-gradle v6, upload-artifact v7. These were chosen to avoid the Node 20 deprecation warnings. Update them if GitHub announces new deprecations.
- Remaining harmless notices: "Gradle 8.7 is out of date" (the project's own version) and the Ubuntu 26 migration of `ubuntu-latest` on 2026-10-19. If a build breaks after that date, pin `runs-on: ubuntu-24.04`.

## 7. Security notes
- Only you can push to the fork, so only you can change what runs with the secrets. Do not add collaborators and keep GitHub 2FA on.
- Do not change a workflow to `pull_request_target`, and do not run builds of untrusted pull requests with the secrets.
- A published Release is public. The APK has no secrets in it, but it is your signed build.
- Artifacts need a GitHub login. They expire after 90 days, so keep your own copy of an APK you want to keep.

## 8. Troubleshooting
| Symptom | Cause / fix |
|---|---|
| Run fails at "Write signing config" with `Secret ... is not set` | A secret is missing. Set it (section 5). |
| Run fails at "Guard - PR" | Branch lacks the Skip-crash fix. Build `master` of the fork, or merge the fix. |
| Phone says "App not installed" or signature conflict | The cert in the run summary differs from your recorded value. Check the secrets match the original keystore. |
| `-Werror` or "warnings found" error | Wrong JDK. The workflow must use Java 17. |
| Dependency download fails (`jitpack.io`) | JitPack outage. Re-run the job later. |
| Monthly sync fails | Merge conflict with upstream. Merge by hand (section 4). |
| Monthly sync did not run | Scheduled workflows were paused after 60 days idle. Re-enable in the Actions tab. |
| Merge succeeded but no build started | Check the Sync upstream run log. The build is started with `gh workflow run build-apk.yml`. Run it manually. |
| Artifact download asks for login | Expected. Sign in to GitHub, or use the Release option. |

## 9. Recreating this from scratch (fork lost or new account)
1. Fork `jberkel/sms-backup-plus` on GitHub.
2. Add the two workflow files (copy from this repo's `.github/workflows/`) and push to the fork's `master`.
3. Make sure the Skip-crash fix is in (PR #1121 or the equivalent `LimitedCursor` change).
4. Set the three secrets (section 5). Use the **same keystore** as before.
5. Run **Build APK** and compare the certificate SHA-256 to the value in section 3.
