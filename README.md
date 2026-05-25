# eufy-security-ws HB3 Fix — Local HAOS Add-on

**Status: Working as of 2026-05-25**

Cameras connected to a **HomeBase 3 (HB3)** NVR silently drop all motion and person detection events in `eufy-security-ws` v2.1.0 / `eufy-security-client` v3.8.0. This repository contains the patched local add-on that fixes it.

---

## Problem

HomeBase 3 delivers all camera events to the eufy-security-ws process via **P2P only** (`is_push: 0` — Eufy does not cloud-push HB3 events at all). The P2P path delivers a `CMD_NOTIFY_PAYLOAD` (cmd 1351) outer frame wrapping an inner command **2037** (`CMD_CAMERA_PUSH_NOTIFY`) with a JSON payload like:

```json
{ "msg_type": 18, "event_type": 3102, "device_sn": "T8160P...", "name": "Front Door", ... }
```

- `msg_type: 18` = `DeviceType.HB3`
- `event_type: 3101` = `HB3PairedDevicePushEvent.MOTION_DETECTION`
- `event_type: 3102` = `HB3PairedDevicePushEvent.FACE_DETECTION` (person detection)

**eufy-security-client v3.8.0 has no handler for cmd 2037.** The `handleDataControl` switch falls through to a `"Not implemented"` debug log and the message is dropped. No event reaches HA. No binary sensor flips. No automations fire.

The upstream fix exists in [bropat/eufy-security-client PR #888](https://github.com/bropat/eufy-security-client/pull/888) but was never merged.

---

## Root Cause Deep-Dive

### Event flow (stock v3.8.0 — broken)

```
HB3 station
  └─ P2P → CMD_NOTIFY_PAYLOAD (1351)
               └─ inner cmd 2037 payload
                    ← handleDataControl: "Not implemented" → DROPPED
```

### Event flow (patched — working)

```
HB3 station
  └─ P2P → CMD_NOTIFY_PAYLOAD (1351)
               └─ inner cmd 2037 payload
                    ← session.js handler: emit("push notification", {...innerPayload, station_sn})
                         ← station.js: processPushNotification(msg) + emit("push notification", msg)
                              ← eufysecurity.js: onPushMessage(msg)
                                   ← device.js Camera.processPushNotification:
                                        else if (msg.msg_type === DeviceType.HB3):  // 18
                                            case MOTION_DETECTION (3101): DeviceMotionDetected = true
                                            case FACE_DETECTION   (3102): DevicePersonDetected = true
                                                 ← HA binary sensor flips → automation fires
```

Three files needed patching in the installed `eufy-security-client` build (not just the two in PR #888):

| File | Change |
|------|--------|
| `p2p/session.js` | Add `else if (json.cmd === 2037)` handler in `handleDataControl`; parse inner JSON; emit `"push notification"` |
| `http/station.js` | Add `p2pSession.on("push notification", ...)` listener; route through `processPushNotification` and re-emit |
| `eufysecurity.js` | Add `station.on("push notification", ...)` listener; call `onPushMessage` |

PR #888 only patched `session.ts` and `types.ts` (adds the CommandType enum entry). It does NOT add the station.js and eufysecurity.js listeners needed to propagate the event through the chain in v3.8.0.

---

## Why a Local Add-on?

The official add-on (`402f1039_eufy_security_ws`) installs from the bropat repository and can't be modified in place. HAOS local add-ons (`/addons/<slug>/`) allow a custom `Dockerfile` + build context. The patched add-on:

- Uses the official Docker Hub image as the base (`bropat/hassio-eufy-security-ws-amd64:2.1.0`)
- Overwrites exactly the four JavaScript files that need patching
- Is pinned to v2.1.0 — upgrading the base image requires re-extracting and re-patching

---

## Deployment

### Files on server (build context)

```
/opt/docker/eufy-patch/docker-build/
  Dockerfile
  session.js      → patched p2p/session.js
  types.js        → patched p2p/types.js (adds CMD_CAMERA_PUSH_NOTIFY = 2037 to CommandType enum)
  station.js      → patched http/station.js
  eufysecurity.js → patched eufysecurity.js
```

### Files on HAOS

```
/addons/eufy_ws_hb3fix/
  config.yaml     (slug: eufy_ws_hb3fix, version: 2.1.0-hb3fix4, init: false)
  Dockerfile      (same as docker-build/Dockerfile)
  session.js
  types.js
  station.js
  eufysecurity.js
```

### How to rebuild after changing a patch file

1. Copy updated file to HAOS: `scp <file> root@<haos-ip>:/addons/eufy_ws_hb3fix/<file>` (replace `<haos-ip>` with your Home Assistant OS IP address)
2. Bump `version` in `/addons/eufy_ws_hb3fix/config.yaml`
3. From HAOS shell:
   ```sh
   curl -s -X POST \
     -H "Authorization: Bearer $SUPERVISOR_TOKEN" \
     http://supervisor/store/reload
   curl -s -X POST \
     -H "Authorization: Bearer $SUPERVISOR_TOKEN" \
     http://supervisor/addons/local_eufy_ws_hb3fix/update
   ```
4. Start the add-on from HA UI → Settings → Add-ons → eufy-security-ws (HB3 fix)

---

## Bugs Encountered During Development

### Bug 1 — `s6-overlay-suexec: fatal: can only run as pid 1`

**Symptom**: Add-on starts then immediately crashes with this error.

**Cause**: The Docker Hub base image uses **s6-overlay v3** as its init system. HAOS supervisor's default `init: true` adds its own init wrapper, which prevents s6-overlay from running as PID 1.

**Fix**: Add `init: false` to `config.yaml`. This tells the supervisor to let the container's own entrypoint be PID 1.

---

### Bug 2 — `Cannot find module './adts'`

**Symptom**: Add-on starts then crashes with a Node.js module resolution error.

**Cause**: Took `session.js` from [jwtobias's PR #888 fork](https://github.com/jwtobias/eufy-security-client/tree/HB3-Notify) which was based on a newer `develop` branch that depends on a module (`./adts`) not present in v3.8.0.

**Fix**: Extract the **original v3.8.0 session.js** from the base Docker image using:
```sh
docker create --name eufy-tmp bropat/hassio-eufy-security-ws-amd64:2.1.0
docker cp eufy-tmp:/usr/src/app/node_modules/eufy-security-client/build/p2p/session.js /tmp/session-orig.js
docker rm eufy-tmp
```
Then apply a **minimal patch** to the original file rather than replacing it wholesale.

---

### Bug 3 — `ghcr.io 403 Forbidden` on base image pull

**Symptom**: `docker buildx build --pull` fails when Dockerfile uses `FROM ghcr.io/bropat/...`.

**Cause**: The GitHub Container Registry (ghcr.io) image requires authentication/permissions.

**Fix**: Use the Docker Hub image (`bropat/hassio-eufy-security-ws-amd64:2.1.0`) instead.

---

### Bug 4 — PR #888 fix is incomplete for v3.8.0 (only 2 of 3 needed files)

**Symptom**: After applying PR #888's session.js patch (adapted for v3.8.0), the `"push notification"` event IS emitted from `P2PClientProtocol`, but nothing happens downstream.

**Cause**: In v3.8.0, `Station` does not listen for `"push notification"` from `p2pSession`, and `EufySecurity` does not listen for `"push notification"` from `Station`. The event is emitted into the void.

**Fix**: Patch both `station.js` and `eufysecurity.js` to wire the event through.

---

### Non-Bug — `type` field in emitted push notification

**Initially suspected**: Including `type: innerPayload.msg_type` (= 18) in the emitted push notification would cause `Camera.processPushNotification` to enter the CUS cloud push branch instead of the HB3 branch.

**Actually**: The `Camera` class's condition (line 2797 of device.js) is:
```js
if (message.event_type === CusPushEvent.SECURITY && message.device_sn === this.getSerial())
```
NOT `if (message.type !== undefined)`. Since `event_type: 3102` ≠ `CusPushEvent.SECURITY` (= 1), the first condition is FALSE regardless of whether `type` is set, and the `else if (msg_type === DeviceType.HB3)` branch runs correctly. The `type` field is harmless.

---

## Patch Details

### session.js

Location in tree: `node_modules/eufy-security-client/build/p2p/session.js`

Find the `CMD_NOTIFY_PAYLOAD` handler block (search for `"storage info hb3"`). The block ends with `else { rootP2PLogger.debug(...Not implemented...)`. Add this `else if` immediately before the final `else`:

```js
else if (json.cmd === 2037) {
    // CMD_CAMERA_PUSH_NOTIFY (2037): P2P camera push notification from HB3 station
    // Fix for HB3 motion/person detection events (event_type 3101/3102)
    // See: https://github.com/bropat/eufy-security-client/pull/888
    const innerPayload = (0, utils_3.parseJSON)(
        typeof json.payload === "string" ? json.payload : JSON.stringify(json.payload),
        logging_1.rootP2PLogger
    );
    if (innerPayload !== undefined) {
        logging_1.rootP2PLogger.debug(
            `Handle DATA ${types_1.P2PDataType[message.dataType]} - CMD_NOTIFY_PAYLOAD - CMD_CAMERA_PUSH_NOTIFY (HB3)`,
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

### types.js

Location: `node_modules/eufy-security-client/build/p2p/types.js`

In the `CommandType` enum, add the missing entry (find `CMD_NOTIFY_PAYLOAD = 1351` and add near it):

```js
CommandType[CommandType["CMD_CAMERA_PUSH_NOTIFY"] = 2037] = "CMD_CAMERA_PUSH_NOTIFY";
```

### station.js

Location: `node_modules/eufy-security-client/build/http/station.js`

In the `Station` constructor, after all existing `this.p2pSession.on(...)` listeners (the last one is `"sequence error"`), add:

```js
// HB3 fix: route P2P camera push notifications (cmd 2037) through the push notification chain
this.p2pSession.on("push notification", (message) => {
    this.processPushNotification(message);
    this.emit("push notification", message);
});
```

### eufysecurity.js

Location: `node_modules/eufy-security-client/build/eufysecurity.js`

In the `addStation` setup block, after the `station.on("storage info hb3", ...)` listener (and before `this.addStation(station)`), add:

```js
// HB3 fix: route P2P camera push notifications through the device push notification chain
station.on("push notification", (message) => this.onPushMessage(message));
```

---

## Notes on cmd 1312

In addition to cmd 2037 (immediate at-motion-start notification), the HB3 also sends cmd 1312 via P2P after recording completes. This has a different envelope structure:

```
CMD_NOTIFY_PAYLOAD (1351)
  └─ inner: { cmd: 1312, payload: { type: "19", ..., payload: "<base64>" } }
                                                                └─ base64 decodes to same msg_type/event_type JSON
```

cmd 1312 is also unhandled and hits "Not implemented." Since cmd 2037 fires first and successfully triggers the HA automation, cmd 1312 dropping is not a functional problem. It could be handled in a future patch for completeness (same decode-and-emit pattern, but requires an additional base64 decode of the nested `payload.payload` field).

---

## Upstream Status

- [bropat/eufy-security-client PR #888](https://github.com/bropat/eufy-security-client/pull/888): Open, unmerged as of 2026-05-25
- [bropat/hassio-eufy-security-ws](https://github.com/bropat/hassio-eufy-security-ws): No equivalent issue/PR filed

This fix should be reported upstream. See `GITHUB_POST.md` for a draft issue/PR comment.
