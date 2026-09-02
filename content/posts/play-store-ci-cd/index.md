---
date: "2026-03-08T13:36:33+07:00"
draft: false
title: "Getting Google Play Developer API Credentials"
summary: "Creating the service account and JSON key in Google Cloud, granting it release access in Play Console, and using it from a GitHub Actions workflow."
categories:
  - Guides
tags:
  - android
  - play-store
  - ci-cd
  - google-cloud
  - github-actions
---

Automating Play Store uploads needs a service account whose credentials live in Google Cloud but whose *permissions* live in Play Console. The two consoles don't link to each other, which is why this takes longer to figure out than it should.

## Before you start

- You need the **Account Owner** role in Play Console to reach the API settings. Admin isn't enough.
- You need access to both the Play Console and the Google Cloud Console.

## 1. Pick the Cloud project

Open the [Google Cloud Console](https://console.cloud.google.com/) and select the project linked to your Play Console account, or create a new one.

## 2. Enable the API

**APIs & Services → Library**, search for **Google Play Android Developer API**, click **Enable**.

Nothing else works until this is on, and the error you get if you skip it doesn't mention it.

## 3. Create the service account

**APIs & Services → Credentials → Create credentials → Service account**.

Give it a name — `app-automation` or similar — then **Create and Continue**, then **Done**. You can skip the optional role-granting step; the permissions that matter are granted in Play Console, not here.

## 4. Generate the JSON key

Click the new service account's email address, go to the **Keys** tab, then **Add Key → Create new key → JSON → Create**.

The file downloads immediately and is the only copy — Google doesn't keep one. It's a full credential: anyone holding it can act as that service account. Never commit it.

## 5. Grant it access in Play Console

This is the step that's easy to miss, because nothing in the Cloud Console suggests it exists.

Open Play Console → **Setup → Users and permissions → Invite new users**. Paste the service account's email address (copy it from the Cloud Console — it looks like `app-automation@project-id.iam.gserviceaccount.com`). Under **Account permissions**, grant what the automation needs — **Release manager** covers uploading and rolling out — then **Invite user**.

Permissions can be scoped to specific apps rather than the whole account, which is worth doing if the service account only ever ships one of them.

## Using it from GitHub Actions

Store the JSON key as a repository secret (**Settings → Secrets and variables → Actions**) called `PLAY_SERVICE_ACCOUNT_JSON` — paste the whole file contents as the value.

Then [`r0adkll/upload-google-play`](https://github.com/r0adkll/upload-google-play) handles the upload:

```yaml
name: Deploy to Play Store

on:
  release:
    types: [published]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17

      - name: Build release bundle
        run: ./gradlew bundleRelease

      - name: Upload to Play Store
        uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.PLAY_SERVICE_ACCOUNT_JSON }}
          packageName: com.example.app
          releaseFiles: app/build/outputs/bundle/release/app-release.aab
          track: internal
          status: completed
```

Start with `track: internal`. It's visible only to your test group, so a misconfigured workflow can't reach real users while you're still getting it working. Promote to `alpha`, `beta` and `production` once the pipeline is proven.

Two things that will fail the first run:

- **The app must already have one manually-uploaded release on that track.** The API can't create a track from nothing.
- **The bundle has to be signed.** Signing config and keystore secrets are a separate problem from API access — the upload step will reject an unsigned artifact.
