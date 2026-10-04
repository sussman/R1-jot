# Phase 0 runbook: does the Gadgets path reach Jot?

A cheap test that runs before any ESP32 or R1 work. It answers one question:

> When a paired community gadget posts a message to Muse, does **Jot** (with
> Ben's memory and personality) answer, or does a generic Spark model answer?

Time needed: about an hour, most of it waiting on installs and asking the probes.

---

## 1. What this test does and doesn't prove

| If it passes, we know | It does **not** tell us |
|---|---|
| Pairing, the SDK token, and the Noise session work on Ben's account | Whether **voice** turns (`/api/voice/dictation`) get the same full context. That's the ESP32 test, spec section 9 |
| Messages from a gadget reach Ben's personal VM | Whether Meta will keep it this way (spec risk T2) |
| A fresh gadget side chat can use long-term memory, not just the current chat | Anything about R1 hardware |

If it fails, it fails cheaply, and the spec's section 10.4 fallbacks apply.

## 2. Correction to what Dot said in chat: not the Mac Mini

The Linux SDK installer requires **Linux with `apt` and BlueZ**. It exits on
macOS (`install.sh` line 78: `uname -s` must be `Linux`). Pairing also needs
**Bluetooth LE** on the machine itself, because the Muse app pairs over BLE.

Pick one of these hosts:

| Host | Verdict |
|---|---|
| **Raspberry Pi 3B+, 4, 5, or Zero 2 W** with Pi OS Bullseye or later | ✅ The intended target. Easiest by far. |
| Any spare PC or laptop with Debian 11+ or Ubuntu 22.04+ and working BLE | ✅ Works the same way. |
| A Linux VM on the Mac (UTM/Parallels) plus a USB BLE dongle passed through | ⚠️ Possible, but BLE passthrough into a VM is fiddly. Last resort. |
| Docker on the Mac | ❌ No BLE, no systemd. |

The rest of this runbook assumes a Raspberry Pi. Other Debian-family hosts
use the same steps.

## 3. Before you start

- [ ] The Pi is on your **home** Wi-Fi, a trusted network. Community pairing
      can't detect a man-in-the-middle (spec T5).
- [ ] You have an **SDK token** from <https://gadgets.muse.ai/settings/sdk-tokens>
      (Account > SDK tokens). It starts with `mgst_`. Treat it like a password:
      never commit it to this repo.
- [ ] You've read the Gadget SDK Terms: <https://gadgets.muse.ai/sdk-terms>.
- [ ] The Muse iOS app is on your phone.
- [ ] Ground-truth answers for the probes (spec section 9.2) are written down
      **somewhere private**. Don't put them in this repo.

## 4. Steps

### Step 1: Create a low-privilege account for Muse

By default, the Linux SDK lets Muse run **any shell command** (`system.run`)
as the account it's installed for. If that account can use sudo, so can Muse.
For this test, give it an account with no sudo:

```sh
sudo adduser --disabled-password --gecos "" musejot
# Confirm it has no sudo (should print "not allowed" or nothing):
sudo -l -U musejot
```

### Step 2 (optional, recommended): Turn off shell and file-write access

For maximum caution, register **only** `device.health` and `file.read`, so the VM
can't run commands or write files on the Pi at all. This matches spec section 12.1
("least privilege is a design requirement").

```sh
git clone https://github.com/facebookincubator/muse-gadget-sdk.git
cd muse-gadget-sdk/linux
# Edit src/musegadget/executor.py: in COMMAND_SPECS, delete the
# "system.run" and "file.write" entries. Leave the rest untouched.
```

Then install from this local copy in Step 3 (`--from .`). Skip this step if
you're comfortable with the no-sudo account alone.

### Step 3: Install the SDK

Run this on the Pi, from your normal admin account.

**Stock SDK:**

```sh
curl -fsSL https://raw.githubusercontent.com/facebookincubator/muse-gadget-sdk/main/linux/install.sh -o install.sh
less install.sh     # read it first
bash install.sh --run-as musejot --sdk-token mgst_YOUR_TOKEN
```

**Or, if you did Step 2** (run it from inside `muse-gadget-sdk/linux`):

```sh
bash install.sh --from . --run-as musejot --sdk-token mgst_YOUR_TOKEN
```

The installer asks before handing Muse the account, then **opens BLE pairing
for 10 minutes** and prints the device name (`MuseGadgetXXXXXX`).

### Step 4: Pair from the Muse app

1. **Settings > Devices > Developer mode**: turn it on.
2. **Settings > Devices > Add Device** (the **+** in the top right).
3. Pick the `MuseGadgetXXXXXX` that the installer printed.
4. Accept the "community device" warning. It's yours.
5. When asked for Wi-Fi, pick the network shown. No password is needed.

If the 10-minute window closes, run `sudo musegadget pair`.

### Step 5: Confirm it's connected

```sh
musegadget info                       # "paired: yes"
sudo systemctl status musegadget      # active (running)
sudo journalctl -u musegadget -n 50   # look for the VM fetch + link registration, no 401s
```

✅ **Checkpoint A:** pairing works on Ben's account. Write down the date and SDK commit.

### Step 6: Smoke test

```sh
musegadget send-user-msg "Hello from the test Pi. Please reply with one word."
```

Expect `Sent to your Muse.` in the terminal. **The reply does not come back to
the terminal.** `send-user-msg` only delivers. Look for the message and
Jot's reply in the **Muse app's main chat**.

### Step 7: Memory probes in a fresh side chat

This is the real test. Use a **brand-new side chat**, so the answers can't
come from earlier messages in the same conversation. They have to come from
Jot's long-term memory.

```sh
SID=$(python3 -c 'import uuid; print(uuid.uuid4())')
echo "$SID"    # write this down

musegadget send-user-msg --session-id "$SID" "What are my cats' names, and which one sits on shoulders?"
```

Wait for the reply in the app's side chat. Then send each of the rest, **one at a time**:

```sh
musegadget send-user-msg --session-id "$SID" "What is the Bridge Test?"
musegadget send-user-msg --session-id "$SID" "What did we decide about Middle Fork Snoqualmie?"
musegadget send-user-msg --session-id "$SID" "What is my daily calorie target, and what should you never nag me about?"
musegadget send-user-msg --session-id "$SID" "Summarize what you know about the Hermes panel skill."
```

**Style probe** (Dot's addition): memory can come through while personality
doesn't. Send one open-ended prompt where Jot's voice is distinctive, then send
the identical prompt to generic Spark (the general API or a fresh non-Jot chat)
for a side-by-side comparison:

```sh
musegadget send-user-msg --session-id "$SID" "Give me your honest take on how my week is going, in your usual style."
```

> **Unknown to watch for:** the SDK docs say `--session-id` "posts into a side
> chat", but they don't say whether the VM **replies** in side chats the same way
> it does in the main chat. If side-chat messages land with no reply, repeat
> Step 7 without `--session-id` in a quiet moment, and note it in the results.

### Step 8: Continuity test (spec criterion S3)

```sh
musegadget send-user-msg "Note for later: my workshop key is now kept in the blue toolbox."
```

About an hour later, ask from the **phone** (normal chat, not the gadget):
*"Where's my workshop key?"* Then repeat in reverse: tell the phone a new
fact, and ask for it from the gadget's side chat.

## 5. Scoring and what happens next

Score each probe as **correct**, **partial**, or **wrong/generic**, against
the private ground truth.

| Result | Meaning | Next |
|---|---|---|
| **4–5 correct** + style matches Jot | The gadget path reaches Jot with full context | Go to the ESP32 **voice** test (spec Phase 1). That's the remaining unknown |
| **2–3 correct** | Personal VM, but reduced context for gadget turns | Document the gap; check whether a main-chat (no `--session-id`) retry scores higher; Ben decides |
| **0–1 correct**, generic tone | Gadget turns don't reach Jot | Stop. Spec section 10.4 Fallback C. Write the short negative report |

## 6. Results log

Copy this into `docs/results/phase0-YYYY-MM-DD.md` once you've run it. Keep the
actual answers vague, or out of the repo if they're personal.

```markdown
Date:                 
SDK commit:           (git -C muse-gadget-sdk rev-parse --short HEAD)
Host:                 (e.g. Pi 4, Pi OS Bookworm 64-bit)
Commands registered:  (stock / health+read only)
Side-chat id:         

| Probe              | Score (correct/partial/wrong) | Notes |
|--------------------|-------------------------------|-------|
| 1 Cats             |                               |       |
| 2 Bridge Test      |                               |       |
| 3 Middle Fork      |                               |       |
| 4 Calories / nag   |                               |       |
| 5 Hermes skill     |                               |       |
| Style probe        |                               |       |
| Continuity → phone |                               |       |
| Continuity ← phone |                               |       |

Side chats replied?   yes / no
Decision:             go / pivot / stop
```

## 7. Teardown

- **Keep it** if the test passes. This Pi can become the spec's Fallback B
  bridge (R1 → Pi → Jot).
- **Unpair:** Muse app > Settings > Devices > remove the gadget.
- **Uninstall:** `bash install.sh --uninstall --purge`
- **Revoke the token** at gadgets.muse.ai if you're done with it.
- `sudo deluser --remove-home musejot`
