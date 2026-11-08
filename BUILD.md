# Building from a clean clone

Verified with MSVC 14.44 (VS Build Tools 2022), CMake 4.4, Git for Windows,
on Windows 10/11 x64, Python 3.10 for the helper scripts.

```bat
git clone https://github.com/OpenSteam001/OpenSteamTool.git
cd OpenSteamTool
git fetch origin pull/212/head:pr212 && git merge --no-edit pr212
git fetch origin pull/178/head:pr178 && git merge --no-edit pr178
git apply patches/0001-keep-local-stats-when-cloud-redirect-inactive.patch

cmake -S . -B build -G "Visual Studio 17 2022" -A x64 ^
      -DGIT_EXECUTABLE="C:/Program Files/Git/cmd/git.exe" ^
      -DCMAKE_CXX_FLAGS="/utf-8"
cmake --build build --config Release --target OpenSteamTool
```

Output: `build/Release/OpenSteamTool.dll`, plus `dwmapi.dll` and `xinput1_4.dll`.

## Five things that break on a clean clone

1. **`GIT_EXECUTABLE` not found.** CMake bakes a version string from `git describe`.
   Pass the full path; without it the configure step fails.
2. **`add_library(lua_static)` with no sources.** The lua target is filled by a
   `file(GLOB ...)`. If the checkout path contains `[` or `]`, CMake treats those
   as glob character-class delimiters, matches nothing, and the target is empty.
   Either move the checkout to a bracket-free path, or replace the glob in
   `cmake/Lua.cmake` with an explicit `set(LUA_CORE_SOURCES ...)` listing the 32
   `.c` files.
3. **Mojibake / syntax errors in `Hooks_Manifest.cpp`.** The sources are UTF-8
   but MSVC assumes the local ANSI code page. Always add `-DCMAKE_CXX_FLAGS="/utf-8"`.
4. **toml++ in no-exception mode.** The project builds toml++ with exceptions off,
   so `try { toml::parse(...) } catch (...)` does not compile. Three call sites
   need the result-style API:
   `auto parsed = toml::parse(path); if (parsed.failed()) { ... } parsed.table()` —
   in `IPCLoader.cpp`, `PatternLoader.cpp`, `Config.cpp`.
5. **Stale `CMakeCache.txt`.** An old cache keeps `/DTOML_EXCEPTIONS=1` alive and
   the exception errors come back. Overwrite the whole flag string
   (`-DCMAKE_CXX_FLAGS="/utf-8"`) or delete the build directory.

## Verifying the `RecvPkt` hook landed

The pattern is `48 8B C4 55 48 8D A8 E8 FB FF FF`. In the client build used here,
`steamclient64.dll` has it at RVA `0x5BCA50`, and the file offset is RVA − `0xC00`
because `.text` is mapped 0xC00 above its raw offset:

```
python -c "d=open('steamclient64.dll','rb').read(); print(d[0x5BBE50:0x5BBE5A].hex())"
# 488bc455488da8e8fbffff48
```

`strings -a OpenSteamTool.dll | grep RecvPkt` should also hit.
