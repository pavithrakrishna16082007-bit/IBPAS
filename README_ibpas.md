# IBPAS VISION X + IBPAS Platform — setup, protocol, safety, tests

Research prototype. Not clinically validated, not a medical device. Test with the hand **unloaded and clear of people**.

## 1. What was analysed (and what was not)

`true_hand.py` and the Arduino sketch were **not available**. The protocol below comes from the Python
controller code that was pasted earlier in the conversation (V3.5 and its predecessors). Anything the firmware
does that is not visible in that Python code is **unknown** and is not used.

| Direction | Message | Seen in code | Meaning (as used by the Python code) |
|---|---|---|---|
| PC → Arduino | `HELLO` | yes | handshake after the port opens |
| PC → Arduino | `ANGLES,t,i,m,r,p` | yes | five integers 0–180, one per finger |
| Arduino → PC | `READY` | yes | firmware ready |
| Arduino → PC | `FSR:<float>` | yes | force-sensor value (**units unverified**) |
| Arduino → PC | `SERVO,<5 values>` | yes | servo output/feedback (PWM counts; PCA9685 is 12-bit, 0–4095) |
| Arduino → PC | `IMU,ms,ax,ay,az,gx,gy,gz,pitch,roll` | yes | MPU6050 data (**units unverified**) |
| Arduino → PC | `IMU,READ_ERROR` / `IMU,ERROR` | yes | IMU read failure |
| Arduino → PC | `MODE,<text>` | yes (as received text) | **only seen as telemetry**; `MODE,CAM` / `MODE,IMU` as *commands* are **not verified** and stay blocked |

Not verified: command acknowledgements, firmware-side limits/timeouts, servo direction, real FSR/IMU units.

## 2. Files

```
ibpas_vision_x.py     desktop dashboard v5.1: camera + MediaPipe, 3D CAD-style twin (Denavit-Hartenberg kinematics,
                      optional STL import), serial link, TX gate, plots, CSV export, --selftest (14 tests)
ibpas_platform.py     serial/data gateway + SQLite + mobile web + SMS + EMG adapter + backup/rollback, --selftest (15 tests)
ibpas_config.example.json   configuration template (no secrets)
env.example.txt             environment-variable template (passwords, SMS credentials)
README_ibpas.md             this file
```
Both `.py` files must be in the same folder.

```
Arduino --USB--> SerialLink (ONE owner) --> DeviceState --> TelemetryStore --+--> desktop UI (OpenCV)
                       ^                                                      +--> DataService --> SQLite (async queue)
                       |                                                      +--> FaultMonitor --> AlertBus --> SMS
                CommandBroker  <-- allowlist, safety guard, mode arbitration, ramp, watchdog
                   ^         ^
        vision (TX gate)   mobile web API (login, CSRF, TTL, roles)
```

## 3. Windows setup (PowerShell, Python 3.9–3.12)

```
cd C:\path\to\folder
py -3.10 -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install numpy==1.26.4 opencv-python==4.9.0.80 mediapipe==0.10.14 pyserial==3.5 cryptography
python ibpas_vision_x.py --selftest
python ibpas_platform.py --selftest
```
`cryptography` is only needed for `--make-cert`. If your current environment already runs the old controller, you may not need to install anything.

**Back up first, then run**
```
python ibpas_platform.py --backup true_hand.py
$env:IBPAS_WEB_PASSWORD = "choose-a-long-passphrase"
python ibpas_platform.py --port COM5          (desktop window + web on http://127.0.0.1:8443/)
python ibpas_platform.py --headless --no-camera --port COM5
```
If `IBPAS_WEB_PASSWORD` is not set, a random one-time password is printed in the console.

**Phone on the same Wi-Fi (HTTPS required for non-local access)**
```
python ibpas_platform.py --make-cert
python ibpas_platform.py --lan --tls-cert ibpas_cert.pem --tls-key ibpas_key.pem --port COM5
```
Open the printed `https://<pc-ip>:8443/` on the phone and accept the one-time self-signed-certificate warning.
Allow Python through Windows Firewall for **Private** networks only. **Do not port-forward this to the internet.**
For access away from home, use a private tunnel (for example Tailscale) rather than opening a public port.

**Remote control is OFF by default.** To allow it: set `"remote_control_enabled": true` in the config (or `--enable-remote-control`),
then at the PC press **M** to switch the control mode VISION → MOBILE. The phone can always request HOLD; it can never switch to MOBILE.

**SMS:** copy the variables from `env.example.txt` into PowerShell, set `"sms_enabled": true` in the config. Without credentials it runs
as a dry run and logs `NOT_SENT_DRY_RUN`. Twilio is implemented against its documented REST API; for other gateways
(Fast2SMS, MSG91, …) use `IBPAS_SMS_PROVIDER=generic` and the URL/body templates — check the provider's current API and any
local sender-registration rules. Neither provider path has been exercised against a live provider; they were tested against local mock servers.

## 4. Safety design and limits

* TX to the Arduino is **disabled by default** and needs two confirmations; it suspends the instant tracking is lost.
* One writer: every serial write passes `CommandBroker.write()`: allowlist (`HELLO`, `ANGLES` 0–180; `MODE,*` blocked) → control mode (VISION / MOBILE / HOLD) → connection check.
* Mobile commands: authenticated operator + CSRF token + same-origin check, angle cap (default 100°), ramp (default 60°/s), time-to-live, stale-telemetry refusal,
  closing is blocked while FSR raw ≥ overload threshold (raw units, uncalibrated), heartbeat watchdog → HOLD.
* Command states: PENDING, EXECUTING, SENT, **OBSERVED** (servo feedback changed afterwards), NO_FEEDBACK, REJECTED, EXPIRED, ABORTED, DONE. Nothing is ever "confirmed".
* **HOLD is not an emergency stop.** It only stops the gateway from sending further angles; the servos keep holding the last angle and power stays on.
  A phone/internet button must never be relied on for safety.
* Smallest firmware changes that would make this properly safe: (1) a firmware watchdog that stops updating servos if no valid `ANGLES` arrives for ~500 ms;
  (2) a `STOP`/`RELEASE` command; (3) a hardware emergency stop (PCA9685 OE pin / servo power relay) that does not depend on software; (4) per-finger angle limits enforced in firmware.
* Not done: EMG control (only a blocked adapter skeleton exists — no EMG data is generated or trusted), user accounts beyond operator/viewer, audit-grade logging.

## 5. Test cases

Automated (`--selftest`): syntax; parameters; geometry + render; **DH forward kinematics vs mesh frames, coupling, joint limits, Jacobian, inverse solve**;
STL import; serial parsing + split chunks; thread shutdown; TX gate; exports; render at 3 sizes × 5 modes; config rules; data-quality flags; SQLite async + retention;
allowlist; broker safety/ramp/expiry/watchdog; HOLD aborts motion; faults + alert clearing; **web auth, lockout, CSRF, roles, host check, command flow**; TLS;
SMS (status truthfulness, rate limit, duplicate suppression, no secret leakage); EMG blocked; backup/rollback hash check.

Manual, with the hardware (hand **unloaded**):
1. Camera: start, confirm preview, hide the hand → TRACKING LOST banner and TX suspended.
2. Serial: connect, confirm FSR/SERVO/IMU rows update; unplug USB → SERIAL ERROR alert within ~2 s.
3. Local TX: arm (two clicks), move a hand in view, confirm `ANGLES` lines in the decoded-frame panel; verify direction of each servo before closing on anything.
4. Mobile (LAN): log in, wrong password ×5 → lockout; switch to MOBILE at the PC; Open/Close; pull the phone off Wi-Fi → mode falls to HOLD after the heartbeat timeout.
5. Overload: press the FSR → Close is refused, Open allowed.
6. SMS: use a test number; confirm provider status in the SMS log before trusting delivery.

## 6. Change log
* `ibpas_vision_x.py` 5.0 → 5.1: Denavit-Hartenberg kinematics replaces the angle-split model; Jacobian/IK helpers; KINEMATICS mode; workspace curves; STL/CAD-map import;
  telemetry/event listeners; replaceable command sender (so every write can pass the broker). FSR values are shown as raw (the old code called them newtons).
  Behaviour changes versus V3.5: TX disabled by default; per-frame CSV logging replaced by on-demand long-format export.
* `ibpas_platform.py` 1.0: new.

## 7. Rollback
```
python ibpas_platform.py --backup true_hand.py                       (creates backups\true_hand.py.<time>.bak + .sha256)
python ibpas_platform.py --rollback backups\true_hand.py.<time>.bak --to true_hand.py
```
The restore verifies the SHA-256, keeps the replaced file as `true_hand.py.replaced_<time>`, and never modifies the original during normal operation.
