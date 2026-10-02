# SMS Backup+ personal setup notes

Written 2026-10-02. Everything below was set up and tested on an Android 16 phone (API 36),
Linux laptop, SMS Backup+ **1.6.0-BETA9 (versionCode 1609)** built from source.

> No passwords or keys are in this file. See **Keep safe** for what to back up separately.

## 1. What this is

- The app is the maintained continuation of jberkel/sms-backup-plus. jberkel's repo is where new code
  and merges land (Mibou's fork was merged into it). Mibou's own repo is stale.
- It backs up SMS, MMS and call logs to Gmail/IMAP. **RCS ("Chat") messages are not backed up.**
  Google Messages keeps them in its own storage, not in Android's SMS database. This is a limit of the app.
- The app is installed from a self-built APK. It is not on Play or F-Droid, so nothing updates it automatically.
- Package ID is `sms.backup.plus`. The pre-2019 package `com.zegoggles.smssync` is the old one and was uninstalled.

## 2. Keep safe (do this before wiping anything)

| Item | Location | Why |
|---|---|---|
| Signing key | `~/.config/sms-backup-plus/release.keystore` | If lost, updates won't install over the existing app. You would have to uninstall and lose app settings. |
| Key passwords | `keystore.properties` in the repo root (gitignored) | Needed to sign. |
| Gmail app password | your password manager | The app uses IMAP with an app password. Never use the main Gmail password. |
| Tasker backup | Tasker, then Data, then Backup | The profile and task below live only inside Tasker. |

## 3. Laptop environment (Linux Mint / Ubuntu 24.04)

1. JDK **17** is required: `sudo apt install openjdk-17-jdk`. JDK 21 fails because `-Werror` turns a Java 8
   source/target deprecation warning into an error.
2. Android SDK, command-line only (no Android Studio):
   - Download `commandlinetools-linux-*_latest.zip` from developer.android.com/studio into
     `~/Android/Sdk/cmdline-tools/latest/`.
   - Run `sdkmanager --licenses`, then install `platform-tools`, `platforms;android-35`, `build-tools;35.0.0`.
   - No NDK is needed.
3. Add to `~/.bashrc`:
   ```bash
   export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
   export ANDROID_HOME=$HOME/Android/Sdk
   export ANDROID_SDK_ROOT=$ANDROID_HOME
   export PATH=$ANDROID_HOME/platform-tools:$ANDROID_HOME/cmdline-tools/latest/bin:$PATH
   ```
4. In the repo, create `local.properties` (gitignored) containing `sdk.dir=/home/<you>/Android/Sdk`.
5. Create a signing key, then `keystore.properties` in the repo root (gitignored):
   ```
   storeFile=/home/<you>/.config/sms-backup-plus/release.keystore
   storePassword=...
   keyAlias=smsbackup
   keyPassword=...
   ```
   Without `keystore.properties` the debug build is unsigned (the build file sets `signingConfig null`).
   Debug and release use the same key when the file exists.
6. Git remotes:
   - `origin` = `https://github.com/jberkel/sms-backup-plus.git` (live repo, `master` tracks it)
   - `mibou` = `https://github.com/Mibou/sms-backup-plus.git` (stale, reference only)
   - `fork` = `git@github.com:<your-github-user>/sms-backup-plus.git` (your fork)
   - Repo-local commit email is the GitHub noreply address, so the personal Gmail address stays out of commits.

## 4. Build and install

```bash
cd ~/Code/sms-backup-plus
git pull                           # only fast-forwards on master
./gradlew assembleRelease          # needs JDK 17
adb install -r app/build/outputs/apk/release/app-release.apk
```

Or use the script **`~/bin/sms-backup-update [adb-serial]`**. It fetches, builds the release, picks the
device (it handles the duplicate wireless entries), installs with `-r`, and launches the app.
It lives outside the repo on purpose. If `~/bin` is lost, the three commands above do the same.

- `adb install -r` keeps app data. It refuses, without wiping anything, if the signing key differs.
- Debug build: `./gradlew assembleDebug`, output `app/build/outputs/apk/debug/app-debug.apk`.
- Unit tests: `./gradlew testDebugUnitTest`.

### Wireless debugging (phone)
1. Phone: Developer options, then Wireless debugging, then **Pair device with pairing code**.
2. Laptop: `adb pair <ip>:<pair-port>` and enter the code. This is done once.
3. Laptop: `adb connect <ip>:<connect-port>`. The connect port is the one on the main Wireless
   debugging screen, **not** the pairing dialog, and it changes when Wireless debugging is toggled.
4. The same phone can appear twice in `adb devices` (IP entry and an mDNS `adb-...` entry).
   Remove the extra with `adb disconnect '<mDNS name>'`, or pass `-s <ip:port>`.

## 5. Fix carried locally: PR #1121

- PR: https://github.com/jberkel/sms-backup-plus/pull/1121, branch `fix/invalid-token-limit` on your fork.
- Problem: on Android 16 the app crashed with `IllegalArgumentException: Invalid token LIMIT` when you chose
  **Skip** in the first-backup dialog. The app put `LIMIT` inside the sort order. The same bug broke any backup
  with a max-items limit (`BUGS.md` known issue 4).
- Fix: plain sort orders plus a client-side `LimitedCursor`.
- **Until it is merged, build from your fix branch, or from a master that contains it.** Building plain
  upstream `master` brings the Skip crash back. `~/bin/sms-backup-update` warns if `LimitedCursor.java` is missing.
- After it is merged: `git checkout master && git pull`, then delete the local branch.

## 6. App settings (SMS Backup+)

### First connection
- Connect with **IMAP and a Gmail app password** (Gmail's OAuth2 access is blocked for this app). IMAP must be
  enabled in Gmail settings. Backups go to the Gmail account you configured.
- **First backup dialog: choose Skip** if the messages are already in Gmail. Skip only marks existing messages
  as backed up. Choosing Backup re-uploads everything (your whole history) and creates duplicates in Gmail.
- If you started the first backup by mistake: stop it, menu (three dots) then **Reset**, tap Backup again, and
  choose **Skip**.

### Backup options
- **SMS:** enabled. **Backup Call log:** enabled (needs the phone permission). It is off by default.
- **3rd party integration:** **ON**. Without it the app ignores Tasker's broadcast.
- **Extra debug information:** off. Turn it on only when diagnosing a problem.
- **Max items per sync:** keep it at unlimited.
- Phone: set the app to **Don't optimize** under battery settings. Aggressive vendor battery managers (e.g. ColorOS) otherwise delays or kills backups.

### Schedules (battery-first, backup-only)
| Setting | Value | Notes |
|---|---|---|
| Auto backup | on | The master switch. |
| Regular schedule | **24 hours** | Longest timer offered (30 min, 1h, 2h, 6h, 24h, Never). Daily safety net. |
| Incoming schedule | **1 hour** (or Never) | Options: 1 min, 3 min, 30 min, 1h, Never. Used for incoming texts and calls. |
| Backup after call | on | Only works with Auto backup on, and uses the Incoming schedule. Not tested on Android 16. |
| Wi-Fi only | optional | Saves mobile data. |

- Earlier I ran with Regular 30 min and Incoming 1 min. That works but costs more battery.
- If Auto backup is off, only Tasker (below) starts backups. Call logs then upload only when a backup runs
  for another reason, for example a text arriving.

## 7. Tasker integration

### Why the old setup broke
The old app listened for the action `com.zegoggles.smssync.BACKUP`. After the package rename it is
**`sms.backup.plus.BACKUP`**. Tasker's built-in **3rd Party Action, then SMS Backup+** entry still sends the old
action, and showed as unavailable (red) because the old package is gone. **Do not use that entry.** Use Send Intent.

### Task: "SMS Backup+ now"
Action: **System, then Send Intent**

| Field | Value |
|---|---|
| Action | `sms.backup.plus.BACKUP` |
| Cat | None |
| Mime Type | empty |
| Data | empty |
| Extra | empty |
| Package | `sms.backup.plus` |
| Class | empty |
| Target | **Broadcast Receiver** |

The **Package** field matters. With it the broadcast is explicit, so Android delivers it to the app's manifest
receiver. Without it the broadcast is implicit and may not arrive.

### Profile: trigger on incoming text
- Event: **Phone, then Received Text** (Type SMS), linked to the task above.
- This is how your profile was described to me. I never saw its exact configuration, so check it against
  your device.
- Tasker must stay alive. Exempt Tasker from battery optimization too.

### Optional profile: calls (not needed if Auto backup and Backup after call are on)
- Event: Phone, then **Phone Idle**, which I believe fires at the end of a call. This is unverified.
- Task: Wait 5 seconds, then the same Send Intent. The wait lets Android write the call to the log first.

### Test from the laptop (same broadcast Tasker sends)
```bash
adb shell am broadcast -a sms.backup.plus.BACKUP -p sms.backup.plus
adb logcat -d | grep SMSBackup
```
Expected log lines:
- `backup requested via broadcast intent` means it was accepted.
- `...but ignored` means 3rd party integration is off in the app.
- `Starting backup (N messages)` followed by `Nothing to do.` means it uploaded N items, then found nothing left.

## 8. How to verify a backup reached Gmail
Open the email in the `SMS` label (calls use the `Call log` label). The headers show:
- `X-smssync-version: 1609` is the app version.
- `X-smssync-backup-time` is the upload time in GMT.
- `X-smssync-id` is the Android SMS database id.
- `X-smssync-date` is the message time in milliseconds.

The first end-to-end test (an OTP text) was uploaded 3 seconds after arrival by the Tasker profile.

## 9. Known issues and troubleshooting

| Symptom | Cause / fix |
|---|---|
| Skip crashes the app | Build is missing the PR #1121 fix. Build from the fix branch. |
| `Nothing to do.` right after a message | Check the message is plain SMS, not RCS (the app can't see RCS). It may also already have been uploaded. |
| Message missing from Gmail | If it's an RCS Chat message, it won't be backed up. Turn off Chat features in Google Messages to keep it as SMS. |
| Duplicates in Gmail | The first backup was run instead of Skip. Delete duplicates in Gmail by hand. |
| Tasker does nothing | Check 3rd party integration is on and the Send Intent has the Package field set. |
| Backups stop on their own | ColorOS battery management. Set the app and Tasker to Don't optimize. |
| `Object already registered` or `Missing event handler` in the log | Harmless, caught and logged by the app. Ignore. |
| `adb devices` shows the phone twice | Same phone (IP and mDNS). `adb disconnect '<mDNS name>'` or use `-s`. |
| Install says signature mismatch | Different key from the installed app. Use the original keystore, or uninstall and lose app data. |
| Gmail login fails | OAuth2 is blocked by Google. Use IMAP with an app password (see `BUGS.md`). |

Other known upstream bugs, in `BUGS.md`: calendar sync flaky on some devices, MMS sender and receiver swapped
on some devices.

## 10. After a factory reset: rebuild checklist
1. Restore the signing key and `keystore.properties`, then redo section 3 on the laptop (or reuse the existing laptop setup).
2. Build and install (section 4).
3. Open the app, connect IMAP with the Gmail app password, and choose **Skip** at the first backup.
4. Enable SMS, Call log, 3rd party integration, and Auto backup with the schedules in section 6, then set Don't optimize.
5. Restore Tasker from its backup, or recreate the task and profile from section 7, and run the task once.
6. Check Gmail for a test message (section 8).
