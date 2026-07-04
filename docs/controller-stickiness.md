# Controller Input Stickiness / Freeze

## Problembeschreibung (wörtlich aus Usersicht)

Alle paar Minuten bleiben die Controller-Eingaben hängen. Es werden im Steam-Client (auf dem Streaming-Server) keine weiteren Eingaben mehr angenommen. Die **letzte Eingabe bleibt einfach stecken**: Drückt man z.B. nach links, macht das Spiel weiter, als ob man dauernd nach links drücken würde.

Musik und Video des Spiels laufen **ungehindert weiter** – nur die Controller-Eingaben bleiben hängen.

## Betroffene Umgebung

- **Streaming-Client:** IHSplay auf webOS TV (LG Fernseher)
- **Streaming-Server:** Headless PC (kein Bildschirm) – darauf kann nicht lokal gespielt werden; nur Remote-Zugriff über Streaming
- **Controller:** Beliebig (Gamepad via Bluetooth oder USB am TV)

## Nicht betroffen

- **Moonlight/Sunshine** auf demselben TV – mit der selben App (Sunshine) auf demselben Fernseher tritt das Problem **nicht** auf
- **Remote Play per Laptop** – mit demselben Controller und demselben Streaming-Server tritt das Problem **nicht** auf
- **Controller-Hardware** – der Controller ist nicht defekt (funktioniert in anderen Szenarien einwandfrei)

## Symptome im Detail

1. **Häufigkeit:** Tritt alle paar Minuten auf
2. **Eingabe klemmt:** Die letzte Aktion wiederholt sich permanent (z.B. Charakter läuft endlos nach links)
3. **Stream läuft:** Audio und Video des Spiels laufen normal weiter – kein Ruckeln, kein Aussetzer
4. **Controller-Verbindung intakt:** Langes Drücken der **Select-Taste** beendet den Streaming-Modus zuverlässig – man verlässt Steam und kommt zurück ins IHSplay-Launcher-Menü auf webOS. Das beweist:
   - Die Bluetooth/USB-Verbindung vom Controller zum TV ist aktiv
   - Die webOS-App (IHSplay) reagiert noch auf Controller-Eingaben
   - Die Session zu Steam läuft (sonst könnte man nicht per Select zurück)
5. **Verbindung zum Server besteht:** Die Netzwerkverbindung zwischen TV und Streaming-Server ist nicht abgerissen (Video/Audio laufen ja weiter)

## Workaround

1. **20-30 Sekunden warten** – dann löst sich der Block meist von selbst. In der Zwischenzeit verliert man aber oft das Spiel.
2. **Oder:** Lange Select-Taste drücken → aus dem Streaming aussteigen → zurück im IHSplay-Launcher → sofort neu verbinden → Problem ist für eine Weile behoben.

## Schlussfolgerung

Da das Problem nur mit IHSplay (nicht mit Moonlight/Sunshine oder Laptop Remote Play) auftritt, liegt die Ursache in der IHSplay-Applikation oder der zugrundeliegenden `ihslib`-Bibliothek. Hardware und Netzwerk sind ausgeschlossen.

## Bisherige Patches (alle wirkungslos)

### `ihslib` 8bf075f – "disable retransmission for HID input messages"
Deaktiviert die Retransmission für `k_EStreamControlRemoteHID`-Nachrichten im Control-Channel.
```c
bool retransmit = (type != k_EStreamControlRemoteHID);
```
**Ergebnis:** Problem besteht weiterhin.

### `ihsplay` 5576d47 – "fix: connection timeout, cancel, disconnect crash, controller stall, cursor lookup"
IHSplay-seitige Fixes für Verbindungs-Handling.
**Ergebnis:** Keine Besserung.

### `ihsplay` d43b913 – "fix: cancel works during CONNECTING state, reset HID on any disconnect"
Setzt HID-Device beim Disconnect zurück.
**Ergebnis:** Keine Besserung.

## Analyse des Sendepfads (Stand Juli 2026)

Der Patch 8bf075f setzt `retransmit=false` für HID-Nachrichten in `IHS_SessionChannelControlSend()` (ch_control.c:118).

Der Sendepfad ist dann:
1. `IHS_SessionChannelControlSend()` → `IHS_SessionChannelQueueFrame()` → `IHS_SessionQueuePacket()` → QueuedPacket.retransmit=false
2. `SessionSendWorker()` (session.c:263) pollt die Queue, sendet per UDP
3. Weil `retransmit=false` wird `IHS_RetransmissionQueue()` NICHT aufgerufen
4. Bei Paketverlust (UDP) wird die HID-Input-Nachricht **stumm verworfen**

### Root Cause

Der Session-UDP-Socket wird **blocking** erstellt (`socket()` ohne `O_NONBLOCK`, `sendto()` mit flags=0). Wenn der Kernel-UDP-Sendebuffer voll ist (typisch bei WLAN-Engpässen auf webOS TVs mit kleinen Standard-Puffern), blockiert `sendto()` den gesamten `SessionSendWorker`-Thread für Sekundenbruchteile bis Sekunden.

Der HID-Poll-Timer feuert alle **8 ms** (125 Hz) und queue-t neue HID-Reports in denselben Shared-Queue, den der Send-Worker abarbeitet. Wenn der Send-Worker blockiert (>8ms), wächst der Queue unbegrenzt.

**Warum 20-30 Sekunden:** Der blockierte `sendto()`-Aufruf muss warten, bis der Kernel Daten loswerden kann (WiFi TX ring, ACK-Timeout). Sobald Platz ist, entlädt sich der gestaute Queue schlagartig. Bis der nächste Stau kommt, dauert es wieder ein paar Minuten.

### Fix (Commit <TBD>)

Zwei unabhängige Maßnahmen, die sich ergänzen:

1. **`SO_SNDTIMEO` (10ms)** auf dem Session-UDP-Socket in `IHS_UDPSocketOpen()` (`ihs_udp_posix.c:60-61`). `sendto()` blockt nie länger als 10ms. Bei Timeout wird das Paket verworfen (EAGAIN) – der nächste Poll sendet frischen State.

2. **HID-Poll-Rate reduziert** von 8ms (125 Hz) auf **33ms (~30 Hz)** (`HID_POLL_INTERVAL_MS` in `manager.c:35`). Reduziert die Queue-Last um Faktor 4 und damit die Wahrscheinlichkeit von Buffer-Exhaustion drastisch.

**Warum 30 Hz ausreichen:** Die meisten Spiele laufen mit ≤60 fps und erfassen Eingaben mit ≤60 Hz. 30 Hz HID-Update-Rate liegt über der menschlichen Wahrnehmungsschwelle für Eingabeverzögerung und entspricht dem Polling-Intervall vieler Bluetooth-Controller.

**Ergebnis:** Problem besteht weiterhin unverändert (User-Feedback nach Deploy).

### Fix-Versuch 2: `SO_SNDBUF` auf 512KB erhöht (Commit `5ef33b2`)

Hypothese: Kernel-Sendbuffer (default ~208KB) läuft bei WiFi-Congestion voll, `sendto()` läuft in den `SO_SNDTIMEO` und droppt alle Pakete (Video, Audio, HID) still, bis der Buffer wieder Platz hat.

**Ergebnis:** Problem besteht weiterhin unverändert. Das spricht **gegen** die Theorie "Kernel-Sendbuffer voll" als alleinige Ursache.

## Neue Hypothese: Server deaktiviert Input-Streaming (unbestätigt)

`ch_control.c` implementiert `OnSetClientConfig()`, das auf die Server-Nachricht `CSetStreamingClientConfig` reagiert (mirrored von Steams `BStreamingInput`/`BStreamingAudio`/`BStreamingVideo`). Wenn der Server `enable_input_streaming=false` sendet, setzt der Client `session->state.streamingInput = false`. Das ist die **einzige** Stelle im Code, die dieses Flag außerhalb der Initialisierung (`true` bei Session-Start) verändert – rein Server-getrieben, kein Client-Bug.

Wirkung, wenn `streamingInput == false`:
- `IHS_HIDManager`s Poll-Tick pollt Devices weiterhin (State bleibt aktuell), aber `IHS_SessionHIDSendReport()` returned sofort ohne zu senden (`control_hid.c:214`)
- `IHS_SessionChannelControlSendHIDMsg()` verwirft die Nachricht komplett, noch bevor sie gepackt wird (`control_hid.c:140`)
- Maus/Tastatur (`control_input_mouse.c`, `control_input_kbd.c`, `control_input_touch.c`) sind **genauso** betroffen

**Warum das zum Symptom passt:**
- Kompletter Stillstand aller Eingaben (nicht nur Controller) für die Dauer der Deaktivierung
- "Letzter Input bleibt hängen": Wenn z.B. der Stick nach links gehalten wird und der Server in diesem Moment Input deaktiviert, kommt die spätere "losgelassen"-Meldung nie beim Server an → Server-seitiger virtueller Controller bleibt im letzten übermittelten Zustand hängen, bis Input reaktiviert wird und der Client den aktuellen Zustand nachliefert
- Erklärt "erholt sich von selbst" – ein serverseitiger Zustand/Timer
- Erklärt auch, warum beide bisherigen Netzwerk-Fixes (SO_SNDBUF, Poll-Rate) wirkungslos blieben – das Problem liegt oberhalb der UDP-Transportebene, auf Protokoll-/Anwendungsebene

**Unbestätigt, weil:**
- Kein Log-Beweis, dass dies tatsächlich passiert
- Unklar, warum der Server das für genau 20-30s täte (müsste per Ghidra-RE des Steam-Binaries geklärt werden, siehe `core/CLAUDE.md`)
- Könnte auch etwas völlig anderes sein

### Diagnose-Maßnahme (Commit `<TBD>`, kein Fix – nur Messung)

Da Live-Logs vom TV nicht praktikabel abrufbar sind (kein einfaches `ares-log` für native Apps, SSH-Zugriff nicht dokumentiert/verifiziert), wird die Diagnose **direkt in der App** sichtbar gemacht:

- `ch_control.c`: Neue Zähler `g_inputDisableCount`, `g_lastInputDisableDurationMs`, `g_inputDisabledSinceMs` (Prozess-lebenszeit-statics, überleben Session-Ende)
- Bei jedem `streamingInput`-Wechsel true→false wird die Startzeit gemerkt und der Zähler erhöht; bei false→true wird die Dauer berechnet
- Neue öffentliche API `IHS_SessionGetInputStreamingDiagnostics()` in `include/ihslib/session.h`
- Support-Screen (`app/ui/support/support.c`) zeigt jetzt: `Input disabled by server: Nx (last: Xms)` [+ `[CURRENTLY DISABLED]` falls gerade aktiv]

**Test-Anleitung für den User:**
1. Build installieren, streamen bis der Freeze auftritt
2. Entweder 20-30s warten bis es sich von selbst löst, oder per langem Select-Druck aussteigen
3. Zurück im IHSplay-Launcher → Support-Screen öffnen
4. Prüfen ob `Input disabled by server: 1x` (oder mehr) angezeigt wird
   - **Wenn ja:** Hypothese bestätigt, Server deaktiviert Input – nächster Schritt ist Ghidra-RE warum
   - **Wenn `0x`:** Hypothese widerlegt, Ursache liegt woanders (z.B. lokale Eingabeverarbeitung, Verschlüsselung/Sequenznummern, etc.)

Siehe `docs/build-pipeline.md` für Build + Deploy.
