# iOS / TestFlight autonomy — what's actually possible

_Last updated: 2026-05-25_

## Why this exists

The first ShareZPresso iOS port hit two hard walls that aren't covered in `ZERO_TOUCH_SETUP.md`. Both come from Apple's tooling, not from Mosh's machine. Both have one-time fixes; once done, every future Claude session on the same Mac is autonomous for iOS work.

## Wall 1 — macOS Keychain trust prompts kill non-interactive git push

**Symptom:** `git push` fails with `fatal: could not read Username for 'https://github.com': Device not configured`, even though `git credential fill` works in an interactive shell.

**Why:** macOS Keychain's `git-credential-osxkeychain` helper triggers an "Allow / Always Allow / Deny" prompt the FIRST TIME each new binary path tries to read an item. Claude Code's Bash tool runs without a TTY, so the prompt can't be confirmed and the helper silently returns no creds.

**There is no software workaround.** Setting `credential.helper`, copying the binary to a non-spaces path, none of it works because the underlying issue is the Keychain trust dialog needing a human.

**Fix (one-time):** Add a GitHub Classic PAT to `secrets.txt`. Then every Claude session uses the PAT directly via URL embed or `Authorization: token <PAT>` header — bypasses Keychain entirely.

```bash
# Generate at: https://github.com/settings/tokens/new
# Scopes: repo, workflow, admin:repo_hook  (delete_repo optional)
GITHUB_PAT_CLASSIC=ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

This was already in `ZERO_TOUCH_SETUP.md` item 4 — the new lesson is that the macOS keychain fallback _doesn't_ work, contrary to what one might assume from `git credential fill` succeeding interactively.

## Wall 2 — Xcode auto-provisioning requires an Apple ID in Xcode → Accounts

**Symptom:** `xcodebuild -allowProvisioningUpdates archive` fails with:
```
error: No Account for Team "JLG9PFLHA2". Add a new account in Accounts
settings or verify that your accounts have valid credentials.
error: No profiles for 'com.example' were found
```
…even when you pass `-authenticationKeyID <ASC_KEY> -authenticationKeyIssuerID <ID> -authenticationKeyPath <p8>`.

**Why:** The `-authenticationKey*` flags only authenticate App Store Connect API calls (used by `-exportArchive --upload` and `xcrun altool`). For PROVISIONING UPDATES — registering App IDs, requesting profiles — Xcode uses a separate Apple ID session that lives in Xcode → Settings → Accounts. The ASC API key, despite being more powerful in concept, doesn't satisfy this code path.

**Fix (one-time, ~30 seconds per machine, re-prompts every ~3 months when 2FA rotates):**

1. Open Xcode
2. Cmd + , (Settings)
3. Accounts tab
4. "+" → Apple ID → enter Apple ID + password + 2FA code

After this, `xcodebuild -allowProvisioningUpdates` works non-interactively — Xcode reuses the cached session to talk to the developer portal, create missing App IDs, request profiles, etc.

## What WAS doable via pure ASC API (and what wasn't)

| Task | ASC REST API | Notes |
|---|---|---|
| List apps + bundles | ✅ `GET /v1/apps`, `GET /v1/bundleIds` | Works, no Apple ID needed |
| Create Bundle ID | ✅ `POST /v1/bundleIds` | Works |
| Create Bundle ID capability (PUSH, APP_GROUPS, etc.) | ⚠️ `POST /v1/bundleIdCapabilities` | Works for most, but APP_GROUPS requires the App Group itself to exist first, and that's only creatable via the developer portal UI or Xcode auto-creation |
| Create iOS Distribution certificate | ✅ `POST /v1/certificates` w/ CSR | Doable: generate CSR with openssl, submit, install returned .cer + private key into keychain with `security import -T /usr/bin/codesign` to avoid the trust prompt |
| Create Provisioning Profile | ✅ `POST /v1/profiles` | Doable once Bundle ID + cert exist |
| Set Coolify env vars | ✅ `POST /api/v1/applications/{uuid}/envs` | See COOLIFY_API_QUIRKS.md |
| Upload IPA to TestFlight | ✅ `xcrun altool --upload-package` w/ ASC API key | Works once you have an IPA |

**The App Group dead end is the wall.** If your app uses App Groups (anything with a Share Extension, Notification Service Extension, Today widget, etc.), there's currently no fully-API path to create the group. Xcode auto-creates it on first build via `-allowProvisioningUpdates` — which needs the Apple ID in Xcode.

## Recommendation for ZERO_TOUCH_SETUP

Add a sixth one-time item to `ZERO_TOUCH_SETUP.md`:

**6. Xcode signed-in Apple ID (30 sec)**

Open Xcode → Cmd+, → Accounts → "+" → enter Apple ID. Required for any iOS project Claude builds; one-time per Mac, re-prompts every ~3 months on 2FA rotation.

## Tested-and-working autonomous iOS pipeline (post-fix)

Assuming items 1-6 of ZERO_TOUCH_SETUP are satisfied, this works end-to-end with zero prompts:

```bash
# 1. Generate Xcode project from project.yml
xcodegen generate

# 2. Archive with auto-signing (Xcode talks to developer portal as needed)
xcodebuild -project App.xcodeproj -scheme App \
  -configuration Release \
  -destination 'generic/platform=iOS' \
  -archivePath build/App.xcarchive \
  -allowProvisioningUpdates \
  -authenticationKeyID "$ASC_KEY" \
  -authenticationKeyIssuerID "$ASC_ISSUER" \
  -authenticationKeyPath ~/.appstoreconnect/private_keys/AuthKey_${ASC_KEY}.p8 \
  DEVELOPMENT_TEAM="$APPLE_TEAM_ID" \
  CODE_SIGN_STYLE=Automatic \
  archive

# 3. Export IPA
cat > build/export.plist <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>method</key><string>app-store-connect</string>
  <key>teamID</key><string>$APPLE_TEAM_ID</string>
  <key>signingStyle</key><string>automatic</string>
  <key>uploadSymbols</key><true/>
</dict></plist>
EOF

xcodebuild -exportArchive \
  -archivePath build/App.xcarchive \
  -exportPath build/export \
  -exportOptionsPlist build/export.plist \
  -allowProvisioningUpdates \
  -authenticationKeyID "$ASC_KEY" \
  -authenticationKeyIssuerID "$ASC_ISSUER" \
  -authenticationKeyPath ~/.appstoreconnect/private_keys/AuthKey_${ASC_KEY}.p8

# 4. Upload to TestFlight
xcrun altool --upload-package build/export/App.ipa \
  --type ios \
  --apiKey "$ASC_KEY" \
  --apiIssuer "$ASC_ISSUER" \
  --bundle-id com.example.app \
  --bundle-version "$BUILD_NUMBER" \
  --bundle-short-version-string "$MARKETING_VERSION"
```

## Why skip fastlane

`fastlane` works fine on a properly set-up Mac but adds a Ruby gem dependency that's fragile on system Ruby 2.6 (default on macOS 24+, EOL'd by Ruby itself). The raw `xcodebuild` + `xcrun altool` flow above does the same thing with zero gem installs. Choose fastlane only if Mosh's machine has a clean Ruby 3.x via rbenv.
