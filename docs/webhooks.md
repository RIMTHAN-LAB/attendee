# Webhooks

Webhooks send your server real-time updates when something important happens in Attendee, so that you don't need to poll the API.

Currently, webhooks are only supported for one type of event: when a bot changes its state. This can be used to alert your server when a bot joins a meeting, starts recording or when a recording is available.

## Creating a Webhook

To create a webhook:

1. Click on "Settings → Webhooks" in the sidebar
2. Click "Create Webhook" 
3. Provide an HTTPS URL that will receive webhook events
4. Select the triggers you want to receive notifications for (we currently have one trigger: `bot.state_change`)
5. Click "Create" to save your subscription

## Webhook Payload

When a webhook is delivered, Attendee will send an HTTP POST request to your webhook URL with the following structure:

```
{
  "idempotency_key": < UUID that uniquely identifies this webhook delivery >,
  "bot_id": < Id of the bot associated with the webhook delivery >,
  "bot_metadata": < Any metadata associated with the bot >,
  "trigger": < Trigger for the webhook. Currently, the only trigger is bot.state_change, which is fired whenever the bot changes its state. >,
  "data": < Event-specific data >
}
```

### Payload for `bot.state_change` trigger

For webhooks triggered by `bot.state_change`, the `data` field contains:

```
{
  "new_state": < The current state of the bot >,
  "old_state": < The previous state of the bot >,
  "created_at": < The timestamp when the state change occurred >,
  "event_type": < The type of event that triggered the state change >,
  "event_sub_type": < The sub-type of event that triggered the state change >,
}
```

### Using webhooks to know when the recording is available

The most common use case for webhooks is to be notified when the meeting has ended and the recording is available. You can do this by listening for the `post_processing_completed` event type.

The data field will look like this

```json
{
  "new_state": "ended",
  "old_state": "post_processing",
  "created_at": "2023-07-15T14:30:45.123456Z",
  "event_type": "post_processing_completed",
  "event_sub_type": null,
}
```

## Debugging Webhook Deliveries

Go to the 'Bots' page and navigate to a Bot which was created after you created your webhook. You should see a 'Webhooks' tab on the page. Clicking it will show a list of all the webhook deliveries for that bot, whether they succeeded and the response from your server.

## Verifying Webhooks

To ensure the webhook requests are coming from Attendee, we sign each request with a secret key. You can verify this signature to confirm the authenticity of the request.

The signature is included in the `X-Webhook-Signature` header of each webhook request.

## Webhook Retry Policy

If your endpoint returns a non-2xx status code or fails to respond within 10 seconds, Attendee will retry the webhook delivery up to 3 times with exponential backoff.

## Code examples for processing webhooks

Here are some code examples for processing webhooks in different languages.

### Python

This is a simple flask server that runs on port 5005. It listens for webhook requests and verifies the signature.
```python
import json
import logging
import hmac
import hashlib
import base64

from flask import Flask, request

app = Flask(__name__)
port = 5005

# Add your secret you got from the dashboard here
webhook_secret = "<YOUR_SECRET>"

def sign_payload(payload, secret):
    """
    Sign a webhook payload using HMAC-SHA256. Returns a base64-encoded HMAC-SHA256 signature
    """
    # Convert the payload to a canonical JSON string
    payload_json = json.dumps(payload, sort_keys=True, ensure_ascii=False, separators=(",", ":"))

    # Decode the secret
    secret_decoded = base64.b64decode(secret)

    # Create the signature
    signature = hmac.new(secret_decoded, payload_json.encode("utf-8"), hashlib.sha256).digest()

    # Return base64 encoded signature
    return base64.b64encode(signature).decode("utf-8")

@app.route("/", methods=["POST"])
def webhook():
    # Try to parse as JSON
    payload = json.loads(request.data)
    print("Received payload =", payload)
    signature_from_header = request.headers.get("X-Webhook-Signature")
    signature_from_payload = sign_payload(payload, webhook_secret)
    print("signature_from_header =", signature_from_header)
    print("signature_from_payload =", signature_from_payload)
    if signature_from_header != signature_from_payload:
        return "Invalid signature", 400
    print("Signature is valid")

    # Respond with 200 OK
    return "Webhook received successfully", 200


if __name__ == "__main__":
    print(f"Webhook server running at http://localhost:{port}")
    print("Ready to receive webhook requests")
    log = logging.getLogger("werkzeug")
    log.setLevel(logging.ERROR)  # Only show errors, not request info
    app.run(host="0.0.0.0", port=port, debug=False)
```

### Javascript (Node.js)

This is a simple express server that runs on port 5005. It listens for webhook requests and verifies the signature.

```javascript
// webhook.js
import express from "express";
import crypto from "crypto";

const app  = express();
const port = 5005;

// Put the base‑64 secret from your dashboard here
const WEBHOOK_SECRET = "<YOUR_SECRET>";

/* ---- helpers ----------------------------------------------------------- */

function sortKeys(value) {
  if (Array.isArray(value))           return value.map(sortKeys);
  if (value && typeof value === "object" && !(value instanceof Date)) {
    return Object.keys(value)
      .sort()
      .reduce((acc, k) => ({ ...acc, [k]: sortKeys(value[k]) }), {});
  }
  return value;
}

/** Sign a payload and return a base‑64 HMAC‑SHA256 digest */
function signPayload(payload, secretB64) {
  const canonical = JSON.stringify(sortKeys(payload));
  const secretBuf = Buffer.from(secretB64, "base64");
  return crypto
    .createHmac("sha256", secretBuf)
    .update(canonical, "utf8")
    .digest("base64");
}

/* ---- middleware & route ----------------------------------------------- */

app.use(express.json({ limit: "1mb" })); // parse JSON body

app.post("/", (req, res) => {
  const payload              = req.body;
  const signatureFromHeader  = req.header("X-Webhook-Signature") || "";
  const signatureCalculated  = signPayload(payload, WEBHOOK_SECRET);

  console.log("Received payload =", payload);
  console.log("signature_from_header =", signatureFromHeader);
  console.log("signature_from_payload =", signatureCalculated);

  if (signatureCalculated !== signatureFromHeader) {
    console.log("Signature is invalid")
    return res.status(400).send("Invalid signature");
  }

  console.log("Signature is valid");
  res.send("Webhook received successfully");
});

/* ---- start server ------------------------------------------------------ */

app.listen(port, () => {
  console.log(`Webhook server running at http://localhost:${port}`);
  console.log("Ready to receive webhook requests");
});
```
