# AGENTS.md - IHSplay

Übersicht über das Projekt `github-ihsplay`.

## Was ist IHSplay?

**IHSplay** ist ein SDL2-basierter Steam Link Client für **webOS TV** (primär) und **Raspberry Pi**. Es implementiert das Steam Remote Play Protokoll, um Spiele von einem Steam-Rechner auf TV-Geräte zu streamen.

- **Upstream:** https://github.com/mariotaku/ihsplay
- **Fork:** h4de5/ihsplay – mit Patches für verbesserte webOS-Kompatibilität
- **Sprache:** C (C11)
- **Build-System:** CMake (≥3.16)
- **Lizenz:** GPL v3

## Submodule

| Pfad | Repository | Beschreibung |
|---|---|---|
| `core` (≡ `ihslib`) | https://github.com/h4de5/ihslib.git (Branch: `master`) | Steam In-Home Streaming Library (h4de5-Fork, LGPL v3) |
| `third_party/lvgl` | https://github.com/lvgl/lvgl.git | LVGL Grafik-Bibliothek |
| `third_party/ss4s` | https://github.com/mariotaku/ss4s.git | SteamLink Simple Streaming System (AV-Output) |
| `third_party/commons` | https://github.com/mariotaku/commons-c.git | C-Utilities (Logging, Arrays, CEC, Luna-Sync) |
| `cmake/sanitizers` | https://github.com/arsenm/sanitizers-cmake.git | Address-Sanitizer CMake-Module |

Siehe `docs/` für Details:

| Datei | Beschreibung |
|---|---|
| `docs/build.md` | Build-Prozess (GitHub Actions CI, Download, Installation) |

## Projektstruktur

```
.
├── app/                        # Frontend (SDL2 + LVGL)
│   ├── backend/                # Stream & Input Backend
│   │   ├── host_manager.c/h    # Steam-Host-Discovery
│   │   ├── input_manager.c/h   # Input-Routing (Gamepad/Maus/Tastatur)
│   │   ├── stream_manager.h    # Stream-State-Manager (Interface)
│   │   └── stream/             # Streaming-Implementierung
│   │       ├── stream_manager.c/h  (internal)
│   │       ├── stream_input.c/h
│   │       └── stream_media.c/h
│   ├── lvgl/                   # LVGL-Customizing
│   │   ├── display.c/h, keypad.c/h, mouse.c/h, theme.c/h
│   │   ├── ext/                # LVGL-Erweiterungen
│   │   │   ├── lv_child_group.c/h, lv_dir_focus.c/h, msgbox_ext.c/h
│   │   └── fonts/              # Bootstrap-Icons-Font
│   ├── platform/               # Platform-spezifisch
│   │   ├── common/             # Plattformunabhängige Routinen
│   │   └── webos/              # webOS-spezifisch (Luna-Service-API)
│   ├── settings/               # App-Einstellungen
│   ├── ui/                     # UI-Screens (LVGL-Fragments)
│   │   ├── common/             # Shared (Fehler, Fortschritt)
│   │   ├── connection/         # Verbindungsdialog (PIN, Error)
│   │   ├── hosts/              # Host-Browser
│   │   ├── session/            # Session-UI + Streaming-Overlay
│   │   ├── settings/           # Einstellungsbildschirme
│   │   └── support/            # Support/Info (Version anzeigen)
│   └── util/                   # Hilfsfunktionen
│       ├── client_info.c/h, random.c/h, refcounter.h
│       ├── listeners_list.c/h
│       └── video/sps/          # H.264-SPS-Parser
├── cmake/                      # CMake-Module
│   ├── AresPackage.cmake       # webOS-Packaging
│   ├── PackageWebOS.cmake
│   ├── PackageDebian.cmake
│   └── sanitizers/             # Address-Sanitizer-CMake
├── core/                       # ≡ ihslib (Submodule)
│   ├── include/ihslib/         # Public API (audio, buffer, client, hid, etc.)
│   ├── src/                    # Library-Implementierung
│   │   ├── client/             # Discovery, Auth, Streaming
│   │   ├── session/            # Session-Management, Retransmission, Framing
│   │   ├── hid/                # HID-Device-Management + SDL-Backend
│   │   ├── crypto/             # Crypto (mbedtls oder openssl)
│   │   ├── platforms/          # Platform-Abstraktionen (UDP, Thread, IP)
│   │   └── protobuf/           # Protobuf-Definitionen (3 .proto-Dateien)
│   ├── samples/                # Nutzungsbeispiele
│   └── tests/                  # Unit-Tests
├── deploy/
│   ├── webos/                  # Appinfo, Icons
│   └── raspbian/               # sysroot-packages.list
├── tests/                      # IHSplay-Tests (CMakeLists.txt + app/)
├── third_party/                # Externe Abhängigkeiten
│   ├── lvgl/                   # LVGL (submodule)
│   ├── ss4s/                   # SS4S (submodule)
│   └── commons/                # commons-c (submodule)
├── tools/
│   ├── webos/                  # easy_build.sh + easy_install.sh
│   └── resource-tools/         # Font-Generierung (TypeScript/Gulp)
├── .github/workflows/
│   ├── build-test.yml          # CI: Build für jeden Push
│   └── release.yml             # CI: Release bei Tag-Erstellung
└── *.patch                     # 11 Patches (Fixes + CI-Anpassungen)
```

## Patches (eingespielt via `git am`)

Die `.patch`-Dateien im Root sind via `git am` eingespielt und im Commit-Verlauf sichtbar. Die `.patch`-Dateien dienen als Referenz/Backup.

Commit-Reihenfolge (älteste Patches zuerst):

| Commit | Patch | Beschreibung |
|---|---|---|
| `5576d47` | 0001 | Fix: Connection-Timeout, Cancel, Disconnect-Crash, Controller-Stall, Cursor-Lookup |
| `2c7a002` | 0002 | CI: NDK-Download – Filename dynamisch aus GitHub-API ermitteln |
| `825057c` | 0003 | CI: NDK-Auswahl – korrekten 32-bit arm-webOS-Tarball verwenden |
| `9cad152` | 0004 | CI: Disk-Space vor NDK-Extrakt freigeben, nach Workspace extrahieren |
| `9d40952` | 0005 | CI: NDK auf webos-b17b4cc pinnen |
| `9f1e6a0` | 0006 | CI: überflüssigen free-disk-space-Schritt entfernen |
| `9564b48` | 0007 | CI: Raspi-Job entfernen, Actions auf Node.js 24-kompatibel updaten |
| `4bbdea7` | 0008 | CI: Release-Workflow – NDK-Tag pinnen, Action-Versionen updaten |
| `d43b913` | 0009 | Fix: Cancel während CONNECTING-State, HID-Reset bei Disconnect |
| `847de72` | 0010 | Update ihslib auf 26ade25 |
| `cf63f24` | 0011 | **Revert** "chore: update ihslib submodule to latest (26ade25)" |
| `73d12aa` | 0013 | Feat: Version im Support-Screen, ihslib → h4de5-Fork |

Zusätzlicher Commit (nicht aus Patch):
- `1815a79` – "Add personal contribution note to README" (manuell, von Andi/h4de5)

**Wichtig:** Patch 0013 ändert `.gitmodules` – core zeigt auf `h4de5/ihslib.git` (Branch: `main`).

## Eigenständiges `github-ihslib`-Repository

**Pfad:** `/workspace/development/github/github-ihslib/`
**Remote:** `https://github.com/h4de5/ihslib` (Branch: `master`)
**Lizenz:** **LGPL v3** (im Unterschied zu IHSplay GPL v3)

Enthält einen zusätzlichen Patch (`0012_ihslib-fix-disable-retransmission-for-HID-input-messages.patch`), der **nicht** im IHSplay-Root liegt:

- Disable Retransmission für HID-Input-Messages – Controller-Input ist zeitkritisch; Retransmission von veraltetem Input-State blockiert den Queue für 20-30 Sekunden.

Dieses Repo ist ein **eigenständiges Klon** der ihslib-Bibliothek, getrennt vom Submodule `core` in ihsplay. Enthält ebenfalls eigene `.github/workflows/build-test.yml`.

## Architektur

```
IHSplay (SDL2 App)
  ├── LVGL (UI-Rendering) ← benutzt SDL2
  ├── IHSlib (Streaming-Protokoll)
  │   ├── Discovery (UDP Broadcast)
  │   ├── Authorization (RSA/AES)
  │   └── Session (UDP-Stream + HID-Rückkanal)
  ├── SS4S (AV-Output)
  │   ├── webOS: LGNC, NDL (webOS4/5), Starfish Media Pipeline
  │   └── Linux: SDL, ALSA, PulseAudio, MMAL
  └── Commons-c (Hilfsbibliotheken)
```

## Build

**Gebaut wird nur über GitHub Actions CI** – der lokale Workspace hat keine NDK-Toolchain.

### Pipeline (bei jedem Push)

1. `actions/checkout` mit Submodules
2. NDK + Tools downloaden (ares-cli-rs, webosbrew-toolbox)
3. `./tools/webos/easy_build.sh`
4. `.ipk` als Artifact (`webos-snapshot`) hochladen
5. Optional: `ares-install` auf den TV

### CI/CD Workflows

| Workflow | Trigger | Output |
|---|---|---|
| `build-test.yml` | Jeder Push (außer Tags) | `.ipk` als Artifact |
| `release.yml` | GitHub-Release erstellt | `.ipk` ans Release gehängt |

Beide verwenden:
- `webosbrew/ares-cli-rs` → `ares-package` für IPK-Packaging
- `webosbrew/dev-toolbox-cli` → `webosbrew-toolbox` für Manifest-Generierung/Verifikation
- NDK von `openlgtv/buildroot-nc4` Tag `webos-b17b4cc`

Siehe `docs/build.md` für Details.

## Git-Wrapper

Im Projekt-Root liegt `git-wrapper` – ein Bash-Skript, das `git commit|pull|push|merge` 1:1 an echtes Git durchreicht.

**Nutzungsregel (MUSS):**
- Der git-wrapper darf **nur** verwendet werden, nachdem der konkrete Git-Befehl (mit allen Parametern) dem Benutzer via `question`-Tool angezeigt wurde und dieser die Ausführung explizit bestätigt hat.
- Der git-wrapper darf **nie** implizit, aus Versehen, aus Gewohnheit oder ohne vorherige Frage ausgeführt werden.
- Jeder Aufruf zählt als destruktive Git-Operation und unterliegt denselben Regeln wie direkte `git commit/push/merge/pull`-Aufrufe.

**Unterstützte Befehle:** `add`, `rm`, `commit`, `push`, `merge`, `pull`
