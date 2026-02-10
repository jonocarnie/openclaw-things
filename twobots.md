# Sharing a Single Telegram Bot Between Two OpenClaw Gateways

**Goal**: Run two independent OpenClaw gateways (each on its own server) but have them both respond to the same Telegram bot, so a user can interact with either instance through one chat.

---

## 1️⃣ Create a Telegram Bot (once)

1. Open **BotFather** on Telegram.
2. Send `/newbot` and follow the prompts – give it a name and a username.
3. Copy the **Bot Token** (e.g., `123456:ABC‑DEF1234ghIkl‑zyx57W2v1u123ew11`).

> **Security tip**: Never commit the token to a public repo. Store it in an environment variable or a file with `chmod 600`.

---

## 2️⃣ Configure Both Gateways to Listen to the Same Token

Edit each gateway’s `config.yaml` (usually under `/home/jarvis/.openclaw/workspace/` or `/etc/openclaw/`). Add a distinct Telegram channel for each instance, using the **same token** but a unique `id`/`prefix`.

```yaml
# Server A – config.yaml
channels:
  telegram_a:
    type: telegram
    token: ${TELEGRAM_TOKEN}          # same token on both servers
    polling: true                     # long‑polling (simpler than webhooks)
    chatId: -1001234567890            # optional: restrict to a specific group
    prefix: "/a "                    # messages starting with this go to A only
    id: telegram_a
```
```yaml
# Server B – config.yaml
channels:
  telegram_b:
    type: telegram
    token: ${TELEGRAM_TOKEN}
    polling: true
    chatId: -1001234567890            # can be the same chat or a different one
    prefix: "/b "                    # messages starting with this go to B only
    id: telegram_b
```

### How to apply the token
```bash
# on both servers
export TELEGRAM_TOKEN=123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11
```
Then start (or restart) the gateways:
```bash
openclaw gateway restart   # or start if not running
```
Both gateways will now receive **every** update that the bot gets, but they will only act on messages that begin with their own prefix. This prevents duplicate replies.

---

## 3️⃣ Optional: Prefix‑Free Forwarding (Telegram‑Router Skill)

If you don’t want users to remember `/a` or `/b`, install a tiny router skill that forwards any message **without a prefix** to the other gateway.

### 3.1 Install the skill (once per server)
```bash
clawhub install telegram-router
```
*(If `clawhub` isn’t installed, first run `npm i -g @openclaw/clawhub`.)*

### 3.2 Enable it on each server
```bash
# Server A – forward to B
openclaw skill enable telegram-router \
  --channel telegram_a \
  --target telegram_b

# Server B – forward to A
openclaw skill enable telegram-router \
  --channel telegram_b \
  --target telegram_a
```
The skill works like this:
- It listens on its attached channel (`telegram_a` or `telegram_b`).
- If a message **does not start with the instance’s prefix**, it forwards the raw text to the *other* channel using the OpenClaw `message` tool.
- The opposite direction works the same way.

Now a user can simply type:
```
Hey, what’s the status of server A?
```
The first gateway that receives the update will forward it to the other, run the command, and the reply will appear in the same Telegram chat.

> **Alternative**: Keep the prefixes and skip the router skill if you prefer explicit control (e.g., `/a backup` runs on server A, `/b backup` runs on server B).

---

## 4️⃣ Sending Messages *from* OpenClaw to Telegram

Both gateways can push messages to the same chat using the `message` tool. Example from **Server A**:
```bash
message \
  action=send \
  channel=telegram_a \
  message="✅ Server A backup completed at $(date)"
```
From **Server B** you would use `channel=telegram_b`. Because both channels point at the same bot/token, the user sees a single, unified conversation.

---

## 5️⃣ Things to Watch Out For

| Issue | Why it Happens | Fix |
|-------|----------------|-----|
| **Duplicate replies** | Both gateways react to the same update. | Use a unique `prefix` per instance *or* the `telegram-router` skill to funnel messages to only one side. |
| **Bot token leakage** | Accidentally committing the token. | Store it in an environment variable (`TELEGRAM_TOKEN`) or a file with `chmod 600`. |
| **Polling vs. webhook** | Polling is easy but uses a few extra API calls. | If you have a public HTTPS endpoint, set `webhook: true` and point both gateways to the same URL; a tiny forwarder can then broadcast the payload to both gateways. |
| **Rate‑limit conflicts** | Two gateways may hit Telegram’s per‑bot limits. | Keep traffic low (use prefixes, avoid spamming) or let only one gateway do the heavy‑lifting (the other just forwards). |
| **Different chat groups** | You might want each server to talk to a *different* group. | Set a distinct `chatId` in each channel config; the same bot can be a member of many groups. |

---

## 6️⃣ Quick Checklist to Get You Running

1. **Create the bot** → obtain token.  
2. **Export the token** on both servers (`export TELEGRAM_TOKEN=…`).  
3. **Add a `telegram_*` channel** to each `config.yaml` with a unique `prefix`.  
4. **Restart both gateways** (`openclaw gateway restart`).  
5. **(Optional) Install & enable `telegram-router`** if you want a no‑prefix interface.  
6. **Test**: send a message with `/a ` → only Server A replies; send `/b ` → only Server B replies.  
7. **Add any cron jobs / agents** you need; they can now all talk to the same Telegram chat.

---

### TL;DR
- **One bot token → two `telegram_*` channel definitions** (different `id`/`prefix`).
- **Use prefixes** to keep the two gateways from answering the same message.
- **If you want a single “no‑prefix” interface**, install the `telegram-router` skill to forward messages between the two channels.
- **Both gateways can send messages** to the same chat by using their own channel name in the `message` tool.

That’s all you need to host the files on GitHub – just copy `twonodes.md` and `twobots.md` from the workspace into your repo.
