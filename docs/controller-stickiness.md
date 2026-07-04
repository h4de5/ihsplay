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

Siehe `docs/build-pipeline.md` für Build + Deploy.
