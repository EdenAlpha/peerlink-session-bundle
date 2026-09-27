# eFootball 11.0.1 — M17: The KGS Wire Protocol (LIVE-VERIFIED)

## THE HEADLINE: first live contact with Konami's servers succeeded

`POST http://ntl.service.konami.net/ntl/api/GateInfo.php` with
`req=` + **lowercase-hex(JSON)** answers **200 OK**:

```
STATUS: 200
API_STATUS: 1
LOG_ACTIVE: 1
SERVER_TIME: 1790480086
PUT_LOG_URL: http://ntlus.service.konami.net/ntl/api/general/ReportLog.php
```

Wire details (all decoded from libUE4.so and verified live):
- Request builder: **0x7d0c06c** — snprintf template
  `{"titleCode":"%s","locale":"%s","version":"%s","extra":"%s","apiLevel":"%d"}`
  (string @ 0xbef54b), then hex-encode (alphabet @ 0x72e650), prefix `req=`
  (0x7d0c1d4), POSTed as x-www-form-urlencoded.
- Response parser (in the same function): reads `STATUS`, `API_STATUS`,
  `LOG_ACTIVE`, `SERVER_TIME`, `PUT_LOG_URL` — key:value lines. That is
  the complete GateInfo contract.
- The server rotates regional NTL endpoints: ntl / ntlus / ntleu / ntljp2
  .service.konami.net (all Apache+PHP, all reachable).
- titleCode values are not validated ("pes22", "pesam", "XWW020-E1" all
  accepted) — the field is used for routing/logging only.

## The KGS request format: MessagePack (fully decoded)

The Cmd*.php request bodies are **MessagePack maps**, written by:

| function | role |
|---|---|
| 0x74e38c4 | submit (called by all 328 builders, takes field count) |
| 0x74e65e8 | map header (0x80\|n / 0xde n16 / 0xdf n32) |
| 0x745cbd0 | str marker (0xa0\|n / 0xd9 / 0xda) |
| 0x745cd70 | int writer (pos-fixint / 0xd2 int32 BE) |
| 0x74e35a4 | envelope serializer |
| 0x74e3510 | envelope parser (responses) |
| 0x6f83ca0 | msgpack DOM parser (0x30-byte record array) |
| 0x76b1530 / 0x76b1b28 | login builder / serializer (23 body fields, w2=0x17) |

Envelope (always the first 6 map entries, in this order):
```
"msgid"       : "CmdLogin.php"          # the endpoint name
"rqid"        : int
"user_id"     : int
"session_id"  : str
"my_platform" : str
"s_keyword"   : str
<body fields ...>                          # count passed to submit
```
Example guest-token envelope (87 bytes, produced by peerlink/kgs.py):
```
86 a5"msgid" bc"CmdGetKgsGuestLoginToken.php" a4"rqid" 01
   a7"user_id" 00 aa"session_id" a0 ab"my_platform" a0 a9"s_keyword" a0
```
Login body fields (23): auth_code, hash, client_version, platform,
device_identifier, device, lang, country_code[], os_version, model_name,
gpu_name, soc_name, is_rooting, phy_mem_used_mib, phy_mem_available_mib,
app_storage_used_mib, app_storage_available_mib, data_storage_used_mib,
data_storage_available_mib, payment_store_link_send_info.

Client-side error response the game builds locally (0x74e4378):
`{msgid, rqid, result, msg, maintenance_eula_season}`.

## Runtime config (executed under Unicorn, dumped live)

Registrar 0x7d66580 (+ family) executed on the proven uc_loader2 harness;
config struct at **0xa4b0148** (slots every 0x18 bytes):

| slot | value |
|---|---|
| 0xa4b0188 | "https" |
| 0xa4b01a0 | "pesam.stun.service.konami.net" (LONG, heap ptr) |
| 0xa4b01b8 | " DEV1" |
| 0xa4b01d0 | "pes22-game.cs.konami.net" (LONG, heap ptr) |
| 0xa4b01e8 | "/" |
| 0xa4b0200 | "pes22" |

- URL resolver 0x7d6532c → URL builder 0x7d65884:
  **URL = scheme + "://" + host + path + titleCode**
  = `https://pes22-game.cs.konami.net/pes22` (port 443, config@0xa4b0140).
- RSA-2048 PEM (800 bytes) is copied at boot into config+0x298
  (0x7d66e2c), overridable by a **ChangeServer.bin** file next to the
  game data (QA hook, 0x7d6677c reads it; if absent, embedded key used).
- "PES/1.0 (…)" User-Agent is built by 0x7dbc430 (the send preamble).
- 0x7dbc7f8 concat chain: request URL = <base> + "/" + <msgid>.

## The remaining unknown: the live API base

`https://pes22-game.cs.konami.net/pes22/CmdLogin.php` (and /CmdLogin.php,
/api/…, /kgs/…, /XWW020-E1/…, and the same on info.service.konami.net)
answer **404 "File not found."** from nginx+PHP-FPM — the server is
alive and PHP-routed, but the production path differs.

The game itself sources the request URL from a **4-entry env table at
0xa4cff68** (0x814a04c getter; flag at +0x2c, SSO string at +0x30),
populated at runtime by the online session — i.e. the real base URL is
delivered by bootstrap data we have not yet captured. The static config
default (pes22-game.cs.konami.net/pes22) is either a pre-prod default
(" DEV1" tag!) or is overridden before first login.

Attack vectors for next session:
1. Find the 0xa4cff68 table writer (search stores to page 0xa4cf000 with
   string assigns; candidates in the 0x8123xxx–0x814Axxx cluster).
2. Drive the 12-state online bootstrap state machine (0x7dc7164, jump
   table @0xc905b8) under Unicorn with a fabricated session and dump the
   URL after state 5 (session create, vtable 0x97cf020).
3. The gRPC layer (OnlineSystemgRPCClient.cpp, command_service.pb.cc)
   may itself reveal the target in its channel args.

## Source-tree map (leaked dev paths — navigation gold)

```
pes\Game\Online\OnlineSystem\Api\OnlineSystemApiManagerVer2.cpp   → 0x7afe21c
pes\Game\Online\OnlineSystem\Api\OnlineSystemApiManagerver3.cpp   → 0x7b0766c
pes\Game\Online\OnlineSystem\gRPC\OnlineSystemgRPCClient.cpp      → 0x7ddc444
pes\Game\Online\OnlineSystem\gRPC\ProtocolBuffer\command_service.pb.cc
pes\Game\Online\OnlineSystem\Multiplay\OnlineSystemMultiplay.cpp
pes\Game\Online\OnlineMode\Task\Lobby\OnlineModeTaskLobbyRoom.cpp  → 0x7a1bca0
pes\Game\Online\OnlineMode\Task\Match\OnlineModeTaskMatchSession.cpp → 0x7a559a8
pes\Game\Match\Online\MatchOnlineNegotiator.cpp / MatchOnlineSession.cpp
```
Logger = 0x698d3a4(file, line, …); 259 unique dev paths extracted.

## Transport inventory

- **NTL/curl HTTP class** vtable 0x98225a0 (15 methods): GET 0x7d03b68,
  POST 0x7d03c68, KGS POST w/ headers 0x7d04148 (CURLOPT_URL 10002,
  POSTFIELDS 10015, POSTFIELDSIZE 60, HTTPHEADER 10023 via
  curl_easy_setopt = **0x6886498**; curl static-linked + OpenSSL).
- **UPnP/SOAP variant** 0x7d03e10 (Content-Type: text/xml, M-POST,
  SOAPAction) — NAT traversal only.
- **gRPC** for match coordination (MatchControlSession et al).
- Request queue + send: 0x7b345e8 (manager getter) / 0x7b345f4 (send).

## Deliverable

`peerlink_restored/peerlink/kgs.py` — the Python KGS client:
- `gate_info()` — LIVE-VERIFIED 200 OK
- `build_request()` / `parse_response()` — byte-exact msgpack
  (verified: envelope hex `86a56d73676964…` matches the game's writer)
- `login_request()` — the 23-field CmdLogin.php body
- `CMD` — the full endpoint map (login/rooms/match/tutorial)
- `post_cmd()` / `probe_cs()` — CS transport + URL research helper
- embedded RSA-2048 public key

## Status board

| milestone | state |
|---|---|
| GateInfo live contact | ✅ DONE (200 OK, format verified) |
| msgpack request format | ✅ DONE (byte-exact, round-trip tested) |
| runtime config dump under Unicorn | ✅ DONE |
| RSA + ChangeServer.bin flow | ✅ decoded |
| CS API base URL | ⏳ 404s on static default; table writer hunt continues |
| guest login → room create → match | blocked on URL above |
