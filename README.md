# OpenSteamTool — patched kernel build

Prebuilt, patched `OpenSteamTool.dll` (x86-64) plus the signature tables it needs,
and the source patch that differs from upstream
[`OpenSteam001/OpenSteamTool`](https://github.com/OpenSteam001/OpenSteamTool).

Everything here was compiled from a checkout of upstream commit `2a08b0b`
with two upstream PRs applied and one local fix on top.

## What is patched

| # | Change | Why |
|---|--------|-----|
| 1 | Upstream PR **#212** (`CNetPacket` layout) | Without it, enabling the `RecvPkt` pattern crashes the client on startup. |
| 2 | Upstream PR **#178** (recursive lua dir scan) | Lets `config/lua/**` be picked up recursively. |
| 3 | **Local**: `Hooks_NetPacket_UserStats::HandleRecv_ClientGetUserStatsResponse` bails out when CloudRedirect is inactive (see `patches/`) | The stock build replies "stats empty, eresult OK" to every app registered through a lua. The client trusts that answer and rewrites `appcache\stats\UserGameStats_<steamid>_<appid>.bin` into its 38-byte skeleton, so local achievements are wiped. This is the "achievements disappear after dropping and re-importing a game" report. |

Everything else — `RecvPkt`, manifest request-code handling, licence
impersonation — is stock upstream behaviour, just compiled with `RecvPkt`
actually enabled.

## Layout

```
dist/      OpenSteamTool.dll 1,500,672 B  sha256 9ce0acef3447b6ce73bb45b24a43d42337351e74a97bc5002af688eb93638e41
           dwmapi.dll 108,032 B     proxy DLL that loads OpenSteamTool into the client
           xinput1_4.dll 110,592 B  same, for the 32-bit surface
           opensteamtool.toml       runtime config (manifest provider, timeouts)
tables/    signature tables, keyed by the SHA-256 of the Steam binaries they match
patches/   0001-keep-local-stats-when-cloud-redirect-inactive.patch
BUILD.md   how it was built, and the five things that break on a clean clone
SHA256SUMS.txt
```

## Install

1. Close Steam.
2. Copy `dist/OpenSteamTool.dll`, `dist/dwmapi.dll`, `dist/xinput1_4.dll` and
   `dist/opensteamtool.toml` into the Steam install root (next to `steam.exe`).
3. Copy the tables so the folder names line up:

   ```
   <Steam>\opensteamtool\pattern\steamclient\<sha256-of-steamclient64.dll>.toml
   <Steam>\opensteamtool\pattern\steamui\<sha256-of-SteamUI.dll>.toml
   <Steam>\opensteamtool\ipc\steamclient\<sha256-of-steamclient64.dll>.toml
   ```

   Tables in this repo are named after the exact binaries they were dumped from:

   | file | steamclient64.dll / SteamUI.dll sha256 |
   |------|----------------------------------------|
   | `253e0a11…3142dbbc` | steamclient64.dll |
   | `cd9226db…7cabb610` | SteamUI.dll |

   A table is only used when its filename equals the SHA-256 of the binary in
   that Steam install. After a Steam update the file name changes; dump new
   patterns or wait for someone else to.
4. Start Steam. `logs/steamui_login.txt` should show
   `[ WaitingForLibraryReady ] SetLoginState: Success - OK`.

## Table keys must be FNV-1a 32

`[0x4B1B1D77]` is `CheckAppOwnership`. If a table is keyed with CRC32 or MD5
of the function name, the whole table is discarded and the client pops
`Missing Signatures` — the hash function is not interchangeable.

## Manifest request codes

`handle_send` for `ContentServerDirectory.GetManifestRequestCode#1` asks one of
three providers for the request code of a manifest gid. Only these names are
accepted in `opensteamtool.toml`; an unknown name logs a warning and silently
falls back to the first one.

| name | endpoint | state |
|------|----------|-------|
| `opensteamtool` | `https://manifest.opensteamtool.com/<gid>` | behind Cloudflare, returns 403 — default, effectively dead |
| `wudrm` | `http://gmrc.wudrm.com/manifest/<gid>` | plain decimal body, works |
| `steamrun` | `https://manifest.steam.run/api/manifest/<gid>` | `{"content":"<digits>"}`, works |

`dist/opensteamtool.toml` already points at `wudrm`.

## Known limits

- Signature tables are build-specific. A Steam client update invalidates them
  (and this whole build) until new patterns are dumped.
- `RecvPkt` is enabled, so injected package licences travel through the packet
  layer; that is what keeps a shared library from being reported as "in use by
  someone else".
- Cloud save redirection still needs the third-party `cloud_redirect.dll`; this
  build only hosts it. Without that DLL, `[cloud]` is a no-op — and thanks to
  patch 3, achievements are now left alone in that case instead of being wiped.

## 中文

这是 `OpenSteamTool` 的补丁版内核：上游 `2a08b0b` + PR #212（`CNetPacket` 布局）
+ PR #178（lua 递归）+ 一处本地修复。本地修复解决「删掉 lua 重新入库后成就清零」：
原版对任何通过 lua 注册过 depot 的游戏都回一个「成就全空 + OK」的响应，
客户端照单全收后会把 `appcache\stats\UserGameStats_<steamid>_<appid>.bin`
改写成 38 字节空壳；打了补丁后，CloudRedirect 没启用时直接放行原始响应，
本地成就不会再被覆盖。

签名表按 Steam 客户端二进制的 SHA-256 命名，只对同名二进制生效；Steam 一更新
就需要重新 dump。换码 provider 建议用 `wudrm`（`opensteamtool` 已被 Cloudflare 拦成 403）。
