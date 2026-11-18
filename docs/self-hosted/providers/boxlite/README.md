# BoxLite

Run the Gemini API self-hosted environment runtime inside a
[BoxLite](https://boxlite.ai/) sandbox using the `boxlite` CLI and the hosted
BoxLite REST API.

## Prerequisites

On your local machine, install the
[`gemini-api` CLI](https://github.com/google-gemini/gemini-api-cli), the
[`boxlite` CLI](https://github.com/boxlite-ai/boxlite/blob/main/docs/reference/cli/README.md#installation--verification),
`curl`, and `jq`. The commands below use Bash.

Log in to your BoxLite account in the browser and select the cloud context.
Clear any previous environment overrides so later commands use this login:

```bash
unset BOXLITE_API_KEY BOXLITE_REST_URL BOXLITE_REST_PATH_PREFIX \
  BOXLITE_CONTEXT BOXLITE_PROFILE
boxlite --context cloud auth login --url https://api.boxlite.ai --method browser
boxlite context use cloud
```

On SSH or a headless machine, replace `--method browser` with `--method device`
and open the displayed verification URL in a browser on another device.
Login saves the credentials, server URL, and account routing prefix; a BoxLite
API key and manual routing-prefix configuration are not required.

Obtain a [Gemini API key](https://ai.google.dev/gemini-api/docs/api-key) and
create a `self_hosted` environment. Save its ID and scoped key; the API returns
`environment_key` only once, at creation time:

```bash
export GEMINI_API_KEY="your-gemini-api-key"

ENVIRONMENT=$(curl -fsSL -X POST \
  "https://generativelanguage.googleapis.com/v1beta/environments" \
  -H "x-goog-api-key: $GEMINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"mode":"self_hosted"}')
export GEMINI_ENVIRONMENT_ID GEMINI_ENVIRONMENT_KEY
GEMINI_ENVIRONMENT_ID=$(printf '%s' "$ENVIRONMENT" | jq -er '.id')
GEMINI_ENVIRONMENT_KEY=$(printf '%s' "$ENVIRONMENT" | jq -er '.environment_key')
unset ENVIRONMENT
```

## Quick Start

```bash
# 1. Create a detached sandbox with a persistent main process.
SANDBOX_ID=$(boxlite run -d --cpus 1 --memory 512 \
  --entrypoint /bin/sleep python:3.12-slim-bookworm infinity)

# 2. Install gemini-api and prepare a workspace owned by UID 1000.
boxlite exec "$SANDBOX_ID" -- bash -ceu '
  apt-get update
  apt-get install -y --no-install-recommends curl ca-certificates
  curl -fsSL \
    https://raw.githubusercontent.com/google-gemini/gemini-api-cli/main/scripts/install.sh \
    -o /tmp/install-gemini-api.sh
  GEMINI_VERSION=v0.4.6 GEMINI_INSTALL_DIR=/usr/local/bin \
    bash /tmp/install-gemini-api.sh
  rm /tmp/install-gemini-api.sh
  mkdir -p /workspace
  chown 1000:1000 /workspace
'

# 3. Start the worker as a non-root user, passing only the scoped key.
boxlite exec -d -u 1000:1000 -w /workspace \
  -e HOME=/workspace -e GEMINI_ENVIRONMENT_KEY \
  "$SANDBOX_ID" -- /usr/local/bin/gemini-api \
  --no-keyring --no-interactive environments connect \
  --root-dir /workspace --workdir /workspace
```

The worker makes outbound HTTPS connections to Google; no inbound port is
needed. Keep `GEMINI_API_KEY` and your BoxLite login credentials on your local
machine.
Structured file tools are confined to `/workspace`; Bash commands execute
inside the BoxLite sandbox.

## Run an Interaction

From the same local shell:

```bash
INTERACTION_ID=$(gemini-api agent run \
  'Create hello.txt containing "Hello from BoxLite!" followed by a newline, read it back, and run uname -a in bash.' \
  --agent antigravity-preview-09-2026 \
  --body "{\"environment\":\"$GEMINI_ENVIRONMENT_ID\"}" \
  --stream=false --jq '.id')

gemini-api agent status "$INTERACTION_ID" --stream=false --jq '.status'
boxlite exec "$SANDBOX_ID" -- cat /workspace/hello.txt
```

Expect interaction status `completed` and file contents `Hello from BoxLite!`.

## Cleanup

Remove the sandbox to stop the worker, then delete the Google resources:

```bash
boxlite rm -f "$SANDBOX_ID"
gemini-api agent delete-interaction "$INTERACTION_ID"
gemini-api environments delete "$GEMINI_ENVIRONMENT_ID"
unset GEMINI_ENVIRONMENT_KEY
```

If setup fails before an interaction is created, remove any sandbox and
environment already created, skipping deletion of the interaction.
