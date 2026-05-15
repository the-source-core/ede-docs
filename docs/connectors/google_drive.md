# Google Drive Connector — Setup Guide

This guide walks you through creating a Google Cloud project, enabling the Drive API, obtaining OAuth credentials, and configuring the Google Drive connector in EDE.

---

## Prerequisites

- A Google account (personal or Google Workspace)
- Admin access to your EDE instance

---

## Step 1: Create a Google Cloud Project

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Click the project dropdown at the top-left (next to "Google Cloud")
3. Click **New Project**
4. Enter a project name (e.g. `ede-storage`)
5. Select your organization (if applicable) or leave as "No organization"
6. Click **Create**
7. Wait for the project to be created, then select it from the project dropdown

---

## Step 2: Enable the Google Drive API

1. In the left sidebar, go to **APIs & Services → Library**
   - Or visit: `https://console.cloud.google.com/apis/library`
2. Search for **Google Drive API**
3. Click on **Google Drive API** in the results
4. Click **Enable**
5. Wait for the API to be enabled

---

## Step 3: Configure the OAuth Consent Screen

Before creating credentials, you must configure the consent screen that users see during authorization.

1. In the left sidebar, go to **APIs & Services → OAuth consent screen**
   - Or visit: `https://console.cloud.google.com/apis/credentials/consent`
2. Select **User Type**:
   - **Internal** — if you're on Google Workspace and only your org's users will use it
   - **External** — for personal Google accounts or if external users need access
3. Click **Create**
4. Fill in the required fields:
   - **App name**: `EDE Storage` (or your preferred name)
   - **User support email**: your email
   - **Developer contact email**: your email
5. Click **Save and Continue**
6. On the **Scopes** page:
   - Click **Add or Remove Scopes**
   - Search for `Google Drive API` and select the scope: `https://www.googleapis.com/auth/drive`
   - Click **Update**
   - Click **Save and Continue**
7. On the **Test users** page (External only):
   - Click **Add Users**
   - Add the Google account email(s) that will authorize the connector
   - Click **Save and Continue**
8. Review and click **Back to Dashboard**

> **Note:** For External user type, the app starts in "Testing" mode. Only test users you added can authorize. This is fine for EDE connector usage — you don't need to publish the app.

---

## Step 4: Create OAuth Client Credentials

1. In the left sidebar, go to **APIs & Services → Credentials**
   - Or visit: `https://console.cloud.google.com/apis/credentials`
2. Click **+ Create Credentials → OAuth client ID**
3. Select **Application type**: **Web application**
4. Enter a name (e.g. `EDE Connector`)
5. Under **Authorized redirect URIs**, add your EDE callback URL:
   - Development: `http://localhost:8000/api/connectors/oauth/callback/google_drive`
   - Production: `https://your-domain.com/api/connectors/oauth/callback/google_drive`
6. Click **Create**
7. A dialog shows your **Client ID** and **Client Secret** — click **Download JSON**
8. Save the downloaded file (e.g. `credentials.json`)

The downloaded JSON looks like this:

```json
{
    "web": {
        "client_id": "<your-client-id>.apps.googleusercontent.com",
        "project_id": "<your-project-id>",
        "auth_uri": "https://accounts.google.com/o/oauth2/auth",
        "token_uri": "https://oauth2.googleapis.com/token",
        "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
        "client_secret": "<your-client-secret>",
        "redirect_uris": ["http://localhost:8000/api/connectors/oauth/callback/google_drive"]
    }
}
```

> **Important:** This file contains your client credentials but does NOT contain a refresh token. The refresh token is obtained in the next step.

---

## Step 5: Configure the Connector in EDE

1. Log in to your EDE instance
2. Navigate to **Settings → Connectors**
3. Open the **Google Drive** connector (created by default on first install)
4. Click **Import Config** and upload the `credentials.json` file from Step 4
5. Click **Test Connection**

Since the uploaded credentials don't include a refresh token yet, EDE will automatically:

1. Detect the missing `refresh_token`
2. Redirect you to Google's OAuth consent screen
3. You sign in and grant Drive access
4. Google redirects back to EDE with an authorization code
5. EDE exchanges the code for a `refresh_token` and `access_token`
6. The tokens are saved as connector parameters automatically
7. The connection test completes and status changes to **Connected**

Once connected, click **Activate** to enable the connector.

> **Note:** The `redirect_uris` in your Google Cloud OAuth client must include your EDE instance's callback URL (e.g. `http://localhost:8000/api/connectors/oauth/callback/google_drive`). Update this in the Google Cloud Console under **APIs & Services → Credentials → Your OAuth Client → Authorized redirect URIs**.

---

## Verification

After activating the connector:

1. The connector status should show **Connected**
2. Upload a document through EDE's storage module
3. Check your Google Drive — a folder named `ede-storage` (or your configured name) should appear with the uploaded file

---

## Troubleshooting

### "Configuration error: missing required config keys: refresh_token"

The OAuth authorization flow did not complete. Click **Test Connection** again — EDE will redirect you to Google's consent screen to authorize and obtain the refresh token automatically.

### "Connection failed: invalid_grant"

The refresh token has been revoked or expired. This happens when:

- You revoke access at [Google Account Permissions](https://myaccount.google.com/permissions)
- You re-generate the OAuth client secret (invalidates all existing tokens)
- The app is in "Testing" mode and the token expired after 7 days

**Fix:** Run the authorization flow again (Step 5) to get a new refresh token.

### "Connection failed: access_denied"

Your Google account is not listed as a test user. Go to the OAuth consent screen (Step 3) and add your email to the test users list.

### "Error 403: access_not_configured"

The Google Drive API is not enabled for your project. Go back to Step 2.

---

## Security Recommendations

- **Rotate credentials periodically** — re-generate the OAuth client secret and obtain a new refresh token
- **Use a dedicated Google account** — don't use a personal account for production connectors
- **Restrict API scopes** — the connector only needs `https://www.googleapis.com/auth/drive`
- **Monitor API usage** — check the [Google Cloud Console API Dashboard](https://console.cloud.google.com/apis/dashboard) for unexpected activity
- **For production**, consider publishing the OAuth app (removes the 7-day token expiry for external apps in testing mode)
