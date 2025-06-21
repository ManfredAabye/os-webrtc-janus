# os-webrtc-janus

Addon-Modul für [OpenSimulator], um WebRTC-Sprachunterstützung
mit dem Janus-Gateway bereitzustellen.

Eine Erklärung zu Hintergrund und Architektur
wurde auf der
[OpenSimulator Community Conference] 2024
im Vortrag
[WebRTC Voice for OpenSimulator](https://www.youtube.com/watch?v=nL78fieIFYg) präsentiert.

Dieses Addon funktioniert, indem es Viewer-Anfragen für Sprachdienste
entgegennimmt und einen separaten, externen [Janus-Gateway WebRTC-Server] nutzt.
Dieser kann so konfiguriert werden, dass er lokale Regionen-Spatial-Voice
sowie grid-weite Gruppen- und Spatial-Voice unterstützt. Siehe die Abschnitte unten.

Für den Betrieb dieses separaten Janus-Servers siehe
[os-webrtc-janus-docker], das Anweisungen für den Betrieb
des Janus-Gateways unter Linux und Windows WSL mit Docker enthält.

Anleitungen für:

- [Einbindung in OpenSimulator](#Building): Baue OpenSimulator mit WebRTC-Sprachdienst
- [Simulator für Sprachdienste konfigurieren](#Configure_Simulator)
- [Robust Grid Service konfigurieren](#Configure_Robust)
- [Standalone-Region konfigurieren](#Configure_Standalone)
- [Sprachdienst verwalten](#Managing_Voice) (Konsolenbefehle, etc.)

**Hinweis:** Stand Januar 2024 bietet diese Lösung keinen echten
räumlichen Sprachdienst (Spatial Voice) mit Janus. Es gibt Personen,
die an Erweiterungen für Janus arbeiten, um dies zu ermöglichen,
aber die bestehende Lösung bietet nur nicht-räumliche
Sprachdienste über das `AudioBridge`-Janus-Plugin. Außerdem sind
Funktionen wie Stummschalten und individuelle Avatar-Lautstärke noch nicht implementiert.

<a id="Known_Issues"></a>
## Bekannte Probleme

- Kein Spatial Audio
- Man sieht seinen eigenen „weißen Punkt“, aber nicht die anderer Avatare
- Kein Stummschalten
- Keine individuelle Lautstärkeregelung

Und wahrscheinlich mehr, siehe [os-webrtc-janus issues](https://github.com/Misterblue/os-webrtc-janus/issues).

<a id="Building"></a>
## Plugin in OpenSimulator einbinden

`os-webrtc-janus` wird als Quelltext-Build in [OpenSimulator] integriert.
Es nutzt die Addon-Modul-Funktion von [OpenSimulator], sodass der
Build so einfach ist wie das Klonen der `os-webrtc-janus`-Quellen in den
[OpenSimulator]-Quelltextbaum, das Ausführen des Build-Konfigurationsskripts
und dann das Bauen von OpenSimulator.

Die Schritte sind:

```
# OpenSimulator-Quellen holen
git clone git://opensimulator.org/git/opensim
cd opensim     # in das Top-Level-OpenSim-Verzeichnis wechseln

# WebRtc-Addon holen
cd addon-modules
git clone https://github.com/Misterblue/os-webrtc-janus.git
cd ..

# Projektdateien erzeugen
./runprebuild.sh

# OpenSimulator mit dem WebRTC-Addon bauen
./compile.sh

# Die INI-Datei für WebRTC in ein Konfigurationsverzeichnis kopieren, das beim Start gelesen wird
mkdir bin/config
cp addon-modules/os-webrtc-janus/os-webrtc-janus.ini bin/config
```

Diese Schritte erzeugen mehrere `.dll`-Dateien für `os-webrtc-janus`
im Verzeichnis `bin/WebRtc*.dll`. Einige Nutzer haben herausgefunden, dass man – statt die
[OpenSimulator]-Quellen zu bauen – die `.dll`-Dateien auch einfach in ein bestehendes
`/bin`-Verzeichnis kopieren kann. Stelle nur sicher, dass die `WebRtc*.dll`-Dateien
mit der gleichen Version von [OpenSimulator] gebaut wurden, die du verwendest.

<a id="Configure_Simulator"></a>
## Eine Region für Voice konfigurieren

Im letzten Schritt von [Building](#Building) wurde `os-webrtc-janus.ini` in das 
Verzeichnis `bin/config` kopiert. [OpenSimulator] liest alle `.ini`-Dateien
in diesem Verzeichnis, sodass diese Kopieroperation die Konfiguration für `os-webrtc-janus`
hinzufügt. Diese Konfiguration muss für Simulator und Region angepasst werden.

Die Beispiel-`.ini`-Datei hat zwei Abschnitte: `[WebRtcVoice]` und `[JanusWebRtcVoice]`.
Der Abschnitt `WebRtcVoice` konfiguriert, welche Dienste der Simulator für
WebRTC-Voice nutzt. Der Abschnitt `[JanusWebRtcVoice]` konfiguriert alle Verbindungen
vom Simulator zum Janus-Server. Dieser Abschnitt wird nur aktualisiert,
wenn der Simulator einen lokalen Janus-Server für Spatial Voice nutzt.

Die Werte für `SpatialVoiceService` und `NonSpatialVoiceService` zeigen
entweder direkt auf einen Janus-Dienst oder auf einen Robust-Grid-Server, der
den Grid-Sprachdienst bereitstellt. Beide Optionen sind in der Beispiel-`os-webrtc-janus.ini`
vorgesehen – die passende sollte aktiviert werden.

Der Viewer fordert entweder Spatial Voice (für Regionen und Parzellen)
oder Non-Spatial Voice (für Gruppen- oder Einzelgespräche) an.
`os-webrtc-janus` ermöglicht, dass diese beiden Arten von Voice-Verbindungen
durch verschiedene Voice-Dienste abgewickelt werden. Es gibt daher zwei unterschiedliche Konfigurationen:

- Alle Sprachdienste werden durch das Grid bereitgestellt (beide Einträge zeigen auf einen Robust-Dienst).
- Der Region-Simulator stellt einen lokalen Janus-Server für Spatial Voice bereit, während Grid-Dienste für Gruppenchats genutzt werden.

#### Nur Grid-Voice-Dienste

Die häufigste Konfiguration ist ein Simulator, der die vom Grid bereitgestellten
Sprachdienste nutzt. Dafür sieht die `os-webrtc-janus.ini` so aus:

```
[WebRtcVoice]
    Enabled = true
    SpatialVoiceService = WebRtcVoice.dll:WebRtcVoiceServiceConnector
    NonSpatialVoiceService = WebRtcVoice.dll:WebRtcVoiceServiceConnector
    WebRtcVoiceServerURI = ${Const|PrivURL}:${Const|PrivatePort}
```

Dies leitet sowohl Spatial- als auch Non-Spatial-Voice an den Grid-Service-Connector weiter
und `WebRtcVoiceServerURI` verweist auf den konfigurierten Robust-Grid-Service.

Ein `[JanusWebRtcVoice]`-Abschnitt ist nicht nötig, da alles durch die Grid-Services abgedeckt wird.

#### Lokaler Simulator-Janus-Dienst

In einer Grid-Umgebung kann es nötig sein, dass ein einzelner Simulator/eine Region
einen eigenen Janus-Server nutzt, z.B. aus Datenschutzgründen oder zur Entlastung der Grid-Sprachdienste.
In dieser Konfiguration wird Spatial Voice zum lokalen Janus-Dienst geleitet,
während Non-Spatial Voice weiterhin die Grid-Dienste nutzt – so bleiben Gruppenchats grid-weit möglich.

Die `os-webrtc-janus.ini` sieht dann so aus:
```
[WebRtcVoice]
    Enabled = true
    SpatialVoiceService = WebRtcJanusService.dll:WebRtcJanusService
    NonSpatialVoiceService = WebRtcVoice.dll:WebRtcVoiceServiceConnector
    WebRtcVoiceServerURI = ${Const|PrivURL}:${Const|PrivatePort}
[JanusWebRtcVoice]
    JanusGatewayURI = http://janus.example.org:14223/voice
    APIToken = APITokenToNeverCheckIn
    JanusGatewayAdminURI = http://janus.example.org/admin
    AdminAPIToken = AdminAPITokenToNeverCheckIn
```

Beachte: Da der Simulator einen eigenen Janus-Dienst nutzt, müssen die
Verbindungsparameter zu diesem Dienst konfiguriert werden. Details zum Betrieb
und zur Konfiguration eines Janus-Dienstes stehen bei [os-webrtc-janus-docker]. 
Hier müssen die URI zum Janus-Server und die API-Keys angegeben werden.
Das Beispiel enthält Platzhalter.

<a id="Configure_Robust"></a>
## Robust-Server für WebRTC-Voice konfigurieren

Auf Grid-Service-Seite wird `os-webrtc-janus` als zusätzlicher Dienst im Robust-OpenSimulator-Server konfiguriert.
Die Ergänzungen in `Robust.ini` sind:

```
...
[ServiceList]
    ...
    VoiceServiceConnector = "${Const|PrivatePort}/WebRtcVoice.dll:WebRtcVoiceServerConnector"
    ...

[WebRtcVoice]
    Enabled = true
    SpatialVoiceService = WebRtcJanusService.dll:WebRtcJanusService
    NonSpatialVoiceService = WebRtcJanusService.dll:WebRtcJanusService
[JanusWebRtcVoice]
    JanusGatewayURI = http://janus.example.org:14223/voice
    APIToken = APITokenToNeverCheckIn
    JanusGatewayAdminURI = http://janus.example.org/admin
    AdminAPIToken = AdminAPITokenToNeverCheckIn
...
```

Damit wird `VoiceServiceConnector` zu den angebotenen Diensten des Robust-Servers
hinzugefügt, und die WebRtcVoice-Konfiguration legt fest, dass sowohl Spatial- als auch Non-Spatial-Voice
über den Janus-Server laufen, inkl. entsprechender Janus-Konfiguration.

Es ist möglich, mehrere Robust-Services zur Lastverteilung zu konfigurieren.
Ein Robust-Server kann auch ausschließlich `VoiceServiceConnector` in seiner ServiceList haben.

<a id="Configure_Standalone"></a>
## Standalone-Region konfigurieren

[OpenSimulator] kann auch im „Standalone“-Modus betrieben werden, wobei alle Grid-Services und Regionen
in einer Simulator-Instanz laufen. Sprachdienste können hier nützlich sein
für private Meetings oder Tests. Dazu wird ein Janus-Server eingerichtet
und der Standalone-Simulator so konfiguriert, dass alle Voice-Dienste auf diesen Server zeigen:

```
[WebRtcVoice]
    Enabled = true
    SpatialVoiceService = WebRtcJanusService.dll:WebRtcJanusService
    NonSpatialVoiceService = WebRtcJanusService.dll:WebRtcJanusService
    WebRtcVoiceServerURI = ${Const|PrivURL}:${Const|PrivatePort}
[JanusWebRtcVoice]
    JanusGatewayURI = http://janus.example.org:14223/voice
    APIToken = APITokenToNeverCheckIn
    JanusGatewayAdminURI = http://janus.example.org/admin
    AdminAPIToken = AdminAPITokenToNeverCheckIn
```

Das leitet sowohl Spatial- als auch Non-Spatial-Voice an den Janus-Server weiter
und konfiguriert die URI und API-Zugänge für diesen Server.

<a id="Managing_Voice"></a>
## Sprachdienste verwalten (Konsolenbefehle)

Es gibt einige Konsolenbefehle, um das Sprachsystem zu prüfen und zu steuern.
Die aktuelle Liste der Befehle kann im Simulator mit dem Konsolenbefehl `help webrtc` angezeigt werden.

Dieser Abschnitt wird fortlaufend erweitert.

**webrtc list sessions** -- nicht implementiert

**janus info** -- Zeigt viele Details der Janus-Gateway-Konfiguration. Sehr unübersichtliches, nicht formatiertes JSON.

**janus list rooms** -- Listet die vom `AudioBridge`-Janus-Plugin angelegten Räume auf

[SecondLife WebRTC Voice]: https://wiki.secondlife.com/wiki/WebRTC_Voice  
[OpenSimulator]: http://opensimulator.org  
[OpenSimulator Community Conference]: https://conference.opensimulator.org  
[os-webrtc-janus]: https://github.com/Misterblue/os-webrtc-janus  
[Janus-Gateway WebRTC server]: https://janus.conf.meetecho.com/  
[os-webrtc-janus-docker]: https://github.com/Misterblue/os-webrtc-janus-docker  

---
