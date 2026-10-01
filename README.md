FHEM-HM485
==========

Module for FHEM for Homematic-Wired devices

You can find more information in the file FHEM/lib/HM485/readme.txt

Fork chrisse1/FHEM-HM485
------------------------

Dieser Fork basiert auf [kc-GitHub/FHEM-HM485](https://github.com/kc-GitHub/FHEM-HM485)
(Stand `master`, Commit 40a88b8 vom 06.12.2023, Modulversion 0.8.16) und enthält genau
zwei Änderungen an `FHEM/10_HM485.pm` (Version **0.8.17**):

1. **Broadcast-/Gruppenadressen** (z.B. `FF000001`, `FFFFFF01`) unterdrücken nicht mehr
   die Aktualisierung von `state`/`working`/`level` des eigenen Kanals. Bisher landeten
   solche Meldungen nur in `P-`-Readings (`... to unknown_FF000001`), und FHEM zeigte
   veraltete Zustände.
2. **Neues Attribut `stateInterval`**: zyklisches Nachlesen des Kanalzustands, wie
   `get <kanal> state`. Vorgabe: aus. Ohne gesetztes Attribut verhält sich das Modul wie
   bisher.

Alle anderen Dateien sind unverändert gegenüber Upstream.

### Installation

In FHEM zuerst die bisherige HM485-Update-Quelle entfernen (beide Quellen benutzen
dieselbe Kontrolldatei `controls_hm485.txt` und würden sich gegenseitig überschreiben).
Die genaue URL zeigt `update list`, üblicherweise:

    update delete https://raw.githubusercontent.com/kc-GitHub/FHEM-HM485/master/controls_hm485.txt

Dann den Fork eintragen und aktualisieren:

    update add https://raw.githubusercontent.com/chrisse1/FHEM-HM485/master/controls_hm485.txt
    update check
    update

Danach FHEM **richtig neu starten** (`shutdown restart` bzw. Dienst neu starten).
Ein `reload 10_HM485` reicht nicht: Das Modul registriert beim Laden neue Funktionen
(`AttrFn`) und Prototypen, das wird nur bei einem echten Neustart sauber übernommen.

Kontrolle nach dem Neustart: `{ $modules{HM485}{AttrFn} }` liefert `HM485_Attr`, und
in der Attributliste eines HM485-Geräts taucht `stateInterval` auf.

### stateInterval verwenden

Wert in Sekunden, `0` oder nicht gesetzt = aus, Minimum 10.

- **Am Gerät** gilt das Attribut für alle Kanäle des Geräts, die laut Gerätebeschreibung
  ihren Zustand melden können (Schalt-, Rollladen-, Dimmerkanäle, nicht aber Taster).
  Beim HMW_IO_12_Sw7_DR sind das genau die sieben Schaltkanäle 13–19.
- **Am Kanal** gilt es nur für diesen Kanal und hat Vorrang vor dem Gerät.
  `attr <kanal> stateInterval 0` nimmt einen einzelnen Kanal heraus.
- Alle Abfragen aller HM485-Geräte laufen über einen gemeinsamen Verteiler: höchstens
  eine Abfrage alle 2 Sekunden, die Startzeitpunkte zufällig über das Intervall gestreut.
- Es wird nichts gesendet, solange das IO-Gerät nicht `opened` ist, solange die
  Gerätekonfiguration gelesen wird (`configStatus` ≠ `OK`) oder wenn Kanal oder Gerät
  `disable`, `dummy` oder `ignore` gesetzt haben.

Ersatz für die bisherige Krücke (`HM485Aktoren`/`HM485StatusPruefen` in
`99_myUtils.pm` und DOIF `HM485_Statuswacht`), Beispiel:

    # auffälliges Gerät (NEQ1810465, hmwId 000179F3): jede Minute alle Schaltkanäle
    attr <geraet-000179F3> stateInterval 60
    # alle anderen Aktorgeräte: alle 5 Minuten
    attr <anderes-geraet> stateInterval 300
    save

Danach können der DOIF `HM485_Statuswacht` und die beiden Funktionen in `99_myUtils.pm`
gelöscht werden. Ob die Abfragen laufen, zeigt `attr <geraet> verbose 5`
(Logzeilen `stateInterval: query state`).

### Rückbau

1. Attribut entfernen, sonst meldet das Original-Modul beim Start
   „Unknown attribute stateInterval“:

       deleteattr TYPE=HM485 stateInterval
       save

2. Update-Quelle zurückstellen und die Upstream-Dateien erzwingen (die Upstream-Datei ist
   älter als die installierte, ein normales `update` würde sie nicht zurückspielen):

       update delete https://raw.githubusercontent.com/chrisse1/FHEM-HM485/master/controls_hm485.txt
       update add https://raw.githubusercontent.com/kc-GitHub/FHEM-HM485/master/controls_hm485.txt
       update force

3. FHEM neu starten (`shutdown restart`).

### Hinweise

- Für Busmitschnitte `attr <hm485-io> HM485d_logVerbose 3` verwenden (schreibt `Tx`/`Rx`
  ins normale FHEM-Log). `HM485d_logfile` lässt den Daemon in den Hintergrund forken
  (`ServerTools.pm`), FHEM verliert dann die Kontrolle über ihn und es laufen zwei Instanzen.

### Kontrolldatei neu erzeugen (für Maintainer)

`controls_hm485.txt` wird mit `fhemupdate.pl` aus Dateizeit und Dateigröße erzeugt. Git
speichert keine Dateizeiten, deshalb vor dem Erzeugen die Zeiten setzen: unveränderte
Dateien auf ihren bisherigen Eintrag, geänderte auf den neuen Zeitpunkt (Zeitzone
Europe/Berlin, wie die bisherigen Einträge):

    grep ^UPD controls_hm485.txt | while read _ ts sz f; do
      TZ=Europe/Berlin touch -d "${ts/_/ }" "$f"; done
    TZ=Europe/Berlin touch -d "2026-10-01 22:10:00" FHEM/10_HM485.pm   # geänderte Datei(en)
    TZ=Europe/Berlin perl fhemupdate.pl

Prüfen, dass jede Größe stimmt:

    grep ^UPD controls_hm485.txt | while read _ ts sz f; do
      [ "$(stat -c %s "$f")" = "$sz" ] || echo "FALSCH: $f"; done
