# GitHub Post — HB3 Motion/Person Detection Fix

## Where to post

1. **Comment on PR #888** in `bropat/eufy-security-client` — this is the existing (incomplete) fix for the same issue
2. **New issue in `bropat/hassio-eufy-security-ws`** — specifically about the HAOS add-on not working with HB3
3. **HA Community forum** — `eufy_security` custom component thread, lots of HB3 users affected

---

## Draft (PR #888 comment or new issue body)

---

### HomeBase 3 motion/person detection fix — complete 3-file patch for v3.8.0

**Summary**: PR #888 is on the right track but incomplete for `eufy-security-client` v3.8.0. Two additional files need patching (`station.js` and `eufysecurity.js`) to wire the new `"push notification"` event through the routing chain. Here's the complete fix that confirmed working.

---

#### Problem

Cameras connected to HomeBase 3 never trigger motion or person detection events in Home Assistant. The `eufy_security` custom component's binary sensors (`motion_detected`, `person_detected`) never flip.

**Root cause**: HB3 delivers all camera events via P2P only (`is_push: 0` — no cloud push for HB3). The P2P packet is `CMD_NOTIFY_PAYLOAD` (cmd 1351) wrapping inner cmd **2037** (`CMD_CAMERA_PUSH_NOTIFY`). In v3.8.0, `handleDataControl` has no case for cmd 2037, so it falls to:

```
"Not implemented" debug log → event dropped
```

#### Why PR #888 alone doesn't fix it in v3.8.0

PR #888 patches `session.ts` to emit `"push notification"` when cmd 2037 arrives. That's necessary but not sufficient — in v3.8.0, nothing *listens* for that event downstream:

- `Station` (in `station.js`) does not listen for `"push notification"` from `p2pSession`
- `EufySecurity` (in `eufysecurity.js`) does not listen for `"push notification"` from `Station`

The event emits into the void. `Camera.processPushNotification` is never called. No property updates happen.

#### Complete fix — 3 files

**1. `src/p2p/session.ts` (same as PR #888, adapted for v3.8.0)**

In `handleDataControl`, inside the `CMD_NOTIFY_PAYLOAD` handler, add before the final `else { logger.debug(...Not implemented...) }`:

```typescript
} else if (json.cmd === CommandType.CMD_CAMERA_PUSH_NOTIFY) {
    const innerPayload = parseJSON(
        typeof json.payload === "string" ? json.payload : JSON.stringify(json.payload),
        rootP2PLogger
    );
    if (innerPayload !== undefined) {
        rootP2PLogger.debug(
            `Handle DATA ${P2PDataType[message.dataType]} - CMD_NOTIFY_PAYLOAD - CMD_CAMERA_PUSH_NOTIFY (HB3)`,
            {
                stationSN: this.rawStation.station_sn,
                event_type: innerPayload.event_type,
                device_sn: innerPayload.device_sn,
            }
        );
        this.emit("push notification", { ...innerPayload, type: innerPayload.msg_type, station_sn: this.rawStation.station_sn });
    }
}
```

**2. `src/http/station.ts` — NEW (not in PR #888)**

In the `Station` constructor, after all existing `p2pSession.on(...)` event listeners, add:

```typescript
// Route P2P camera push notifications (CMD_CAMERA_PUSH_NOTIFY / cmd 2037) from HB3
this.p2pSession.on("push notification", (message: PushMessage) => {
    this.processPushNotification(message);
    this.emit("push notification", message);
});
```

**3. `src/eufysecurity.ts` — NEW (not in PR #888)**

In the station setup block (where all the `station.on(...)` listeners are added), add:

```typescript
// Route HB3 P2P push notifications into the device push notification chain
station.on("push notification", (message: PushMessage) => this.onPushMessage(message));
```

#### How the full event chain works after the fix

```
HB3 P2P → CMD_NOTIFY_PAYLOAD (1351)
  inner cmd 2037 → session.ts handler
    emit("push notification", { msg_type:18, event_type:3101/3102, device_sn, station_sn })
      station.ts listener
        processPushNotification(msg) [no-op for generic Station]
        emit("push notification", msg)
          eufysecurity.ts listener
            onPushMessage(msg)
              for each device: device.processPushNotification(station, msg, eventDuration)
                Camera class (search device.js for `CusPushEvent.SECURITY`):
                  else if (msg.msg_type === DeviceType.HB3)  // 18
                    switch (msg.event_type):
                      MOTION_DETECTION (3101): updateProperty(DeviceMotionDetected, true)
                      FACE_DETECTION   (3102): updateProperty(DevicePersonDetected, true)
                        → HA binary sensor flips → automation fires
```

#### Event payload structure (observed in production)

```json
{
  "msg_type": 18,
  "event_type": 3102,
  "device_sn": "T8160P...",
  "name": "Front Door",
  "channel": 3,
  "cipher": 0,
  "create_time": 1779691038912,
  "trigger_time": 1779691038912,
  "file_path": "/zx/hdd_data0/Camera03/...",
  "push_count": 1,
  "notification_style": 1,
  "storage_type": 1,
  "vision": 1,
  "mode": 1
}
```

#### Note on `CommandType.CMD_CAMERA_PUSH_NOTIFY`

Also needs to be added to the enum in `src/p2p/types.ts`:

```typescript
CMD_CAMERA_PUSH_NOTIFY = 2037,
```

(This is already in PR #888.)

#### Note on cmd 1312 (bonus finding)

After recording completes, HB3 sends a second P2P notification via inner cmd **1312** with the same payload (base64-encoded, with additional `rec_content` metadata). This is also unhandled and hits "Not implemented." Since cmd 2037 fires first at motion-start and is sufficient to trigger the property update, cmd 1312 dropping is not a functional problem. But it could be handled too with an additional base64-decode step.

#### Test setup

- HomeBase 3 (T8030P, firmware 3.x)
- Multiple T8160P cameras (EufyCam 2C-style, HB3-connected)
- `eufy-security-client` v3.8.0
- `eufy-security-ws` v2.1.0
- Home Assistant OS, `eufy_security` custom component v8.2.4

All cameras now correctly trigger motion and person detection binary sensors in HA, and HA automations fire reliably.
