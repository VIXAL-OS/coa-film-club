# Film Club patch for CoA x64

Film Club adds the **/film** panel, a placeable world screen, and playback controls for Azeroth Plex and Video Cinema to a compatible local CoA x64 source tree. The download contains the Film Club patch, this build guide, and the license text for the helper-derived changes. You supply the base source, tools, dependencies, and matching game data you are authorized to use.

The three edits to existing client files contain only the added integration calls and includes. The rest of the patch adds Film Club source and the changes needed to build its two media workers. The two IPC headers are generated locally from the media projects.

## Prerequisites

Use Windows with Visual Studio 2022 Desktop development with C++, Git, Python 3, CMake and Ninja. Run the following commands in a PowerShell session with the **x64 Visual Studio developer environment** loaded. Use a short path such as `C:\film-build` for the output. Start from a clean, private working copy of the compatible CoA source revision `142003f36998f6a19b8852084ee579c958dd1cfa`. Owning game data and having permission to use a source tree are separate requirements.

The helper inputs used for testing were:

| Input | Revision or version |
| --- | --- |
| AzerothPlex helper source | [9f51402fcd74da49bfc42f44d3c215fc91655f9d](https://github.com/VIXAL-OS/AzerothPlex/tree/9f51402fcd74da49bfc42f44d3c215fc91655f9d) |
| Video Cinema helper source | [a7a0958c6493b52ffb568b7bb8ee7e9dfc1d7e76](https://github.com/VIXAL-OS/wxl-video-screen/tree/a7a0958c6493b52ffb568b7bb8ee7e9dfc1d7e76) |
| libmpv x64 development package | `mpv-dev-x86_64-20261014-git-ec38361c56.7z`, [mpv Windows builds, 20261014](https://github.com/shinchiro/mpv-winbuild-cmake/releases/tag/20261014) |
| WebView2 SDK | Expanded [Microsoft.Web.WebView2 1.0.4129.50](https://www.nuget.org/packages/Microsoft.Web.WebView2/1.0.4129.50) |
| Video Cinema runtime | [Microsoft Edge WebView2 Evergreen](https://developer.microsoft.com/microsoft-edge/webview2/) |

## Apply the patch

Extract the patch release, set these paths to your local folders, and use an **empty** worker preparation directory. The revision and clean-file checks matter because the integration patch uses line positions on that exact revision.

```powershell
$client = 'C:\src\local-coa'
$filmPatch = 'C:\src\coa-film-club\film-club.patch'
$plex = 'C:\src\AzerothPlex'
$cinema = 'C:\src\wxl-video-screen'
$build = 'C:\film-build'
$mpv = 'C:\sdk\mpv-x64'        # Contains include\mpv and libmpv-2.dll
$webview = 'C:\sdk\webview2'   # Contains build\native\include and build\native\x64

if ((git -C $client rev-parse HEAD) -ne '142003f36998f6a19b8852084ee579c958dd1cfa') { throw 'Unexpected client revision' }
if (git -C $client status --porcelain) { throw 'Use a clean client checkout' }
if ((git -C $plex rev-parse HEAD) -ne '9f51402fcd74da49bfc42f44d3c215fc91655f9d') { throw 'Unexpected Plex revision' }
if ((git -C $cinema rev-parse HEAD) -ne 'a7a0958c6493b52ffb568b7bb8ee7e9dfc1d7e76') { throw 'Unexpected Cinema revision' }
git -C $client apply --check --index --unidiff-zero --whitespace=nowarn $filmPatch
if ($LASTEXITCODE) { throw 'Patch check failed' }
git -C $client apply --index --unidiff-zero --whitespace=nowarn $filmPatch
if ($LASTEXITCODE) { throw 'Patch application failed' }
python "$client\FilmClub\prepare-workers.py" $plex $cinema "$build\prepared"
if ($LASTEXITCODE) { throw 'Helper preparation failed' }
Copy-Item "$build\prepared\protocol\*Protocol.h" "$client\WoW\Source\Ui\Film"
```

## Build and lay out the local client

Configure after applying the patch so CMake discovers the new client sources.

```powershell
cmake -S $client -B "$build\client" -G Ninja -DCMAKE_BUILD_TYPE=Release
if ($LASTEXITCODE) { throw 'Client configuration failed' }
cmake --build "$build\client" --target WowClient --parallel 4
if ($LASTEXITCODE) { throw 'Client build failed' }

foreach ($mode in 'Plex','Cinema') {
    $dep = if ($mode -eq 'Plex') { "-DMPV_ROOT=$mpv" } else { "-DWEBVIEW2_ROOT=$webview" }
    cmake -S "$build\prepared\workers\$mode" -B "$build\$mode" -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_TRY_COMPILE_CONFIGURATION=Release $dep
    if ($LASTEXITCODE) { throw "$mode configuration failed" }
    cmake --build "$build\$mode" --target wxl-video-host --parallel 4
    if ($LASTEXITCODE) { throw "$mode build failed" }
}
```

The client build writes `WowClient64.exe` and `XvidDecoder.dll` to `bin\Win` under your local source tree. Put both in a separate local client directory. Place each helper in its own subfolder as shown below. Copy Cinema's `player.html` from `prepared\workers\Cinema` and libmpv from your x64 SDK. For CyTube media that uses yt-dlp, place the pinned `yt-dlp.exe` beside the Plex worker. Its [upstream notices](https://github.com/VIXAL-OS/AzerothPlex/blob/9f51402fcd74da49bfc42f44d3c215fc91655f9d/THIRD_PARTY_NOTICES.md) identify the release and expected hash.

```text
WowClient64.exe
XvidDecoder.dll
Media/
  Cinema/player.html
  Cinema/wxl-video-host.exe
  Plex/libmpv-2.dll
  Plex/wxl-video-host.exe
  Plex/yt-dlp.exe        (for supported external media)
```

The client requires matching Ascension data and a compatible realm. In the world, enter **/film**, select a player and click **Place screen**. Private D3D9 and D3D9Ex tests covered frame rendering, depth and a two-participant loopback party. Native CoA login, gameplay performance, real-account streaming and cross-PC playback timing remain unverified.

## Attribution

The helper changes adapt Azeroth Plex and Video Cinema at the revisions above. Their GPL terms and original notices apply to the helper-derived material. [GPL license text](COPYING) · [Azeroth Plex modifications](https://github.com/VIXAL-OS/AzerothPlex/blob/9f51402fcd74da49bfc42f44d3c215fc91655f9d/MODIFICATIONS.md) · [Video Cinema modifications](https://github.com/VIXAL-OS/wxl-video-screen/blob/a7a0958c6493b52ffb568b7bb8ee7e9dfc1d7e76/MODIFICATIONS.md).

