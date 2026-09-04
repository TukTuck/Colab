# FreeLLMAPI in Google Colab

Ein Colab-Notebook, das [FreeLLMAPI](https://github.com/tashfeenahmed/freellmapi) auf der
temporären Colab-VM startet und seine Daten **verschlüsselt in Google Drive** ablegt, damit
eine neue Colab-VM dieselbe Installation samt Provider-Keys wieder hochzieht.

**→ [`FreeLLMAPI_Colab.ipynb`](FreeLLMAPI_Colab.ipynb)** · in Colab öffnen, Zellen 1–8 der
Reihe nach ausführen.

---

## Einmalige Vorbereitung

Ein einziges Colab-Secret — linke Seitenleiste → 🔑 *Secrets* → *Add a new secret*:

| Feld | Wert |
|---|---|
| Name | `FREELLMAPI_ENCRYPTION_KEY` |
| Wert | 64 Hex-Zeichen aus `openssl rand -hex 32` |
| Notebook access | **ein** |

Optional: `FREELLMAPI_DB_BACKUP_KEY` (getrennter Backup-Schlüssel) und
`FREELLMAPI_UNIFIED_KEY` (damit der Abschlusstest `/v1/models` mit HTTP 200 prüft).

## Erster Start vs. späterer Start

| | Erster Start | Späterer Start |
|---|---|---|
| Zellen | 1–8 | 1–8 |
| Was passiert | Installation, neue Datenbank, Backup nach Drive | Zelle 1 meldet `Modus: SPÄTERER START`, FreeLLMAPI spielt das Backup beim Start automatisch ein |
| Danach | Dashboard → Account anlegen → **Keys** → Provider-Keys eintragen | Account, Provider-Keys und Fallback-Ketten sind wieder da |

Im Log steht der Restore als `[db-backup] restored 372736 bytes from …`; Zelle 5 wertet
diese Zeile aus.

## Wo was liegt

| Ort | Inhalt |
|---|---|
| `Google Drive/FreeLLMAPI/freeapi.db.backup` | Verschlüsselte SQLite (AES-256-GCM + gzip), von FreeLLMAPI geschrieben |
| `Google Drive/FreeLLMAPI/freeapi.db.backup.prev` | Rollback-Kopie, vom Notebook vor jedem Start angelegt |
| `Google Drive/FreeLLMAPI/app-cache-v0.9.5.tar.gz` | Nur Code + `node_modules` + Build, **keine** Daten, **keine** Schlüssel |
| `/content/freellmapi/data/freeapi.db` | Live-Datenbank, flüchtig (WAL braucht echten POSIX-Dateisystem-Lock) |
| `/content/freellmapi/logs/server.log` | Server-Log, flüchtig, bleibt bewusst **nicht** in Drive |

**Niemals im Klartext in Drive:** `ENCRYPTION_KEY` (lebt nur im Colab-Secret und im
Prozessspeicher), Provider-Keys (AES-256-GCM in der SQLite), eine `.env` (wird gar nicht
geschrieben), das Server-Log. Zelle 7 durchsucht Drive aktiv nach dem Schlüssel.

## Verwendete FreeLLMAPI-Schnittstellen

Alles aus dem Repository, Tag `v0.9.5` (Commit `aa28db9`), nichts erfunden:

| Zweck | Variable / Endpunkt | Quelle |
|---|---|---|
| Start ohne Docker | `npm run build` → `node server/dist/index.js` | `docs/install.md` |
| Port | `PORT=3001` | `docs/env/01-variables.md` |
| Bindung | `HOST=127.0.0.1` | `docs/env/01-variables.md` |
| DB-Pfad | `FREEAPI_DB_PATH` | `docs/env/01-variables.md` |
| Verschlüsseltes Backup | `FREEAPI_DB_BACKUP_PATH` | `docs/deployment/02-updates-and-backup.md` |
| Backup-Takt | `FREEAPI_DB_BACKUP_INTERVAL_MS` | `docs/env/01-variables.md` |
| Backup-Schlüssel | `FREEAPI_DB_BACKUP_KEY` (Default: `ENCRYPTION_KEY`) | `docs/env/01-variables.md` |
| Schlüssel | `ENCRYPTION_KEY`, 64 Hex | `docs/env/02-security-and-keys.md` |
| Verzeichnisrechte | `FREEAPI_DB_DIR_HARDENING=1` | `docs/env/01-variables.md` |
| `.env` abschalten | `FREEAPI_ENV_PATH` auf nicht vorhandene Datei | `docs/env/01-variables.md` |
| Health | `GET /api/ping` → `{"status":"ok",…}`, ohne Auth | `server/src/app.ts` |
| API-Spezifikation | `GET /v1/openapi.json`, ohne Auth | `docs/api/01-rest-api.md` |
| Modelle | `GET /v1/models`, mit Unified Key | `docs/api/01-rest-api.md` |

## Vier Dinge, die beim Bauen aufgefallen sind

1. **Docker geht in Colab nicht.** FreeLLMAPI empfiehlt Docker Compose, aber auf verwalteten
   Colab-Runtimes gibt es keinen Docker-Daemon, und die
   [Colab-FAQ](https://research.google.com/colaboratory/faq.html) untersagt
   Container-Techniken zur Umgehung der Anti-Missbrauchs-Regeln. Deshalb der offiziell
   dokumentierte Weg ohne Docker.
2. **`package-lock.json` enthält 40 Einträge auf `registry.npmmirror.com`.** Ist der Mirror
   nicht erreichbar, bricht `npm ci` mit `Exit handler never called!` ab. Zelle 3 schreibt
   den Host auf `registry.npmjs.org` um — identische Pfade und Integrity-Hashes.
3. **`better-sqlite3` ist nur eine optionale Abhängigkeit.** npm schluckt einen Fehlschlag
   also still, und der Server bricht später mit `better-sqlite3 is not installed` ab. Der
   `node:sqlite`-Fallback greift laut `server/src/db/index.ts` nur auf Android. Zelle 3
   installiert und verifiziert das Modul deshalb ausdrücklich.
4. **Ein Fehlstart kann Daten kosten.** FreeLLMAPI überspringt den Restore, sobald die
   Zieldatei existiert und > 0 Byte ist. Ein Abbruch *nach* dem Anlegen der DB hinterlässt
   aber eine leere 4-kB-Datei — der nächste Start käme mit leerer DB hoch und die
   Backup-Pumpe würde das gute Drive-Backup überschreiben. Dagegen: Schlüssel-Preflight in
   Zelle 4, Rollback-Kopie in Zelle 4, Aufräumen der halben DB in Zelle 5.

## Grenzen

- **Colab ist kein Hosting.** Die Colab-FAQ verbietet Web-Dienste „not related to interactive
  compute" und auf der kostenlosen Stufe „bypassing the notebook UI to interact primarily via
  a web UI". Kostenlose Notebooks laufen maximal 12 Stunden. Für Dauerbetrieb: eigener Host.
- **Datenverlustfenster** = `FREEAPI_DB_BACKUP_INTERVAL_MS` (hier 2 Minuten).
- **`ENCRYPTION_KEY` verloren = Daten verloren.** Es gibt laut
  `docs/env/02-security-and-keys.md` keinen Wiederherstellungsweg.
- **Kein öffentlicher Zugriff.** Version 1 bindet nur auf `127.0.0.1`, kein Tunnel.
