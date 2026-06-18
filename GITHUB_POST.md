# GitHub Post — HB3 Motion/Person Detection (PR #888 confirmation + released-build backport)

## Where to post

1. **Comment on [PR #888](https://github.com/bropat/eufy-security-client/pull/888)** in `bropat/eufy-security-client` — independent confirmation that the approach works, plus a pointer to a backport for people on the released build. (Primary — post this first.)
2. *(Hold)* New issue in `bropat/hassio-eufy-security-ws` — only if #888 stalls; link back to it rather than duplicating.
3. *(Hold)* HA Community forum — best once there's a merged fix or an end-user-ready add-on to point at.

> **Accuracy note (important):** PR #888 is **functionally complete on `develop`** — it patches `types.ts`, `interfaces.ts`, `session.ts`, *and* `eufysecurity.ts` (which wires the event through via `station.p2pSession.on("push notification") → onPushMessage`). Do **not** claim the PR is "missing downstream wiring" — it isn't. The separate add-on in this repo exists only because the PR targets `develop` (unmerged, red CI) and its `session.ts` depends on a `./adts` module absent in the released 3.8.0, so the fix has to be re-derived against the released build.

---

## Draft (comment on PR #888)

---

### Confirming this fixes HB3 detection — plus a backport for the released 3.8.0 build

Thanks for tracking this down — I can confirm your approach fixes HomeBase 3 motion/person detection end-to-end. Independent test setup:

- HomeBase 3 (T8030P) with multiple T8160P cameras (HB3-paired)
- `eufy-security-client` 3.8.0 / `eufy-security-ws` 2.1.0
- Home Assistant OS, `eufy_security` custom component 8.2.4

With the cmd `2037` (`CMD_CAMERA_PUSH_NOTIFY`) handler emitting `"push notification"` and the `eufysecurity.ts` listener routing it through `onPushMessage`, the `device.js` Camera HB3 branch (`msg_type === DeviceType.HB3` → `MOTION_DETECTION 3101` / `FACE_DETECTION 3102`) fires correctly. HA binary sensors flip and automations run reliably.

**For anyone stuck until this merges:** the fix can't simply be dropped onto the released 3.8.0 build — this PR's `session.ts` pulls in `./adts`, which doesn't exist in 3.8.0, and the changes only live on `develop`. I backported the same fix to the released build as a Home Assistant **local add-on** (patched build files + Dockerfile + notes):

➡️ https://github.com/davesmithoutdoors/eufy-security-ws-hb3-fix

One implementation note in case it's useful: in the backport I route the event the "long way" (`p2pSession` → `Station` relay/re-emit → `EufySecurity` `station.on(...)`) to match how the other P2P events propagate in the released code; your direct `station.p2pSession` listener is more concise. Both reach `onPushMessage`.

**Bonus finding — cmd 1312.** After a recording completes, HB3 sends a second P2P notification via inner cmd **1312** (same event payload, base64-encoded, plus `rec_content` metadata with file paths). It's also unhandled and hits "Not implemented." Not a functional problem since cmd 2037 fires first at motion-start, but it could be handled too (one extra base64-decode of the nested `payload.payload`) if you want full coverage.

Happy to help with the failing CI checks if that's what's blocking merge.

---

## Reference — observed cmd 2037 payload (production)

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

- `msg_type: 18` = `DeviceType.HB3`
- `event_type: 3101` = `MOTION_DETECTION`
- `event_type: 3102` = `FACE_DETECTION` (person)
