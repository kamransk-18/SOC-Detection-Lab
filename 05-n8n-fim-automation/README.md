# 05 — FIM Automation Extension (n8n: VirusTotal Hash Lookup + Telegram Alerting)

Standalone [n8n](https://n8n.io) workflow extending `04-cross-platform`'s FIM and VirusTotal detections: decouples hash-reputation alerting from Wazuh itself. A webhook receives a file hash (handed off from a FIM event), checks it against VirusTotal, and pushes a verdict straight to Telegram — independent of Wazuh's rule engine and dashboard.

## What's being tested

Whether hash-reputation alerting can run as a fan-out layer outside the SIEM — same VirusTotal verdict, delivered wherever you want (Telegram here, Slack/email/ticket later) without touching Wazuh's rule engine per channel added.

## Workflow

`Webhook (POST)` → `Edit Fields` (manual — normalizes payload to a single `hash` field) → `VirusTotal hash checking` (HTTP Request to VT API) → `Hash Found?` (IF node, true/false) → `Hash Found Message` / `Hash Not Found Message` (Telegram `sendMessage`, one per branch).

## Setup

1. **Webhook node** — set to `POST`. Entry point; wire it to receive a hash from wherever your FIM pipeline hands one off (Wazuh active-response script, scheduled export, or manual test `curl`).
2. **Edit Fields node** (Set node, manual mode) — maps incoming payload down to a single `hash` field so downstream nodes don't care what the source payload looked like.
3. **VirusTotal hash checking node** — HTTP Request node calling VT's file-lookup endpoint (`GET https://www.virustotal.com/api/v3/files/{{ $json.hash }}`) with your API key sent as the `x-apikey` header. Store the key as an n8n credential, not hardcoded in the node.
4. **Hash Found? node** — IF node branching on the VT response (e.g., HTTP `200` + `last_analysis_stats.malicious > 0` → true; `404` or a clean report → false).
5. **Telegram nodes** — `sendMessage` on each branch, each using a bot token + chat ID credential, posting the verdict to your alerting channel.

## Trigger it end-to-end

```bash
# Simulate a FIM event handing off a hash to n8n — run from anywhere that can reach the n8n host
curl -X POST https://<n8n-host>/webhook/<webhook-id> \
  -H "Content-Type: application/json" \
  -d '{"hash": "<sha256-of-a-test-file>"}'
```
Use a known-malicious published hash (e.g. the EICAR file's SHA256) to exercise the **Hash Found** branch, and the hash of a benign file you created yourself to exercise **Hash Not Found**.

## Screenshot checklist

| # | Suggested filename | What it captures | Command / action to run first |
|---|---|---|---|
| 1 | `01-n8n-workflow-canvas.png` | Full workflow canvas (Webhook → Edit Fields → VT lookup → branch → Telegram) | Open the workflow in n8n's editor |
| 2 | `02-n8n-webhook-trigger.png` | n8n execution log showing the incoming webhook POST | `curl -X POST https://<n8n-host>/webhook/<id> -d '{"hash":"..."}'` |
| 3 | `03-n8n-vt-response.png` | VirusTotal hash-checking node's output pane showing the raw API response | Click the node after a manual execution |
| 4 | `04-n8n-hash-found-branch.png` | Execution trace highlighting the **true** path | Trigger with a known-malicious hash |
| 5 | `05-telegram-alert-found.png` | The actual Telegram message received ("Hash Found") | (result of step 4, captured on your phone/Telegram client) |
| 6 | `06-n8n-hash-notfound-branch.png` | Execution trace highlighting the **false** path | Trigger with a benign/unknown hash |
| 7 | `07-telegram-alert-notfound.png` | The Telegram message received ("Hash Not Found") | (result of step 6) |

## Analyst notes

- Duplicates part of what Wazuh's native VirusTotal integration (`04-cross-platform`, section C) already does. Value is **decoupling alerting from the SIEM** — same verdict fans out to Telegram, Slack, email, or a ticket without touching Wazuh's rule engine per channel added.
- "Hash Not Found" means VirusTotal has no record of that hash yet — not that the file is safe. Word the Telegram message so on-call responders don't read it as a clean bill of health.
- Add a shared secret / HMAC check on the webhook and rate-limit/queue it — VirusTotal's free tier caps at 4 requests/minute, and an unauthenticated public webhook can be triggered by anyone who finds the URL.
- Next step: extend the false/true branches to also open a ticket (e.g. TheHive), not just message Telegram — turns this into a real triage entry point instead of a notify-only channel.
