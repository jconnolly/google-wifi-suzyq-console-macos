# Mesh setup — 802.11s wireless backhaul

How to add an OpenWrt-flashed Google WiFi puck as a wireless mesh node
to an existing OpenWrt primary. Uses **802.11s mesh on 5 GHz** for the
backhaul + identical client APs on both bands so phones/laptops see one
seamless network.

This is what `mesh1` (MAC 58:cb:52:aa:16:26) was configured for during
the 2026-05-26 flash session — see
[`captures/flash-session-20260526-1815/`](../captures/flash-session-20260526-1815/)
for full input/output logs.

## Concept

- **Client SSID** = what phones see. Same name + WPA2 password on every
  puck so handoff is seamless. Example: `Cinque Veddy Much`.
- **Mesh ID** = backhaul-only handle, invisible to clients. Pucks with
  same mesh_id form a mesh. Example: `cinque-mesh`.
- **Mesh SAE key** = WPA3-SAE password used between pucks. Different
  from the client password. Random string is fine — only the pucks need
  to know it.

The mesh runs on radio1 (5 GHz, channel 36) alongside the 5 GHz client
AP — IPQ4019 + ath10k-ct supports `AP + mesh point` on the same radio.

## On a mesh node (the puck you just flashed)

After flashing, while still reachable at `192.168.1.1`:

```sh
ssh root@192.168.1.1

# identify this puck
uci set system.@system[0].hostname='mesh<N>'      # mesh1, mesh2, ...
uci commit system

# both radios on, country US
uci set wireless.radio0.country='US'
uci set wireless.radio0.disabled='0'
uci set wireless.radio1.country='US'
uci set wireless.radio1.disabled='0'

# 2.4 GHz client AP
uci set wireless.default_radio0.disabled='0'
uci set wireless.default_radio0.ssid='Cinque Veddy Much'
uci set wireless.default_radio0.encryption='psk2'
uci set wireless.default_radio0.key='<CLIENT_PSK>'
uci set wireless.default_radio0.ieee80211k='1'

# 5 GHz client AP
uci set wireless.default_radio1.disabled='0'
uci set wireless.default_radio1.ssid='Cinque Veddy Much'
uci set wireless.default_radio1.encryption='psk2'
uci set wireless.default_radio1.key='<CLIENT_PSK>'
uci set wireless.default_radio1.ieee80211k='1'

# 5 GHz mesh point (the backhaul)
uci set wireless.mesh=wifi-iface
uci set wireless.mesh.device='radio1'
uci set wireless.mesh.network='lan'
uci set wireless.mesh.mode='mesh'
uci set wireless.mesh.mesh_id='cinque-mesh'
uci set wireless.mesh.encryption='sae'
uci set wireless.mesh.key='<MESH_SAE_KEY>'

# only primary serves DHCP — mesh nodes are dumb APs
uci set dhcp.lan.ignore='1'

uci commit wireless
uci commit dhcp
wifi reload
```

**Gotcha:** if you're on `wpad-basic-mbedtls` (the default), do NOT add
`bss_transition='1'` — that option needs `wpad-full`. ieee80211k works
in basic.

Once the primary is also configured (next section), switch this puck
from static LAN to DHCP so it pulls an IP from the primary:

```sh
uci set network.lan.proto='dhcp'
uci commit network
reboot
```

After reboot the puck will get an IP on the primary's subnet via the
mesh, and you reach it at whatever lease the primary assigns. (Find it
on the primary with `cat /tmp/dhcp.leases | grep <hostname>`.)

## On the primary (the existing OpenWrt router)

Same `mesh_id` + same `MESH_SAE_KEY`. The primary keeps DHCP server
enabled, keeps its WAN, and adds the mesh interface on its 5 GHz radio.

```sh
ssh root@<primary-ip>          # 192.168.10.1 in this setup

uci set wireless.mesh=wifi-iface
uci set wireless.mesh.device='radio1'
uci set wireless.mesh.mode='mesh'
uci set wireless.mesh.network='lan'
uci set wireless.mesh.mesh_id='cinque-mesh'
uci set wireless.mesh.encryption='sae'
uci set wireless.mesh.key='<MESH_SAE_KEY>'
uci commit wireless
wifi reload
```

That's the entire primary-side change. Mesh peers form within a few
seconds (verify with `iw dev phy1-mesh0 station dump` — it should show
each remote puck once joined).

## Where the secrets live

The actual `<CLIENT_PSK>` and `<MESH_SAE_KEY>` for this setup are saved
locally at `~/openwrt-staging/mesh-secrets.txt` (mode 600, NOT in this
repo). Don't commit them.

## Verifying the mesh

On any puck after both ends are configured:

```sh
iw dev phy1-mesh0 station dump  # one entry per remote mesh peer
iw dev phy1-mesh0 mpath dump    # mesh routing table
```

`station dump` is the truth — if it shows a peer with a reasonable RSSI
and `tx packets / rx packets` going up, the backhaul works.

## Why 802.11s and not WDS / batman-adv / DAWN

- **802.11s** is the IEEE-standard mesh on top of WiFi. Single radio
  can do AP + mesh simultaneously. SAE handles auth. Easy uci config.
  Each mesh hop halves throughput, so don't chain more than 2–3 hops
  deep.
- **WDS** is a 2000s-era proprietary "wireless bridge" mode. Works only
  AP-to-AP and is fragile. Skip.
- **batman-adv** is a layer-2 mesh routing protocol *on top of* a mesh
  link (802.11s or ad-hoc). Useful for >3 nodes or complex topology.
  Overkill for a 2–3 puck home setup.
- **DAWN** is a client-steering daemon — it's about *which AP a client
  associates to*, not about backhaul. Orthogonal to mesh; add later if
  steering is poor.
