# Two OpenClaw Nodes Working Together

**Overview**

You have two OpenClaw servers (gateways) and want them to cooperate while keeping their gateways separate. Below is a concise guide that covers pairing the nodes, sharing agents/sessions, syncing skills & configuration, and typical use‑cases.

---

## 1️⃣ Pair the Two Servers (Nodes)

1. **Ensure each server’s gateway is running**
   ```bash
   openclaw gateway status
   ```
   If it isn’t, start it:
   ```bash
   openclaw gateway start
   ```

2. **Generate a pairing request on the *primary* server**
   ```bash
   nodes pair --name secondary --url http://<SECONDARY_IP>:<PORT>
   ```
   This prints a short *pair‑code*.

3. **Approve the request on the *secondary* server**
   ```bash
   nodes approve --code <pair‑code>
   ```
   After approval you’ll see both nodes listed with `nodes status`.

4. **Verify the link**
   ```bash
   nodes describe
   ```
   You should see each node’s `id`, `address`, and `status = online`.

> **Tip:** Keep the two gateways on the same network (or expose them via a VPN) so the heartbeat / ping traffic stays low and secure.

---

## 2️⃣ Share Agents / Sessions Across the Pair

### a) Spawn Sub‑Agents on the Remote Node
```bash
sessions_spawn \
  --task "run my‑backup‑script.sh" \
  --label backup‑job \
  --node <secondary‑node‑id>
```
The sub‑agent runs on the secondary server, but its result is posted back to the primary session automatically.

### b) Send Messages Between Servers
```bash
sessions_send \
  --sessionKey <primary‑session‑key> \
  --message "Backup completed on secondary at $(date)"
```
You can also use `message` with `channel` set to the remote node’s ID if you prefer a direct push.

### c) Distributed Cron Jobs
Create a cron job on the *secondary* that fires a `systemEvent` on the primary:

```json
{
  "name": "sync‑weather",
  "schedule": { "kind": "cron", "expr": "0 * * * *" },
  "payload": { "kind": "systemEvent", "text": "weather‑update" },
  "sessionTarget": "main"
}
```
Add it with:
```bash
cron add --job '<JSON‑above>'
```
When the job runs on the secondary, the primary receives a `systemEvent` that you can handle in a normal session (e.g., fetch weather, post to a channel).

---

## 3️⃣ Keep Skills & Configuration in Sync

| Need | How |
|------|-----|
| **Same skill set** | Use the **clawhub** skill to `clawhub sync` the skill directories, or simply copy the `skills/` folder via `rsync` and run `npm install` on the second server. |
| **Shared config (e.g., API keys, defaults)** | Store common values in a shared file (e.g., `shared-config.yaml`) and load it from both `gateway` configs (`gateway config.patch`). |
| **Version control** | Keep the whole workspace (`/home/jarvis/.openclaw/workspace`) in a Git repo and pull/push on both machines. That way any change you commit on one server is instantly available on the other. |

---

## 4️⃣ Typical Use‑Cases

| Scenario | Recommended Pattern |
|----------|--------------------|
| **Load‑balanced task processing** | Spawn agents on whichever node has the lowest `cpu`/`mem` (query via `nodes describe`) and let the result flow back to the originating session. |
| **Fail‑over / redundancy** | Mirror critical cron jobs on both nodes; each job can check a tiny “leader‑election” flag stored in a shared file or Redis to avoid double‑execution. |
| **Cross‑node notifications** | Use `message` with `channel` set to the remote node’s ID, or broadcast a `systemEvent` that both nodes listen for (via a small “event‑router” agent). |

---

## 5️⃣ Quick Checklist to Get You Started

1. ✅ Start gateways on both servers.  
2. ✅ Pair the nodes (`nodes pair` → `nodes approve`).  
3. ✅ Verify with `nodes status`.  
4. ✅ Sync skills/config (clawhub or rsync + Git).  
5. ✅ Test a cross‑node session: `sessions_spawn` a simple `echo hello`.  
6. ✅ Set up any recurring jobs with `cron add` on the appropriate node.  

Once the pair is healthy, you can treat the two machines as a single distributed OpenClaw “cluster” – agents, cron jobs, and messages will flow between them automatically.

---

**Got a specific workflow in mind?** Let me know (e.g., “I want the secondary server to handle all image‑processing jobs”) and I can sketch the exact `sessions_spawn` / `cron` commands you’d need.
