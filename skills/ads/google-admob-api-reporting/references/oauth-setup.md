# Authorize the AdMob API

Use an existing authorized AdMob connection if it can access the intended publisher and operations. For a new local connection, have the user complete the desktop OAuth flow in their browser. See [AdMob OAuth setup](https://developers.google.com/admob/api/v1/getting-started) and [OAuth for desktop apps](https://developers.google.com/identity/protocols/oauth2/native-app).

## Configure the Google Cloud project

1. Select a Google Cloud project and enable the **AdMob API**. Configure the consent screen in Google Auth Platform. For an external app in **Testing**, add the Google account that will grant consent as a test user. That account also needs permission to access the target AdMob publisher.
2. Choose the [AdMob scope](https://developers.google.com/admob/api/v1/how-tos/authorizing) based on the requested endpoints.
   - `https://www.googleapis.com/auth/admob.report` covers account lookup and performance reports.
   - `https://www.googleapis.com/auth/admob.readonly` also covers app and ad unit inventory.
3. Create a **Desktop app** OAuth client. Download its JSON to `client.json` in the private directory created below. Run these examples in a local Bash shell with Python 3 and a browser.

```bash
set +x
umask 077
admob_dir="$HOME/.config/google-admob"
mkdir -p "$admob_dir"
chmod 700 "$admob_dir"
```

## Grant access and store the credentials

Run this after downloading `client.json`. The Google OAuth library uses a loopback listener and exchanges the browser authorization code for tokens. Choose the inventory scope in `admob_scope` if the workflow requires inventory.

```bash
chmod 600 "$admob_dir/client.json"
python3 -m venv "$admob_dir/venv"
"$admob_dir/venv/bin/python" -m pip install google-auth-oauthlib
admob_scope='https://www.googleapis.com/auth/admob.report'

"$admob_dir/venv/bin/python" - "$admob_dir" "$admob_scope" <<'PY'
import sys
from pathlib import Path

from google_auth_oauthlib.flow import InstalledAppFlow

directory = Path(sys.argv[1])
output = directory / "authorized-user.json"
if output.exists():
    raise SystemExit("Move the existing authorized-user.json to a private backup before reauthorizing.")

flow = InstalledAppFlow.from_client_secrets_file(
    str(directory / "client.json"), scopes=[sys.argv[2]], autogenerate_code_verifier=True
)
credentials = flow.run_local_server(
    host="127.0.0.1", port=0, authorization_prompt_message="", prompt="consent"
)
if not credentials.refresh_token:
    raise SystemExit("Google did not return a refresh token. Verify the OAuth client and grant.")

with output.open("x", encoding="utf-8") as file:
    file.write(credentials.to_json(strip=["token", "expiry"]))
PY
```

Keep the downloaded client and authorized-user files outside source control. Leave shell tracing and HTTP debug logging off. If refresh fails with `invalid_grant`, check that the OAuth client and grant still match and reauthorize when necessary. Google issues AdMob refresh tokens for external apps in Testing with a seven-day lifetime. Revoke unused grants through [Google Account connections](https://myaccount.google.com/connections). [Refresh token expiration](https://developers.google.com/identity/protocols/oauth2#expiration)

## Get a short-lived access token

Run this before the [REST examples](reporting.md) in the same Bash shell. Command substitution keeps the token out of terminal output. Keep `admob_token` as a shell variable and unset it when finished.

```bash
set +x
admob_dir="$HOME/.config/google-admob"
admob_token=$("$admob_dir/venv/bin/python" - "$admob_dir/authorized-user.json" <<'PY'
import sys

from google.auth.transport.requests import Request
from google.oauth2.credentials import Credentials

credentials = Credentials.from_authorized_user_file(sys.argv[1])
credentials.refresh(Request())
print(credentials.token)
PY
)
```
