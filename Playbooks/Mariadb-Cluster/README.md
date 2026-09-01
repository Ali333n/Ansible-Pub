# MariaDB Galera Starter

Dieses Verzeichnis enthält ein ausführbares Startgerüst für:

- drei MariaDB-/Galera-Knoten auf Debian 13
- zwei HAProxy-Knoten
- Keepalived auf beiden HAProxy-Knoten
- eine gemeinsame virtuelle IP
- einen Galera-aware Healthcheck
- optionale Datenbanken und Benutzer

Die Beispieladressen und `eth0` sind Platzhalter. Vor dem ersten Lauf müssen
Inventory, VIP, DNS und Netzwerke an die Umgebung angepasst werden.

## Architektur

Beide Load Balancer führen **HAProxy und Keepalived** aus. Keepalived verschiebt
die VIP zwischen `lb01` und `lb02`. HAProxy verwendet die VIP als Client-
Endpunkt und verbindet sich zunächst bevorzugt mit `db01`; `db02` und `db03`
sind Failover-Backends. Ein Backend wird nur verwendet, wenn sein lokaler
Healthcheck `Primary`, `Synced`, `Connected` und `Ready` meldet.

Der Client-Endpunkt ist der Name oder die VIP, den Anwendungen als
Datenbankadresse verwenden, zum Beispiel `db.example.internal:3306`. Die
Anwendungen kennen dadurch keine einzelnen Galera-Knoten.

## Vorbereitung

1. `inventory/hosts.yml` und die Dateien unter `inventory/group_vars/`
   anpassen.
2. Eine Vault-Datei anlegen:

   ```bash
   ansible-vault create inventory/group_vars/vault.yml
   ```

   Mindestens diese Werte eintragen:

   ```yaml
   vault_galera_sst_password: "ein-langes-zufaelliges-passwort"
   vault_keepalived_auth_password: "max8chr"
   ```

3. Für optionale Datenbanken und Benutzer `mariadb_databases` und
   `mariadb_users` in den Group Vars definieren. Benutzerpasswörter müssen aus
   Vault-Variablen kommen.
4. Prüfen, dass SSH-Zugriff mit `become` auf allen fünf Hosts funktioniert.

Die Collection `ansible.mysql` ist in der Repository-Datei
`../../requirements.yml` ergänzt:

```bash
ansible-galaxy collection install -r ../../requirements.yml
```

## Ausführung

Aus diesem Verzeichnis:

```bash
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml -e galera_bootstrap_enabled=true --ask-vault-pass
```

Der erste Lauf muss den Bootstrap-Schalter explizit erhalten. Danach den
Schalter nicht dauerhaft in den Group Vars setzen und normale Läufe ausführen:

```bash
ansible-playbook site.yml --ask-vault-pass
```

`galera_new_cluster` wird nur beim ausdrücklich markierten Initial-Bootstrap
auf dem festgelegten Bootstrap-Knoten ausgeführt. Die Rolle legt dafür einen
Marker im MariaDB-Datenverzeichnis an. Ein vollständiger Cluster-Ausfall ist
ein eigener, bewusst zu planender Recovery-Vorgang.

## Bewusste Grenzen

Die Rollen konfigurieren keine Backups, kein Monitoring-Dashboard, kein DNS und
keine Anwendungen. Die Firewall-Verwaltung ist standardmäßig deaktiviert, weil
vor dem Aktivieren mindestens SSH-Port und Client-Netze geprüft werden müssen.

MariaDB 11.8 ist für den Start als konservative Debian-13-Basis gesetzt. Eine
Umstellung auf MariaDB 12.3 sollte erst erfolgen, wenn Repository, Galera,
`mariabackup`-SST und die realen Anwendungen in einer Testumgebung geprüft
wurden.
