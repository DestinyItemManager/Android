# GitHub Workflows Documentation

This directory contains GitHub Actions workflows for automating SDK updates and Google Play Store publishing.

## Workflows

### 1. Monthly SDK Bump (`sdk-bump.yml`)

**Purpose:** Automatically updates the Android SDK to the latest version on a monthly schedule.

**Schedule:** Runs at 00:00 UTC on the first day of every month

**Trigger:** 
- Scheduled (monthly cron)
- Manual trigger via `workflow_dispatch`

**What it does:**
1. Fetches the latest Android SDK version from Google's repository
2. Compares with the current SDK version in `app/build.gradle`
3. If an update is available:
   - Updates `compileSdkVersion` and `targetSdkVersion`
   - Increments `versionCode` and `versionName`
   - Verifies the build works with the new SDK
   - Creates a pull request with the changes

**No secrets required** - Uses `GITHUB_TOKEN` for PR creation

---

### 2. Publish to Google Play (`publish-play-store.yml`)

**Purpose:** Automatically publishes new app versions to Google Play Store when version numbers change.

**Trigger:**
- Push to `main` or `master` branch with changes to `app/build.gradle`
- Manual trigger via `workflow_dispatch`

**What it does:**
1. Detects if the `versionCode` in `app/build.gradle` has changed
2. If version changed:
   - Builds a signed release AAB (Android App Bundle)
   - Uploads to Google Play Store (production track)
   - Creates a GitHub release with the version tag

**Required Secrets:**

Configure these secrets in your repository settings (Settings → Secrets and variables → Actions):

| Secret Name | Description |
|-------------|-------------|
| `ANDROID_KEYSTORE_BASE64` | Base64-encoded release keystore file. Create with: `base64 -w 0 your-release-key.keystore` |
| `SIGNING_KEY_ALIAS` | Alias of the signing key in the keystore |
| `SIGNING_KEY_PASSWORD` | Password for the signing key |
| `SIGNING_STORE_PASSWORD` | Password for the keystore file |
| `GOOGLE_PLAY_SERVICE_ACCOUNT_JSON` | JSON key for Google Play service account with publishing permissions |

#### Setting up Google Play Service Account

1. Go to [Google Play Console](https://play.google.com/console)
2. Navigate to Setup → API access
3. Create a new service account or use an existing one
4. Grant the service account the following permissions:
   - "Release to production, exclude devices, and use Play App Signing"
   - "Manage store presence"
5. Create and download a JSON key
6. Copy the entire JSON content and add it as the `GOOGLE_PLAY_SERVICE_ACCOUNT_JSON` secret

#### Setting up Android Keystore

If you don't have a release keystore yet:

```bash
# Generate a new keystore
keytool -genkey -v -keystore release.keystore -alias your-key-alias \
  -keyalg RSA -keysize 2048 -validity 10000

# Convert to base64 for GitHub secret
base64 -w 0 release.keystore > keystore.txt
```

Then add the base64 content as the `ANDROID_KEYSTORE_BASE64` secret.

## Workflow Integration

These workflows work together as follows:

1. **Monthly SDK Bump** creates a PR with SDK updates
2. When the PR is merged to `main`, it triggers the **Publish to Google Play** workflow
3. The app is automatically built and published to Google Play Store
4. A GitHub release is created with the version tag

## Manual Triggers

Both workflows can be manually triggered:

1. Go to Actions → Select the workflow
2. Click "Run workflow"
3. Choose the branch and click "Run workflow"

## Testing

To test the SDK bump workflow without waiting for the monthly schedule:
1. Use the "Run workflow" button in the GitHub Actions UI
2. Or modify the cron schedule temporarily for testing

To test the Google Play publish workflow:
1. Make a version number change in `app/build.gradle`
2. Commit and push to `main` (or use manual trigger)
3. Note: This will actually publish to Google Play if secrets are configured

## Troubleshooting

### SDK Bump Issues

- **Gradle build fails**: The workflow includes a build verification step. If it fails, the PR won't be created.
- **No PR created**: Check if the SDK is already at the latest version in the workflow logs.

### Google Play Publish Issues

- **"Keystore not found"**: Verify `ANDROID_KEYSTORE_BASE64` secret is set correctly
- **"Unauthorized"**: Check that the service account has the correct permissions in Google Play Console
- **"Version code already exists"**: Ensure `versionCode` in `app/build.gradle` is higher than what's currently in Google Play

## Maintenance

- Review and merge SDK bump PRs promptly to keep the app up to date
- Monitor Google Play publish workflow for any failures
- Update secrets if keystore or service account credentials change
- Review and update the workflows as Android/Gradle tooling evolves
