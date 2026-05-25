FROM bropat/hassio-eufy-security-ws-amd64:2.1.0

# Patch: handle CMD_CAMERA_PUSH_NOTIFY (cmd 2037) for HomeBase 3 P2P motion/person events
# Source: https://github.com/bropat/eufy-security-client/pull/888
# Without this, all motion/person detection events from HB3-connected cameras are silently dropped.
# Also patches station.js and eufysecurity.js to route the P2P push notification to device handlers.

COPY session.js      /usr/src/app/node_modules/eufy-security-client/build/p2p/session.js
COPY types.js        /usr/src/app/node_modules/eufy-security-client/build/p2p/types.js
COPY station.js      /usr/src/app/node_modules/eufy-security-client/build/http/station.js
COPY eufysecurity.js /usr/src/app/node_modules/eufy-security-client/build/eufysecurity.js
