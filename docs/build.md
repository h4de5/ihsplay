# Build

**Wichtig:** Gebaut wird **ausschließlich über GitHub Actions CI**. Lokal gibt es keine NDK-Toolchain.

## Pipeline

- **build-test.yml** – Läuft bei **jedem Push** auf jeden Branch
- **release.yml** – Läuft bei **GitHub-Release-Erstellung**

Beide bauen ein `.ipk` für webOS (arm) und hängen es als Artifact an.

## Download

Nach erfolgreichem CI-Build:

1. https://github.com/h4de5/ihsplay/actions – Build auswählen
2. Artifact `webos-snapshot` herunterladen (`.ipk`)

## Install auf TV

```bash
ares-install --device <TV-IP> pfad/zu/app.ipk
```

## Abhängigkeiten (CI)

Wird automatisch in der Pipeline geladen:
- webOS NDK (`openlgtv/buildroot-nc4`, Tag `webos-b17b4cc`)
- `ares-cli-rs` (Packaging)
- `webosbrew-toolbox` (Manifest-Verifikation)
