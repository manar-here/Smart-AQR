# Step 1 — Firebase Foundation

This adds Firebase as a persistent, cloud-reachable data layer on top of the existing local control path. Two things become possible that weren't before:

1. **The ESP32 itself makes outgoing HTTP requests** to fetch pending commands and log data — satisfying the "use HTTP request in ESP32 code to get data from database" course task.
2. Every reading and command is kept as a **permanent research log**, viewable from anywhere (not just your local Wi-Fi), which is the foundation for Step 3's research/session features.

The **existing local control path (dashboard → backend → ESP32 direct HTTP) stays exactly as it was** — it's still what makes button presses feel instant. Firebase runs *alongside* it as an audit trail + remote-access channel, not a replacement.

## 1. Create the Firebase Project (5 minutes, in your browser)

1. Go to [console.firebase.google.com](https://console.firebase.google.com) → **Add project** → name it (e.g. `smart-quadruped-robot`) → you can disable Google Analytics for this project, it's not needed.
2. In the left sidebar: **Build → Realtime Database → Create Database**.
3. Choose a location close to you, then start in **Test mode** for now.
   > ⚠️ Test mode means *anyone with the URL can read/write* — fine for a course project on your own network, but tighten the rules (see bottom of this doc) before showing it publicly or leaving it running long-term.
4. Copy the **Database URL** shown at the top of the Realtime Database page. It looks like:
   `https://smart-quadruped-robot-default-rtdb.<region>.firebasedatabase.app`

## 2. Database Schema

```
/robot
  /pending_command   { command, steps, speed, issuedAt, id }   ← backend writes, ESP32 reads + clears
  /last_executed     { command, executedAt }                   ← ESP32 writes, dashboard can read
  /telemetry         { robotState, distance_cm, servoAngles, updatedAt }  ← ESP32 writes every sync
/readings                                                       ← ESP32 pushes one entry per sync (auto-ID)
  /-Nx7f.../          { distance_cm, timestamp }
  /-Nx7f.../          { distance_cm, timestamp }
```

`/readings` uses Firebase's auto-generated push IDs (`POST` to a Firebase REST path creates a new uniquely-keyed child) — a natural fit for a growing, timestamp-ordered log. Step 3 will nest this under `/sessions/{sessionId}/readings` instead, so a session name can group a batch of readings together.

## 3. Configure Your Credentials

**Backend** — copy `backend/.env.example` to `backend/.env` if you haven't already, and add:
```
FIREBASE_DATABASE_URL=https://YOUR-PROJECT-default-rtdb.YOUR-REGION.firebasedatabase.app
```

**ESP32** — in `esp32/quadruped_robot/config.h`, set:
```cpp
#define FIREBASE_HOST "YOUR-PROJECT-default-rtdb.YOUR-REGION.firebasedatabase.app"
```
(host only, no `https://` prefix — the firmware adds that itself)

## 4. Securing the Database Later

When you're done testing, replace the Test Mode rules (Realtime Database → **Rules** tab) with something like:
```json
{
  "rules": {
    ".read": false,
    ".write": false,
    "robot": { ".read": true, ".write": true },
    "readings": { ".read": true, ".write": true }
  }
}
```
This is still open (no auth) but at least scopes access to only the paths this project actually uses — a reasonable middle ground for a course demo. Full production security would add Firebase Authentication, which is out of scope here but worth mentioning if asked about it in a viva/defense.
