# Home Assistant Ansible Updater

Ein Ansible-Projekt zum automatisierten Aktualisieren von **Home Assistant**, **Home-Assistant-Apps/Add-ons** und **HACS-Komponenten**.

Das Playbook kann regelmäßig, zum Beispiel nachts per Cron, ausgeführt werden. Es prüft zunächst, ob Updates verfügbar sind, installiert nur tatsächlich vorhandene Updates und startet Home Assistant nur dann neu, wenn nach einem HACS-Update ein Neustart erforderlich ist.

## Funktionen

- Home Assistant Core auf Updates prüfen
- Home Assistant Core mit Backup aktualisieren
- Home Assistant Apps/Add-ons erkennen und einzeln aktualisieren
- Unterstützung für aktuelle Home-Assistant-CLI-Versionen ohne `ha apps update --all`
- HACS-Updates über die Home-Assistant-API erkennen
- HACS-Komponenten automatisch aktualisieren
- Nach HACS-Updates prüfen, ob ein Neustart erforderlich ist
- Home Assistant nur bei Bedarf neu starten
- Unterstützung für mehrere Home-Assistant-Instanzen
- Individuelle API-URLs und Tokens pro Instanz über `host_vars`
- Geeignet für automatisierte Ausführung per Cron

## Voraussetzungen

Auf dem Ansible-System:

- Linux
- Ansible
- SSH-Zugriff auf Home Assistant
- SSH-Key-Authentifizierung empfohlen

Auf Home Assistant:

- Home Assistant OS oder eine Installation mit verfügbarer `ha` CLI
- SSH-Zugriff, zum Beispiel über **Advanced SSH & Web Terminal**
- Long-Lived Access Token für API-Abfragen
- HACS, falls HACS-Komponenten aktualisiert werden sollen

## Projektstruktur

```text
homeassistant-ansible-updater/
├── hosts.ini
├── pb_update_homeassistant.yaml
├── host_vars/
│   ├── ha_instanz1.yml.example
│   └── ha_instanz2.yml.example
├── .gitignore
└── README.md
```

## Inventory

Beispiel für `hosts.ini`:

```ini
[homeassistant]
ha_instanz1 ansible_host=192.168.0.254
ha_instanz2 ansible_host=192.168.0.253

[all:vars]
ansible_user=root
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_shell_type=sh
```

Die Inventory-Namen `ha_instanz1` und `ha_instanz2` müssen zu den jeweiligen Dateien unter `host_vars/` passen.

## Host-spezifische Variablen

Für jede Home-Assistant-Instanz wird eine eigene Variablendatei verwendet.

### Instanz 1

Datei:

```text
host_vars/ha_instanz1.yml
```

Beispiel:

```yaml
homeassistant_api_url: "http://192.168.0.254"
homeassistant_api_token: "DEIN_LONG_LIVED_ACCESS_TOKEN"
```

### Instanz 2

Datei:

```text
host_vars/ha_instanz2.yml
```

Beispiel:

```yaml
homeassistant_api_url: "http://192.168.0.253"
homeassistant_api_token: "DEIN_LONG_LIVED_ACCESS_TOKEN"
```

Falls die Home-Assistant-API auf einem abweichenden Port erreichbar ist, muss dieser einfach in `homeassistant_api_url` ergänzt werden.

> **Wichtig:** Echte API-Tokens niemals in ein öffentliches GitHub-Repository committen.

Für ein öffentliches Repository empfiehlt es sich, nur Beispiel-Dateien wie `ha_instanz1.yml.example` und `ha_instanz2.yml.example` einzuchecken und die echten `.yml`-Dateien per `.gitignore` auszuschließen.

## API-Token erstellen

In Home Assistant im Benutzerprofil einen **Long-Lived Access Token** erstellen und pro Instanz in der passenden `host_vars`-Datei hinterlegen.

Zum Testen der API von Instanz 1:

```bash
curl -i \
  -H "Authorization: Bearer DEIN_TOKEN" \
  http://192.168.0.254/api/
```

Zum Testen der API von Instanz 2:

```bash
curl -i \
  -H "Authorization: Bearer DEIN_TOKEN" \
  http://192.168.0.253/api/
```

Bei erfolgreicher Authentifizierung sollte Home Assistant mit `200 OK` antworten.

## Playbook ausführen

Alle Home-Assistant-Instanzen aktualisieren:

```bash
ansible-playbook -i hosts.ini pb_update_homeassistant.yaml
```

Nur Instanz 1 aktualisieren:

```bash
ansible-playbook -i hosts.ini pb_update_homeassistant.yaml --limit ha_instanz1
```

Nur Instanz 2 aktualisieren:

```bash
ansible-playbook -i hosts.ini pb_update_homeassistant.yaml --limit ha_instanz2
```

## Update-Ablauf

Das Playbook arbeitet grundsätzlich in dieser Reihenfolge:

```text
Home Assistant Core prüfen
        ↓
Core bei Bedarf aktualisieren
        ↓
Home Assistant Apps/Add-ons prüfen
        ↓
Apps bei Bedarf aktualisieren
        ↓
HACS-Updates prüfen
        ↓
HACS-Komponenten aktualisieren
        ↓
Prüfen, ob nach den HACS-Updates ein Neustart erforderlich ist
        ↓
Home Assistant Core nur bei Bedarf neu starten
```

## Home Assistant Apps/Add-ons

Home Assistant Apps/Add-ons werden über die `ha` CLI abgefragt und einzeln aktualisiert.

Dadurch kann das Playbook mit CLI-Versionen umgehen, bei denen ein globaler Aufruf wie

```bash
ha apps update --all
```

nicht verfügbar ist.

Das Playbook prüft stattdessen die installierten Apps und aktualisiert nur die Komponenten, für die tatsächlich eine neue Version angeboten wird.

## HACS

HACS-Komponenten sind keine Home-Assistant-Apps/Add-ons und erscheinen daher nicht unter:

```bash
ha apps list
```

Das Playbook fragt HACS-Update-Entities stattdessen über die Home-Assistant-API ab und installiert verfügbare Updates über den `update.install`-Service.

Damit können unter anderem folgende HACS-Komponenten aktualisiert werden:

- Custom Integrations
- Lovelace Cards
- Frontend Plugins
- HACS selbst

Ein Neustart von Home Assistant wird nur durchgeführt, wenn:

1. in diesem Playbook-Lauf mindestens ein HACS-Update installiert wurde und
2. anschließend tatsächlich ein Home-Assistant-Neustart erforderlich ist.

Sind keine HACS-Updates vorhanden oder ist nach den Updates kein Neustart notwendig, wird Home Assistant nicht neu gestartet.

## Automatische Ausführung per Cron

Beispiel: Das Playbook jeden Tag um 03:00 Uhr ausführen.

Zuerst ein Log-Verzeichnis im gewünschten Projektpfad anlegen:

```bash
mkdir -p /path/to/homeassistant-ansible-updater/logs
```

Cron bearbeiten:

```bash
crontab -e
```

Beispiel-Eintrag:

```cron
0 3 * * * /usr/bin/flock -n /tmp/homeassistant-ansible-updater.lock /bin/bash -c 'cd /path/to/homeassistant-ansible-updater && /usr/bin/ansible-playbook -i hosts.ini pb_update_homeassistant.yaml >> logs/homeassistant-update.log 2>&1'
```

Der Pfad `/path/to/homeassistant-ansible-updater` muss an den tatsächlichen Installationsort angepasst werden.

Log anzeigen:

```bash
tail -f /path/to/homeassistant-ansible-updater/logs/homeassistant-update.log
```

## Sicherheit

Echte Zugangsdaten und Tokens sollten niemals unverschlüsselt in Git eingecheckt werden.

Empfohlene Vorgehensweisen:

- echte `host_vars/*.yml` über `.gitignore` ausschließen
- nur `.example`-Dateien veröffentlichen
- Ansible Vault für Secrets verwenden
- Vault-Passwörter niemals im Repository speichern
- für automatisierte Ausführung ein sicher abgelegtes Vault-Passwort verwenden

Beispiel für Ansible Vault:

```bash
ansible-vault encrypt host_vars/ha_instanz1.yml
ansible-vault encrypt host_vars/ha_instanz2.yml
```

Bei automatisierter Ausführung kann anschließend beispielsweise verwendet werden:

```bash
ansible-playbook \
  -i hosts.ini \
  pb_update_homeassistant.yaml \
  --vault-password-file /path/to/vault-password-file
```

## `.gitignore`

Empfohlener Inhalt:

```gitignore
# Secrets
host_vars/*.yml
*.vault
.vault_pass

# Beispiel-Dateien dürfen eingecheckt werden
!host_vars/*.yml.example

# Logs
logs/
*.log

# Ansible
*.retry

# Editor / OS
.DS_Store
.vscode/
.idea/
```

## Beispiel-Dateien für `host_vars`

`host_vars/ha_instanz1.yml.example`:

```yaml
homeassistant_api_url: "http://192.168.0.254"
homeassistant_api_token: "CHANGE_ME"
```

`host_vars/ha_instanz2.yml.example`:

```yaml
homeassistant_api_url: "http://192.168.0.253"
homeassistant_api_token: "CHANGE_ME"
```

Nach dem Klonen können die Dateien kopiert werden:

```bash
cp host_vars/ha_instanz1.yml.example host_vars/ha_instanz1.yml
cp host_vars/ha_instanz2.yml.example host_vars/ha_instanz2.yml
```

Anschließend werden in den `.yml`-Dateien die jeweiligen API-Tokens eingetragen.

## Lizenz

Für ein öffentliches Repository empfiehlt sich eine klare Open-Source-Lizenz, zum Beispiel die **MIT License**.

## Hinweis

Dieses Projekt ist kein offizielles Home-Assistant-Projekt. Automatisierte Updates sollten nur eingesetzt werden, wenn ein funktionierendes Backup-Konzept vorhanden ist.
