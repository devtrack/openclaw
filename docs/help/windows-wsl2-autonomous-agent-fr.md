---
summary: "Guide V2 sécurisé pour déployer OpenClaw sur Windows 11 + WSL2 en mode WhatsApp-first avec un seul runtime et un staging contrôlé"
read_when:
  - Vous voulez piloter OpenClaw uniquement via WhatsApp
  - Vous voulez une architecture autonome mais sous contrôle admin strict
  - Vous voulez une politique READ libre / WRITE-ACTION contrôlée
  - Vous voulez durcir runtime, staging et audit sur une home box
title: "OpenClaw V2 sécurisé sur Windows + WSL2 (WhatsApp-first)"
---

# OpenClaw V2 sécurisé sur Windows 11 + WSL2 (WhatsApp-first)

Ce guide installe **une seule instance OpenClaw** (service `systemd --user`) sur WSL2, avec une politique simple:

- **READ = libre**: l’IA peut observer, lire, scanner, interroger et télécharger pour analyse.
- **WRITE/ACTION = gouverné**: toute action qui modifie l’état du système, déclenche une action externe, ou exfiltre des données hors périmètre autorisé doit obtenir une approbation admin WhatsApp.

Rôles:

- **Admin (vous)** = décideur final.
- **IA** = opérateur de lecture/analyse autonome + exécution d’actions uniquement après approbation explicite.

---

## 1) Principe central : un seul OpenClaw runtime + staging opéré par l’IA

```text
                        WhatsApp (canal unique)
                                 |
               +-----------------+-----------------+
               |                                   |
       user commands                       admin commands
   (lecture / questions)         (/approve, /reject, CONFIRM, "ok")
               |                                   |
               +-----------------+-----------------+
                                 |
                         OpenClaw Runtime
                 (UNIQUE service systemd actif 24/7)
                                 |
                 +---------------+----------------+
                 |                                |
     READ autorisé sans friction         WRITE/ACTION sous approbation
   (lecture locale, scan LAN, HTTPS,     (modifs système, exposition,
    RTSP, APIs lecture, download analyse)  actions mutantes/exfiltration)
                 |
      +----------+----------------------------------------------+
      |                                                         |
 staging (dossiers/scripts, pas un service OpenClaw)     quarantine/approved
      |                                                         |
 fetch -> scan -> rapport -> demande approbation -> promote ----+
                     (admin WhatsApp décide)
```

### Arborescence standard

```bash
mkdir -p ~/assistant/{runtime,staging,quarantine,approved,logs,workspaces,core}
mkdir -p ~/assistant/staging/{reports,proposals,tmp}
mkdir -p ~/.openclaw
chmod 700 ~/assistant ~/.openclaw
chmod 700 ~/assistant/{runtime,staging,quarantine,approved,logs,workspaces,core}
chmod 700 ~/assistant/staging/{reports,proposals,tmp}
```

> **Important**: `staging` est un espace opéré par l’IA (scripts, analyses, benchmarks), **pas un second OpenClaw**. Un seul service runtime tourne.

---

## 2) Installation Windows + WSL2 (base)

### 2.1 Côté Windows (PowerShell admin)

```powershell
wsl --install -d Ubuntu
wsl --set-default-version 2
wsl --update
```

Redémarrer Windows puis lancer Ubuntu.

### 2.2 Activer systemd dans WSL2

```bash
sudo tee /etc/wsl.conf >/dev/null <<'EOWSL'
[boot]
systemd=true
EOWSL
```

Depuis PowerShell:

```powershell
wsl --shutdown
```

Relancer Ubuntu et vérifier:

```bash
ps -p 1 -o comm=
systemctl is-system-running || true
```

### 2.3 Paquets requis

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl wget ca-certificates jq ripgrep git logrotate coreutils findutils cron util-linux iputils-ping net-tools
```

> `nmap` n’est pas obligatoire. Si vous l’avez déjà, utilisez-le. Sinon, utilisez `ping`, `arp -an`, `/proc/net/arp`, ou `ip neigh`.

### 2.4 Node 22 + OpenClaw

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g openclaw@latest
node -v
openclaw --version
```

---

## 3) Configuration OpenClaw (WhatsApp-only)

1. Login:

```bash
openclaw login
```

2. Config de base:

```bash
openclaw config set gateway.mode local
openclaw config set gateway.bind loopback
openclaw config set gateway.port 18789
openclaw config set routing.defaultChannel whatsapp
```

3. Principes sécurité:

- Canal unique: `whatsapp`.
- Allowlist stricte des numéros autorisés (admin + numéro opérateur si distinct).
- Double confirmation pour actions sensibles:
  - `/approve <id>`
  - `CONFIRM <id>`
- **Raccourci explicite accepté**: si l’admin répond clairement `ok` à une question explicite (ex: “est-ce que je peux installer X ?”), cela vaut approbation pour cette action précise.
- Journaliser toute validation/rejet dans `~/assistant/logs/audit.log`.

4. Pairing + statut WhatsApp:

```bash
openclaw channels login --channel whatsapp
openclaw channels status --probe
```

---

## 4) Politique Read vs Write (section centrale)

Le runtime OpenClaw a le droit d’accéder au réseau (LAN + Internet) pour la **lecture/analyse**. Le contrôle se fait sur la nature de l’action (READ vs WRITE/ACTION), pas sur “couper le réseau”.

### 4.1 READ (autorisé sans validation)

- Lecture de fichiers locaux autorisés.
- Scan réseau local (ARP, ping sweep, `nmap` si déjà présent).
- Accès RTSP local en lecture (caméras).
- Requêtes sortantes HTTP(S) pour lecture (docs, APIs read-only, veille).
- Interrogation Alexa/APIs cloud en mode lecture.
- Téléchargement de code en quarantaine pour analyse.
- Stockage local de secrets/tokens dans un répertoire protégé (permissions strictes).

### 4.2 WRITE/ACTION (validation admin obligatoire)

- Ouvrir un port entrant / exposer un service.
- Modifier configuration système, systemd, firewall, DNS.
- Installer/désinstaller des logiciels Windows ou WSL.
- Modifier `run_safe.sh`, policy, allowlists, scripts critiques.
- Appels API mutantes (create/update/delete, déclencheurs externes).
- Déclencher une action physique (domotique, commande vocale, routine).
- Exfiltrer des données (images/vidéo/logs) vers une destination non approuvée.
- Envoyer fichiers/contenus hors des deux numéros WhatsApp autorisés.

### 4.3 Mécanisme d’approbation WhatsApp

- L’IA doit formuler une demande claire: action, impact, périmètre, rollback.
- L’admin répond via WhatsApp:
  - `/approve <id>` puis `CONFIRM <id>`, ou
  - réponse explicite `ok` à la question d’autorisation ciblée.
- Sans approbation explicite, l’action est refusée.
- Chaque demande, approbation, rejet et exécution est inscrite dans `~/assistant/logs/audit.log`.

---

## 5) Contrôle d’exfiltration (données sortantes)

Le runtime peut lire Internet/LAN, mais l’envoi de données sortantes est gouverné.

### 5.1 Niveau 1 (recommandé, logique/applicatif)

Gérer une allowlist de destinations sortantes via configuration/policy (`~/assistant/runtime/egress_allowlist.txt` par exemple):

- Endpoints nécessaires au bridge WhatsApp.
- `api.amazon.com` + endpoints Alexa nécessaires (si intégration).
- Domaines de lecture courants: GitHub, Google, OpenAI, ChatGPT, Gmail, etc. (adapter à vos besoins réels).
- LAN privé (`192.168.0.0/16`, `10.0.0.0/8`, `172.16.0.0/12`) pour RTSP/IoT.

Règles:

- Toute destination non listée déclenche une demande d’approbation admin avant ajout.
- L’IA peut proposer/modifier cette allowlist **avec accord admin**.
- Toute tentative vers destination non autorisée est bloquée + journalisée (audit).

### 5.2 Niveau 2 (optionnel, OS/firewall)

Option de durcissement via iptables/nftables (ou équivalent) pour forcer l’egress allowlist au niveau OS.

- **Optionnel** (pas obligatoire).
- Sous WSL2, comportement réseau/firewall parfois non trivial selon version/hôte Windows.
- À activer uniquement si vous maîtrisez bien le plan de rollback.

---

## 6) Safe Runner / Command Executor (allowlist = 1 exécutable)

Le runtime ne doit exécuter **qu’un seul script**: `~/assistant/core/run_safe.sh`.

### 6.1 Script complet `~/assistant/core/run_safe.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

# run_safe.sh
# Exécuteur contrôlé pour OpenClaw runtime.
# - Refus par défaut.
# - Bloque opérateurs shell dangereux.
# - Autorise seulement une liste réduite de commandes/sous-commandes.
# - Limite les chemins accessibles.
# - Timeout court.
# - Audit append-only best-effort.

BASE="$HOME/assistant"
AUDIT_LOG="$BASE/logs/audit.log"

mkdir -p "$BASE/logs"
touch "$AUDIT_LOG"
chmod 600 "$AUDIT_LOG"

# Best-effort append-only. Sous WSL2, chattr peut être indisponible.
if command -v chattr >/dev/null 2>&1; then
  chattr +a "$AUDIT_LOG" 2>/dev/null || true
fi

usage() {
  echo "Usage: $0 <cmd> [arg1 ...]"
  exit 2
}

log_event() {
  # format: date|pid|status|message
  printf '%s|pid=%s|%s|%s\n' "$(date -Is)" "$$" "$1" "$2" >> "$AUDIT_LOG"
}

is_allowed_path() {
  local p
  p="$(realpath -m "$1")"
  case "$p" in
    "$BASE/runtime"/*|"$BASE/workspaces"/*|"$BASE/logs"/*|"$BASE/approved"/*)
      return 0
      ;;
    *)
      return 1
      ;;
  esac
}

contains_forbidden_tokens() {
  local joined="$*"
  # Interdits stricts: |, >, >>, ;, &&, ||, $(), backticks
  [[ "$joined" == *"|"* ]] && return 0
  [[ "$joined" == *">>"* ]] && return 0
  [[ "$joined" == *">"* ]] && return 0
  [[ "$joined" == *";"* ]] && return 0
  [[ "$joined" == *"&&"* ]] && return 0
  [[ "$joined" == *"||"* ]] && return 0
  [[ "$joined" == *'$('* ]] && return 0
  [[ "$joined" == *'`'* ]] && return 0
  return 1
}

[[ $# -ge 1 ]] || usage

if contains_forbidden_tokens "$@"; then
  log_event "DENY" "forbidden token in: $*"
  echo "DENY: forbidden shell operator"
  exit 126
fi

cmd="$1"
shift || true

# Timeout par défaut: 20s (dans plage 10–30s)
TIMEOUT_BIN="timeout"
TIMEOUT_SEC="20s"

# Commandes autorisées (lecture / diagnostic)
case "$cmd" in
  git)
    sub="${1:-}"
    case "$sub" in
      status|diff)
        ;;
      *)
        log_event "DENY" "git subcommand forbidden: ${sub:-<none>}"
        echo "DENY: only git status|diff"
        exit 126
        ;;
    esac
    ;;
  rg|jq|cat|head|tail|sed|awk|ls|pwd|ping|ip|arp)
    ;;
  # Tests autorisés uniquement dans ~/assistant/workspaces/
  pytest|ctest|dotnet)
    if [[ "$cmd" == "dotnet" && "${1:-}" != "test" ]]; then
      log_event "DENY" "dotnet subcommand forbidden"
      echo "DENY: only dotnet test"
      exit 126
    fi
    ;;
  *)
    log_event "DENY" "command forbidden: $cmd"
    echo "DENY: command not allowed"
    exit 126
    ;;
esac

# Validation stricte des arguments ressemblant à des chemins
for a in "$@"; do
  case "$a" in
    /*|~/*|./*|../*|*/*)
      if ! is_allowed_path "$a"; then
        log_event "DENY" "path forbidden: $a"
        echo "DENY: path outside approved roots"
        exit 126
      fi
      ;;
  esac
done

# Pour les tests, forcer exécution dans workspaces
if [[ "$cmd" == "pytest" || "$cmd" == "ctest" || "$cmd" == "dotnet" ]]; then
  cwd="$(pwd)"
  if ! is_allowed_path "$cwd" || [[ "$(realpath -m "$cwd")" != "$BASE/workspaces"* ]]; then
    log_event "DENY" "tests allowed only in $BASE/workspaces"
    echo "DENY: tests only allowed inside ~/assistant/workspaces"
    exit 126
  fi
fi

log_event "ALLOW" "$cmd $*"
exec "$TIMEOUT_BIN" "$TIMEOUT_SEC" "$cmd" "$@"
```

Activation:

```bash
chmod 700 ~/assistant/core/run_safe.sh
```

### 6.2 Règle OpenClaw

Le composant d’exécution doit pointer **uniquement** vers:

```text
/home/<user>/assistant/core/run_safe.sh
```

Aucune autre commande directe autorisée.

---

## 7) Quarantine -> Scan -> Promote (sans service parallèle)

### 7.1 Règle de politique

- **Autorisé sans validation**: téléchargement en quarantaine si usage strictement lecture/analyse.
- **Validation admin obligatoire**: promotion vers `approved` (activation runtime).
- Si l’intégration nécessite ouverture de port, modification runner/policy, ou installation Windows/WSL: classer en WRITE/ACTION (approbation + intervention admin possible clavier/souris).

### 7.2 Script `~/assistant/core/fetch_to_quarantine.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

# Usage:
#   fetch_to_quarantine.sh <source_url_or_git> <name>

SRC="${1:?source required}"
NAME="${2:?name required}"
BASE="$HOME/assistant"
QROOT="$BASE/quarantine"
STAMP="$(date +%Y%m%d-%H%M%S)"
DEST="$QROOT/${NAME}-${STAMP}"

mkdir -p "$QROOT" "$BASE/logs"
chmod 700 "$QROOT"

if [[ "$SRC" =~ ^https?:// ]]; then
  mkdir -p "$DEST"
  FILE="$DEST/source.bin"
  curl -fL --retry 3 --connect-timeout 10 -o "$FILE" "$SRC"
elif [[ "$SRC" =~ ^git@|^https://.*\.git$ ]]; then
  git clone --depth 1 "$SRC" "$DEST"
else
  echo "Unsupported source: use http(s) URL or git URL"
  exit 2
fi

find "$DEST" -type f -print0 | sort -z | xargs -0 sha256sum > "$DEST/SHA256SUMS"

echo "source=$SRC" > "$DEST/FETCH.meta"
echo "fetched_at=$(date -Is)" >> "$DEST/FETCH.meta"
echo "dest=$DEST" >> "$DEST/FETCH.meta"

echo "OK: $DEST"
```

### 7.3 Script `~/assistant/core/scan_quarantine.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

# Usage:
#   scan_quarantine.sh <quarantine_dir>

TARGET="${1:?quarantine dir required}"
BASE="$HOME/assistant"
REPORT_DIR="$BASE/staging/reports"
mkdir -p "$REPORT_DIR"
REPORT="$REPORT_DIR/scan-$(basename "$TARGET")-$(date +%Y%m%d-%H%M%S).md"

[[ -d "$TARGET" ]] || { echo "Missing dir: $TARGET"; exit 2; }

{
  echo "# Rapport scan quarantine"
  echo
  echo "- Cible: $TARGET"
  echo "- Date: $(date -Is)"
  echo
  echo "## Résumé"
  echo "Scan statique minimal (heuristique) avant toute promotion."
  echo

  echo "## Signaux NPM"
  rg -n '"(postinstall|preinstall|install|prepare)"\s*:' "$TARGET" -g 'package.json' || true
  rg -n '"bin"\s*:' "$TARGET" -g 'package.json' || true
  rg -n 'curl|wget|powershell|Invoke-WebRequest|Invoke-Expression|child_process' "$TARGET" -g '*.js' -g '*.ts' || true
  echo

  echo "## Signaux Python"
  rg -n 'entry_points|scripts|cmdclass|setup\(' "$TARGET" -g 'setup.py' -g 'pyproject.toml' || true
  rg -n '\.whl$|\.so$|\.dll$|\.exe$' "$TARGET" || true
  rg -n 'subprocess|os\.system|requests\.get|urllib\.request|curl|wget' "$TARGET" -g '*.py' || true
  echo

  echo "## Fichiers potentiellement sensibles"
  find "$TARGET" -type f \( -name '*.sh' -o -name '*.ps1' -o -name '*.bat' -o -name '*.exe' -o -name '*.dll' \) | sort || true
  echo

  echo "## Dépendances et permissions (manuel à compléter)"
  echo "- Dépendances détectées: compléter via lecture package.json/pyproject.toml"
  echo "- Permissions requises: FS, réseau, process"
  echo "- Risques: exécution post-install, téléchargement dynamique, binaire opaque"
  echo

  echo "## Verdict"
  echo "- Statut proposé: PENDING_ADMIN_REVIEW"
  echo "- Action: /review <package> puis /showdiff <proposal>"
} > "$REPORT"

chmod 600 "$REPORT"
echo "OK: report=$REPORT"
```

### 7.4 Script `~/assistant/core/promote_approved.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

# Usage:
#   promote_approved.sh <quarantine_dir> <proposal_id>

SRC="${1:?quarantine dir required}"
PID="${2:?proposal id required}"
BASE="$HOME/assistant"
APPROVED="$BASE/approved"
AUDIT="$BASE/logs/audit.log"
MANIFEST_DIR="$APPROVED/manifests"
DEST="$APPROVED/$PID"

[[ -d "$SRC" ]] || { echo "Missing source dir"; exit 2; }
mkdir -p "$APPROVED" "$MANIFEST_DIR" "$BASE/logs"
chmod 700 "$APPROVED" "$MANIFEST_DIR" "$BASE/logs"
touch "$AUDIT"
chmod 600 "$AUDIT"

if [[ -e "$DEST" ]]; then
  echo "Destination exists: $DEST"
  exit 3
fi

cp -a "$SRC" "$DEST"
chmod -R go-rwx "$DEST"

MANIFEST="$MANIFEST_DIR/$PID.manifest"
{
  echo "proposal_id=$PID"
  echo "promoted_at=$(date -Is)"
  echo "source_dir=$SRC"
  echo "approved_dir=$DEST"
  echo "sha256_manifest_start"
  find "$DEST" -type f -print0 | sort -z | xargs -0 sha256sum
  echo "sha256_manifest_end"
} > "$MANIFEST"
chmod 600 "$MANIFEST"

printf '%s|PROMOTE|id=%s|src=%s|dest=%s\n' "$(date -Is)" "$PID" "$SRC" "$DEST" >> "$AUDIT"

echo "OK: promoted to $DEST"
```

Activation:

```bash
chmod 700 ~/assistant/core/{fetch_to_quarantine.sh,scan_quarantine.sh,promote_approved.sh}
```

---

## 8) Workflow d’auto-amélioration contrôlée

1. **Detect**: l’IA identifie un outillage utile.
2. **Fetch en quarantine (READ)**: autorisé sans validation.
3. **Scan + rapport (READ)**: résumé, dépendances, risques, impacts, diff proposé.
4. **Demande admin WhatsApp** pour toute promotion/modification runtime.
5. **Admin décide**:
   - `/approve <id>` puis `CONFIRM <id>`, ou réponse explicite `ok` si question ciblée,
   - ou `/reject <id>`.
6. **Si approuvé**: `promote_approved.sh <dir> <id>` puis application contrôlée.
7. **Audit complet** dans `~/assistant/logs/audit.log`.

### Commandes WhatsApp (convention opératoire)

- `/review <package>` -> lance analyse staging + rapport.
- `/showdiff <proposal>` -> montre diff de proposition.
- `/approve <id>` puis `CONFIRM <id>` -> promotion/apply autorisés.
- `/reject <id>` -> blocage + archivage.

---

## 9) Service systemd user runtime durci

Créer `~/.config/systemd/user/openclaw-gateway.service`:

```ini
[Unit]
Description=OpenClaw Gateway (runtime unique verrouillé)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/env openclaw gateway run --bind loopback --port 18789 --force
Restart=always
RestartSec=5
Environment=HOME=%h
Environment=OPENCLAW_STATE_DIR=%h/.openclaw
WorkingDirectory=%h/assistant/runtime

# Logs
StandardOutput=append:%h/assistant/logs/gateway.log
StandardError=append:%h/assistant/logs/gateway.err.log

# Hardening demandé
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=%h/assistant %h/.openclaw
UMask=0077
RestrictSUIDSGID=true
LockPersonality=true
MemoryDenyWriteExecute=true

[Install]
WantedBy=default.target
```

Activation:

```bash
mkdir -p ~/.config/systemd/user
systemctl --user daemon-reload
systemctl --user enable --now openclaw-gateway.service
systemctl --user status openclaw-gateway.service --no-pager
sudo loginctl enable-linger "$USER"
```

### Compatibilité WSL2

#### Profil minimal (si durcissement casse le runtime)

- Garder: `NoNewPrivileges=true`, `PrivateTmp=true`, `UMask=0077`, `RestrictSUIDSGID=true`, `LockPersonality=true`.
- Retirer temporairement: `MemoryDenyWriteExecute=true` et/ou `ProtectSystem=strict`.

#### Profil renforcé (si compatible)

- Activer tout le bloc ci-dessus.
- Valider avec:

```bash
systemd-analyze security --user openclaw-gateway.service || true
systemctl --user restart openclaw-gateway.service
systemctl --user status openclaw-gateway.service --no-pager
```

---

## 10) Logs, audit append-only et rotation

### 10.1 Audit append-only

```bash
touch ~/assistant/logs/audit.log
chmod 600 ~/assistant/logs/audit.log
if command -v chattr >/dev/null 2>&1; then
  chattr +a ~/assistant/logs/audit.log 2>/dev/null || true
fi
```

> Sous WSL2, `chattr +a` peut être non supporté selon FS/driver. Dans ce cas, garder permissions strictes + audit via scripts only.

### 10.2 Rotation hebdo + compression

Créer `~/.config/logrotate/openclaw.conf`:

```conf
/home/<user>/assistant/logs/*.log {
    weekly
    rotate 8
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    create 0600 <user> <user>
}
```

Test manuel:

```bash
logrotate -d ~/.config/logrotate/openclaw.conf
logrotate -f ~/.config/logrotate/openclaw.conf
```

---

## 11) Budgets tokens (stricts)

Valeurs par défaut recommandées:

- `150000` tokens / jour.
- `30000` tokens / tâche.
- hard stop à **80%** sans override admin.

Mode “session analyse temporaire”:

1. Admin autorise explicitement une hausse ponctuelle.
2. L’IA journalise l’override (raison, durée).
3. Retour automatique au profil strict à la fin de la session.

---

## 12) Daily research sans spam

Script `~/assistant/core/research_daily.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

BASE="$HOME/assistant"
LOCK="$BASE/staging/tmp/research.lock"
STATE_DIR="$BASE/staging/tmp"
OUT="$BASE/staging/reports/research-signals.txt"
HASH_FILE="$STATE_DIR/research.last.hash"

mkdir -p "$BASE/staging/reports" "$STATE_DIR" "$BASE/logs"

exec 9>"$LOCK"
flock -n 9 || exit 0

rg -n "TODO|FIXME|security|token|latency|cost" "$BASE/workspaces" > "$OUT" || true
COUNT="$(wc -l < "$OUT" | tr -d ' ')"
NEWHASH="$(sha256sum "$OUT" | awk '{print $1}')"
OLDHASH="$(cat "$HASH_FILE" 2>/dev/null || true)"

# Notifier seulement si changement + au moins 3 signaux
if [[ "$NEWHASH" != "$OLDHASH" && "$COUNT" -ge 3 ]]; then
  printf '%s|RESEARCH|signals=%s|hash=%s\n' "$(date -Is)" "$COUNT" "$NEWHASH" >> "$BASE/logs/audit.log"
  # Envoi WhatsApp piloté par OpenClaw (selon votre commande/flux)
fi

echo "$NEWHASH" > "$HASH_FILE"
```

Activer:

```bash
chmod 700 ~/assistant/core/research_daily.sh
(crontab -l 2>/dev/null; echo "15 8 * * * $HOME/assistant/core/research_daily.sh >> $HOME/assistant/logs/research.log 2>&1") | crontab -
```

---

## 13) memU : persistance + mode dégradé

### 13.1 Checklist preuve de persistance

1. Écrire une règle (ex: identité admin) dans memU.
2. Redémarrer runtime:

```bash
systemctl --user restart openclaw-gateway.service
```

3. Vérifier que la règle est relue après restart (question de contrôle via WhatsApp + vérification des fichiers memU).

### 13.2 Si memU indisponible

Basculer automatiquement en mode dégradé:

- read-only,
- aucune action externe,
- aucune suggestion intrusive d’exécution,
- notifier admin que memU est indisponible.

---

## 14) Permissions et isolation dossiers

Appliquer:

```bash
chmod 700 ~/assistant ~/.openclaw ~/assistant/quarantine ~/assistant/approved ~/assistant/logs
```

Option `noexec`:

- Possible sur certains montages Linux dédiés.
- Sous WSL2 standard, c’est souvent limité/non pertinent sur `/home`.
- Si indisponible: compenser avec `run_safe.sh`, chemins stricts, séparation runtime/staging, et politique READ/WRITE-ACTION.

---

## 15) Tests de validation / smoke tests

### 15.1 READ: scan LAN (sans nmap)

```bash
ip neigh || true
arp -an || cat /proc/net/arp
for i in 1 10 20; do ping -c 1 -W 1 192.168.1.$i || true; done
```

> Si `nmap` est déjà installé, vous pouvez faire un ping sweep LAN en complément.

### 15.2 READ: accès RTSP local (lecture)

```bash
# Exemple avec ffprobe si présent
ffprobe -v error -rtsp_transport tcp -show_streams rtsp://user:pass@192.168.1.50:554/stream1 || true
```

### 15.3 READ: requête HTTPS sortante (lecture)

```bash
curl -I https://api.github.com
```

### 15.4 WRITE/ACTION: ouverture de port sans approval -> refus

```bash
~/assistant/core/run_safe.sh nc -l 0.0.0.0 9999
# attendu: DENY
```

### 15.5 WRITE/ACTION: ajout destination non allowlist -> demande approval

```text
Tentative: ajouter new-api.example.com à l'egress allowlist
Attendu: création d'une demande WhatsApp admin + pas d'ajout immédiat
```

### 15.6 WRITE/ACTION: promote sans /approve -> refus

```text
Tentative: lancer promote_approved.sh sans approval tracée
Attendu: refus logique orchestrateur + entrée audit DENY
```

### 15.7 Exfiltration: envoi vers domaine non listé -> bloqué + log

```text
Tentative: envoyer un log/image vers un domaine externe non allowlist
Attendu: blocage + ligne d'audit (destination, horodatage, motif)
```

---

## 16) Mise en production (checklist finale)

1. systemd actif dans WSL2 (PID1 = systemd).
2. Une seule instance OpenClaw active (`openclaw-gateway.service`).
3. Dossiers `~/assistant/*` créés et permissions `700` appliquées.
4. Runner unique `~/assistant/core/run_safe.sh` actif.
5. Politique READ libre / WRITE-ACTION contrôlée documentée et appliquée.
6. Contrôle d’exfiltration (allowlist + audit) en place.
7. Pipeline quarantine -> scan -> promote opérationnel.
8. WhatsApp canal unique + allowlist numéros stricte.
9. Double confirmation active: `/approve <id>` puis `CONFIRM <id>` (ou `ok` explicite sur demande ciblée).
10. Audit log actif + rotation configurée.

---

## 17) Exploitation quotidienne (WhatsApp-first, zéro shell au quotidien)

Une fois installé:

- Vous pilotez via WhatsApp.
- L’IA lit, analyse, benchmarke, scanne et prépare en staging sans friction.
- L’IA ne promeut ni n’applique en runtime sans approbation admin explicite.
- Le shell sert à l’installation initiale, maintenance planifiée et audit.
