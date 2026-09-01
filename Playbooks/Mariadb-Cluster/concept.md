# MariaDB Galera Cluster

## Ziel

Es soll ein MariaDB-Galera-Cluster mit drei Datenbankknoten automatisiert
bereitgestellt und betrieben werden:

- `db01`
- `db02`
- `db03`

Alle drei Knoten sollen im normalen Betrieb als `Synced` im gleichen
`Primary`-Cluster arbeiten. Erfolgreich bestätigte Transaktionen werden von
Galera auf die verbundenen Clusterknoten repliziert.

Wichtig: Ein ausgefallener oder abgetrennter Knoten kann während seiner
Abwesenheit nicht aktuell sein. Er holt den fehlenden Stand beim Wiedereintritt
per IST oder SST nach. Das Ziel ist daher ein konsistenter Cluster ohne
dauerhafte Datenabweichung, nicht drei Knoten, die unter allen Fehlerbedingungen
gleichzeitig erreichbar sind.

## Kurzentscheidung: HAProxy und Keepalived

HAProxy und Keepalived lösen unterschiedliche Probleme. Für einen hochverfügbaren
Client-Endpunkt werden sie gemeinsam eingesetzt:

| Komponente | Aufgabe | Erforderlich? |
| --- | --- | --- |
| MariaDB Galera | Replikation, Cluster-Mitgliedschaft, Quorum und State Transfer | Ja |
| HAProxy | Verbindungen auf gesunde MariaDB-Knoten verteilen | Empfohlen |
| Keepalived | Eine virtuelle IP zwischen zwei HAProxy-Knoten verschieben | Für einen eigenen HA-Endpunkt empfohlen |

### Empfohlene Variante

```text
Anwendungen
     |
     v
db.example.internal:3306
     |
     v
VIP 10.10.20.10 (Keepalived/VRRP)
     |
     +-----------------------+
     |                       |
 HAProxy01              HAProxy02
     |                       |
     +-----------+-----------+
                 |
        +--------+--------+
        |        |        |
       db01     db02     db03
        \        |        /
         +-------+-------+
                 |
              Galera
```

- **HAProxy allein** ist möglich, wenn ein einzelner Load Balancer oder ein
  anderer externer Load-Balancer als Ausfallrisiko akzeptiert wird.
- **Keepalived allein** ersetzt HAProxy nicht. Eine VIP kann auf einen Host
  zeigen, entscheidet aber nicht zuverlässig, welcher Datenbankknoten den
  Galera-Zustand `Synced` und `Primary` hat.
- **HAProxy + Keepalived** beseitigt den Single Point of Failure auf der
  Zugriffsseite: HAProxy wählt den Backend-Knoten, Keepalived stellt den
  stabilen Endpunkt bereit.
- Beide Programme können auf denselben zwei Hosts laufen. Sie sind logisch
  getrennte Rollen und sollten in Ansible getrennt konfiguriert werden.
- Alternativ kann ein bereits vorhandener externer Load-Balancer die Aufgaben
  von HAProxy und Keepalived übernehmen.

Keepalived muss den Zustand von HAProxy und dessen Healthcheck überwachen. Eine
VIP darf nicht auf einem Host verbleiben, auf dem HAProxy läuft, aber keine
gesunden Datenbank-Backends mehr anbietet.

## Was Galera leistet

Galera übernimmt:

- Replikation von Transaktionen zwischen verbundenen Knoten
- Cluster-Mitgliedschaft und Quorum
- Erkennung von Knoten- und Verbindungsfehlern
- Incremental State Transfer (IST) und State Snapshot Transfer (SST)
- Multi-Primary-Unterstützung und Konflikterkennung

Galera übernimmt nicht:

- einen stabilen Client-Endpunkt
- Load Balancing oder Datenbank-Healthchecks für Clients
- Backups und Restore
- Schutz vor falschen SQL-Befehlen
- die Konfiguration von Anwendungen

### Quorum

Drei Knoten sind die sinnvolle Mindestgröße für diesen Aufbau. Beim Ausfall
eines Knotens bleiben zwei Knoten übrig und können weiterhin ein `Primary`-
Cluster bilden. Ein einzelner verbleibender Knoten besitzt normalerweise kein
Quorum und soll nicht automatisch erzwungen werden.

Bei einer Netzwerkpartition entscheidet das Quorum, welche Partition aktiv
bleibt. Ein automatisches Bootstrapping auf mehreren Knoten ist unbedingt zu
vermeiden.

## Zielarchitektur

### Datenbankknoten

| Host | Adresse | Funktion |
| --- | --- | --- |
| `db01` | über Inventory festgelegt | MariaDB + Galera |
| `db02` | über Inventory festgelegt | MariaDB + Galera |
| `db03` | über Inventory festgelegt | MariaDB + Galera |

Die drei Datenbankknoten sollten möglichst auf unterschiedlichen physischen
oder logischen Ausfall-Domänen betrieben werden, zum Beispiel auf
unterschiedlichen Proxmox-Hosts.

### Client-Zugriff

Anwendungen verbinden sich nicht direkt mit `db01`, `db02` oder `db03`, sondern
mit einem stabilen Namen beziehungsweise der VIP:

```text
db.example.internal:3306
```

DNS, VIP und konkrete Adressen sind Umgebungsvariablen und gehören nicht fest
in die Galera-Role.

## Verhalten des Datenbank-Endpunkts

Der HAProxy-Healthcheck darf sich nicht auf eine offene TCP-Verbindung zu Port
`3306` beschränken. Ein Knoten kann TCP-Verbindungen annehmen, obwohl er noch
nicht synchronisiert oder nicht Teil des aktiven Primary-Clusters ist.

Ein Backend gilt nur dann als geeignet, wenn mindestens folgende Werte erfüllt
sind:

```text
wsrep_connected            = ON
wsrep_ready                = ON
wsrep_local_state_comment  = Synced
wsrep_cluster_status       = Primary
```

Für die erste produktive Version sollte ein bevorzugter Schreibknoten oder ein
kontrolliertes Backend-Schema festgelegt werden. Galera unterstützt zwar
Multi-Primary, aber beliebig verteilte parallele Schreibzugriffe können
Zertifizierungsfehler und Konflikte erhöhen. Das ist eine Routing-Entscheidung
und keine Voraussetzung für die Replikation.

## Netzwerk und Firewall

| Port | Protokoll | Quelle/Ziel | Zweck |
| --- | --- | --- | --- |
| `3306` | TCP | erlaubte Clients oder HAProxy -> DB-Knoten | MariaDB |
| `4567` | TCP/UDP | DB-Knoten <-> DB-Knoten | Galera-Replikation |
| `4568` | TCP | DB-Knoten <-> DB-Knoten | IST |
| `4444` | TCP | DB-Knoten <-> DB-Knoten | SST |

Zusätzlich benötigt Keepalived zwischen den beiden Load-Balancer-Knoten VRRP
(IP-Protokoll `112`) oder die gewählte Unicast-Variante. Das muss im Netzwerk
und in der Firewall ausdrücklich erlaubt werden.

Regeln:

- Galera-Ports nur zwischen den drei Datenbankknoten freigeben.
- Port `3306` nur für HAProxy und tatsächlich berechtigte Clients freigeben.
- VRRP nur zwischen den beiden Keepalived-Knoten erlauben.
- Die Regeln in Ansible aus der Cluster-Gruppe und den definierten Client-Netzen
  ableiten.

## MariaDB- und Galera-Konfiguration

Die exakten Paketnamen und der Pfad zum Galera-Provider hängen von Distribution
und MariaDB-Version ab. Sie dürfen nicht ungeprüft aus einem Beispiel
übernommen werden.

Die Konfiguration benötigt unter anderem:

```ini
[mysqld]
binlog_format = ROW
default_storage_engine = InnoDB
innodb_autoinc_lock_mode = 2

wsrep_on = ON
wsrep_provider = <distributionsabhaengiger-provider-pfad>
wsrep_cluster_name = "mariadb_cluster"
wsrep_cluster_address = "gcomm://db01,db02,db03"
wsrep_node_name = "<eindeutiger-node-name>"
wsrep_node_address = "<erreichbare-node-adresse>"
wsrep_sst_method = mariabackup
wsrep_provider_options = "gcache.size=<groesse>"
```

Die Cluster-Adresse wird aus dem Ansible-Inventory erzeugt. Node-Name und
Node-Adresse sind pro Knoten eindeutig. Alle Knoten müssen eine kompatible
MariaDB- und Galera-Version verwenden.

GCache sollte so dimensioniert werden, dass kurze Ausfälle möglichst per IST
aufgeholt werden können. Bei längerer Abwesenheit oder zu kleiner GCache ist
weiterhin ein SST erforderlich.

## Ansible-Zielbild

### Inventory

```yaml
all:
  children:
    mariadb_cluster:
      hosts:
        db01:
          ansible_host: <db01-adresse>
        db02:
          ansible_host: <db02-adresse>
        db03:
          ansible_host: <db03-adresse>

    loadbalancer:
      hosts:
        lb01:
          ansible_host: <lb01-adresse>
        lb02:
          ansible_host: <lb02-adresse>
```

Die Gruppengröße muss vor der Installation geprüft werden. Der Bootstrap-Knoten
wird deterministisch aus der Cluster-Gruppe gewählt oder explizit vorgegeben.

### Wichtige Variablen

```yaml
mariadb_version: "<freigegebene-version>"
mariadb_port: 3306
mariadb_bind_address: "<interne-adresse-oder-0.0.0.0>"

galera_cluster_group: mariadb_cluster
galera_cluster_name: mariadb_cluster
galera_bootstrap_node: "{{ groups[galera_cluster_group][0] }}"
galera_sst_method: mariabackup
galera_gcache_size: "<an-schreiblast-angepasste-groesse>"
```

Die Zielversion muss vor dem produktiven Rollout anhand der offiziellen
Supportmatrix, des Betriebssystems und der Anwendungskompatibilität festgelegt
werden. Versionsnummern gehören in Variablen, nicht fest in Tasks oder
Templates.

### Rollen

Die Verantwortlichkeiten bleiben klein und getrennt:

```text
roles/
|-- mariadb_galera/
|-- haproxy/
`-- keepalived/
```

`mariadb_galera` übernimmt nur die Datenbankknoten. HAProxy und Keepalived
werden als eigene Rollen behandelt. Backups, Monitoring und Anwendungen sind
separate Betriebsbausteine.

## Installations- und Bootstrap-Ablauf

Der Ablauf muss zwischen erstmaligem Aufbau, normalem Lauf und vollständiger
Recovery unterscheiden.

1. Voraussetzungen, Betriebssystem, Versionen, Adressen und genau drei
   Clusterknoten prüfen.
2. Repository und MariaDB/Galera-Pakete auf allen Datenbankknoten installieren.
3. Firewall und Galera-Konfiguration auf allen Knoten vorbereiten.
4. Genau einen Bootstrap-Knoten bestimmen.
5. Nur auf diesem Knoten kontrolliert `galera_new_cluster` ausführen.
6. Prüfen, dass dieser Knoten `Primary`, `Synced`, `Connected` und `Ready` ist.
7. `db02` und danach `db03` normal mit `systemctl start mariadb` starten.
8. Nach jedem Beitritt den Zustand prüfen.
9. Erst nach erfolgreicher Clusterprüfung HAProxy und Keepalived aktivieren.

`galera_new_cluster` darf nicht bei jedem Ansible-Lauf ausgeführt werden. Ein
vollständiger Cluster-Ausfall benötigt einen bewusst gestarteten und
dokumentierten Recovery-Vorgang. Auf keinen Fall dürfen mehrere Knoten
gleichzeitig neu gebootstrapped werden.

Konfigurationsänderungen und Neustarts erfolgen im laufenden Betrieb seriell,
zum Beispiel mit `serial: 1`. Der Bootstrap-Ablauf ist ein eigener, kontrolliert
ausgeführter Schritt und kein gewöhnlicher Restart-Handler.

## Verifikation

Nach dem initialen Aufbau und nach jedem Recovery-Vorgang müssen mindestens
folgende Werte geprüft werden:

```sql
SHOW GLOBAL STATUS WHERE Variable_name IN (
  'wsrep_cluster_size',
  'wsrep_cluster_status',
  'wsrep_connected',
  'wsrep_ready',
  'wsrep_local_state_comment'
);
```

Erwarteter Zustand eines vollständig aufgebauten Clusters:

```text
wsrep_cluster_size          = 3
wsrep_cluster_status        = Primary
wsrep_connected             = ON
wsrep_ready                 = ON
wsrep_local_state_comment   = Synced
```

`wsrep_cluster_size = 2` ist während des Ausfalls oder der Wartung eines
Knotens erwartbar, aber kein erfolgreicher Endzustand der Erstinstallation.

## Sicherheit und Datenverwaltung

- Anwendungen verwenden eigene Datenbanken und Benutzer mit minimalen Rechten.
- Der MariaDB-Benutzer `root` wird nicht für Anwendungen verwendet.
- Passwörter gehören ausschließlich in Ansible Vault oder ein vergleichbares
  Secret-Management.
- Client-TLS und Galera-TLS werden abhängig von der Vertrauensgrenze und dem
  Netzwerkdesign geprüft.
- Galera ersetzt kein Backup. Backups müssen außerhalb des Clusters gespeichert
  und regelmäßig durch einen Restore-Test verifiziert werden.
- Ein replizierter `DROP`, ein fehlerhaftes `UPDATE` oder beschädigte Daten
  werden ebenfalls repliziert. Dagegen helfen nur geeignete Backups,
  Wiederherstellungsprozesse und Berechtigungen.

Konkrete Datenbanken, Benutzer und Anwendungen werden erst ergänzt, sobald die
tatsächlichen Verbraucher des Clusters feststehen. In dieses Konzept gehören
keine anwendungsspezifischen Beispiele.

## Betriebs- und Ausfalltests

Vor dem produktiven Einsatz müssen mindestens diese Szenarien getestet werden:

1. Eine Testdatenbank und eine Testtabelle auf `db01` anlegen und die Daten auf
   `db02` und `db03` lesen.
2. Einen Datenbankknoten stoppen. Die beiden übrigen Knoten müssen `Primary`
   bleiben und der Client-Endpunkt muss erreichbar bleiben.
3. Den Knoten wieder starten. Er muss per IST oder SST beitreten und wieder
   `Synced` werden.
4. Einen HAProxy-Knoten stoppen. Die VIP muss auf den zweiten Load Balancer
   wechseln.
5. Einen Galera-Healthcheck absichtlich fehlschlagen lassen. HAProxy darf den
   betroffenen Datenbankknoten nicht mehr als Backend verwenden.
6. Einen vollständigen Cluster-Ausfall und den manuellen Recovery-Ablauf in
   einer kontrollierten Umgebung testen.
7. Netzwerkpartitionen, Rolling Restarts, Transaktionskonflikte und lange
   Transaktionen testen.

## Bewusster Scope der ersten Version

Die erste Version automatisiert:

- Voraussetzungen und Versionsprüfung
- MariaDB-/Galera-Installation
- Firewall
- Inventory-basierte Cluster-Konfiguration
- kontrollierten Initial-Bootstrap
- serielles Starten und Wiederbeitreten der Knoten
- Galera-Healthcheck und Statusprüfung
- idempotente normale Ansible-Läufe

Nicht Bestandteil der ersten Galera-Role sind:

- anwendungsspezifische Konfiguration
- automatische DNS-Änderungen
- automatische Recovery nach beliebigen Fehlern
- erzwungenes Quorum oder automatisches Force-Bootstrap
- Backups und Restore
- Monitoring-Dashboard und Alerting
- HAProxy- und Keepalived-Installation, sofern sie als eigene Rollen gepflegt
  werden

## Offene Entscheidungen vor der Implementierung

Für die Umsetzung der Role sind noch diese Angaben nötig:

1. Welche Distribution und Version laufen auf `db01` bis `db03`?
2. Gibt es zwei dedizierte Hosts für HAProxy und Keepalived?
3. Welche internen IPs, Netzwerkkarte, VIP und DNS-Namen sollen verwendet
   werden?
4. Welche MariaDB-Version und welche Galera-/SST-Pakete sind freigegeben?
5. Soll die erste Version nur den Cluster erstellen oder auch Datenbanken und
   Benutzer aus Variablen anlegen?
6. Soll zunächst ein bevorzugter Schreibknoten verwendet werden oder sollen
   Schreibverbindungen auf mehrere gesunde Knoten verteilt werden?
7. Welche Backup- und Monitoring-Lösung ist außerhalb dieser Role vorgesehen?

## Sinnvolle Reihenfolge

```text
1. Zielsysteme und Netzwerk festlegen
2. Inventory und Ansible-Variablen definieren
3. MariaDB/Galera auf allen drei Knoten installieren
4. Firewall und Galera-Konfiguration ausrollen
5. db01 kontrolliert bootstrappen
6. db02 und db03 beitreten lassen
7. Clusterzustand und Replikation testen
8. HAProxy mit Galera-aware Healthcheck einrichten
9. Keepalived mit VIP und Service-Tracking einrichten
10. Ausfall-, Recovery- und Restore-Tests durchführen
```

## Referenzen

Konkrete Parameter und Paketnamen sind immer gegen die Dokumentation der
tatsächlich eingesetzten MariaDB-Version zu prüfen:

- MariaDB Galera Cluster: Architektur und Betrieb
- MariaDB Galera Cluster: Bootstrap und Recovery
- MariaDB WSREP-Status und Monitoring
- MariaDB State Transfer und SST-Methoden