# PeerLink — session bundle delivery links

## Bundle v9 (M14: net layer + full-stack engine lockstep, 1200/1200 bit-exact)
- **PRIMARY: https://gofile.io/d/jYuvKEDs**  (PeerLink_session_bundle_v9.zip, 6.3 MB, md5 79452307a7d6640f7240e6ee1c3b4055)
- backup 1: https://tmpfiles.org/wUwuEYpaReYU/peerlink_session_bundle_v9.zip (short retention)
- backup 2: https://litter.catbox.moe/29q2nr.zip (72h)

## What v9 proves (all commands inside the bundle)
1. demo_match.py           — 1200/1200 ticks bit-exact (net layer, threads)
2. demo_process_mode.py    — 300/300 bit-exact (net layer, processes)
3. engine_twinrun.py x2    — engine deterministic across independent instances
4. demo_engine_net.py      — FULL STACK: original libUE4.so engine, two
   processes, real UDP room-code friend match: 1200/1200 ticks, 62 kicks
   via the game's own installer, 44/44 checksum exchanges, zero divergence

## GitHub
- repo: https://github.com/EdenAlpha/peerlink-session-bundle (has v8;
  v9 zip ready to add — commit 9cdccfb in the local repo)
- prior session bundles: v6/v7/v8 in this directory

## Older links (prior sessions)
- v8: see PeerLink_session_bundle_v8.zip in this directory
- v7 mirror (may have expired): https://gofile.io/d/jRMj05Ng
- v7 alt (may have expired): https://temp.sh/aKtKm
