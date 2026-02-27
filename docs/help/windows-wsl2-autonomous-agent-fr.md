---
summary: "Guide V2 sécurisé pour déployer OpenClaw sur Windows 11 + WSL2 en mode WhatsApp-first avec un seul runtime et un staging contrôlé"
read_when:
  - Vous voulez piloter OpenClaw uniquement via WhatsApp
  - Vous voulez une architecture autonome mais sous contrôle admin strict
  - Vous voulez durcir runtime, staging et audit sur une home box
title: "OpenClaw V2 sécurisé sur Windows + WSL2 (WhatsApp-first)"
---

# OpenClaw V2 sécurisé sur Windows 11 + WSL2 (WhatsApp-first)

Ce guide installe **une seule instance OpenClaw** (service `systemd --user`) sur WSL2.

- **Admin (vous)** = décideur final.
- **IA** = opérateur en staging + exécution runtime **uniquement après approbation explicite WhatsApp**.
- **Aucune action externe sensible** (mail, paiement, appel API critique, commandes de production) sans validation admin.

---

## 1) Principe central : un seul OpenClaw (runtime verrouillé) + staging opéré par l’IA

```text
                        WhatsApp (canal unique)
                                 |
               +-----------------+-----------------+
               |                                   |
       user commands                       admin commands
   (lecture / questions)           (/approve, /reject, CONFIRM)
               |                                   |
               +-----------------+-----------------+
                                 |
                         OpenClaw Runtime
                 (UNIQUE service systemd actif 24/7)
                                 |
                 +---------------+----------------+
                 |                                |
         lecture contrôlée                  exécution contrôlée
   (runtime/workspaces/logs/approved)   (UNIQUE runner: run_safe.sh)
                 |
      +----------+----------------------------------------------+
      |                                                         |
 staging (scripts + dossiers, pas un service OpenClaw)   quarantine/approved
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

> **Important**: `staging` est un espace de travail (fichiers/scripts), **pas un second OpenClaw**.

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
sudo apt install -y curl wget ca-certificates jq ripgrep git logrotate coreutils findutils cron util-linux
```

### 2.4 Node 22 + OpenClaw

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g openclaw@latest
node -v
openclaw --version
```

---

## 3) Configuration OpenClaw (WhatsApp-only, séparation logique user/admin)

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

3. Principes de sécurité à appliquer dans votre config OpenClaw:

- Canal unique: `whatsapp`.
- Allowlist stricte des numéros admin (ex: `+33...`).
- Séparation logique:
  - **user commands**: lecture, questions, synthèse.
  - **admin commands**: approbation/rejet/changements runtime.
- Double confirmation pour actions sensibles:
  - `/approve <id>`
  - `CONFIRM <id>`
- Journaliser toute validation/rejet dans `~/assistant/logs/audit.log`.

4. Pairing + statut WhatsApp:

```bash
openclaw channels login --channel whatsapp
openclaw channels status --probe
```

---

## 4) Safe Runner / Command Executor (allowlist = 1 exécutable)

Le runtime ne doit exécuter **qu’un seul binaire/script**: `~/assistant/core/run_safe.sh`.

### 4.1 Script complet `~/assistant/core/run_safe.sh`

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
  rg|jq|cat|head|tail|sed|awk|ls|pwd)
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

### 4.2 Règle OpenClaw

Le composant d’exécution doit pointer **uniquement** vers:

```text
/home/<user>/assistant/core/run_safe.sh
```

Aucune autre commande directe autorisée.

---

## 5) Code externe : quarantine -> scan -> promote (sans service parallèle)

### 5.1 Script `~/assistant/core/fetch_to_quarantine.sh`

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

### 5.2 Script `~/assistant/core/scan_quarantine.sh`

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

### 5.3 Script `~/assistant/core/promote_approved.sh`

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
  sha256sum $(find "$DEST" -type f | sort)
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

## 6) Workflow d’auto-amélioration contrôlée (PROPOSE -> DEFEND -> VALIDATE -> APPLY)

1. **Detect**: l’IA détecte un nouveau skill/outillage (ex: optimisation tokens, MCP).
2. **Fetch en quarantine**: `fetch_to_quarantine.sh`.
3. **Scan + rapport**: `scan_quarantine.sh` + rapport lisible:
   - résumé fonctionnel,
   - permissions,
   - dépendances,
   - risques,
   - impact tokens estimé (bench si possible),
   - scripts/fichiers suspects,
   - diff proposé (config/runner).
4. **Demande admin WhatsApp**.
5. **Admin décide**:
   - `/approve <id>` puis `CONFIRM <id>`
   - ou `/reject <id>`
6. **Si approuvé**:
   - `promote_approved.sh <dir> <id>`
   - redémarrage runtime contrôlé.
7. **Audit complet** dans `~/assistant/logs/audit.log`.

### Commandes WhatsApp (convention opératoire)

- `/review <package>` -> lance analyse staging + rapport.
- `/showdiff <proposal>` -> montre diff de proposition.
- `/approve <id>` puis `CONFIRM <id>` -> promotion/apply autorisés.
- `/reject <id>` -> blocage + archivage.

---

## 7) Service systemd user runtime durci

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

## 8) Runtime vs réseau/install

Règles runtime (prod):

- pas d’installation de paquet,
- exécution via `run_safe.sh` seulement,
- pas d’accès staging/quarantine depuis runtime,
- réseau du runtime à minimiser (loopback pour gateway + flux nécessaires WhatsApp).

Vérification pratique (smoke plus bas): tenter une commande d’installation via runner -> doit être refusée.

---

## 9) Logs, audit append-only et rotation

### 9.1 Audit append-only

```bash
touch ~/assistant/logs/audit.log
chmod 600 ~/assistant/logs/audit.log
if command -v chattr >/dev/null 2>&1; then
  chattr +a ~/assistant/logs/audit.log 2>/dev/null || true
fi
```

> Sous WSL2, `chattr +a` peut être non supporté selon FS/driver. Dans ce cas, garder permissions strictes + audit via scripts only.

### 9.2 Rotation hebdo + compression

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

## 10) Budgets tokens (stricts)

Valeurs par défaut recommandées:

- `150000` tokens / jour.
- `30000` tokens / tâche.
- hard stop à **80%** sans override admin.

Mode “session analyse temporaire”:

1. Admin autorise explicitement une hausse ponctuelle.
2. L’IA journalise l’override (raison, durée).
3. Retour automatique au profil strict à la fin de la session.

---

## 11) Daily research sans spam

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

## 12) memU : persistance + mode dégradé

### 12.1 Checklist preuve de persistance

1. Écrire une règle (ex: identité admin) dans memU.
2. Redémarrer runtime:

```bash
systemctl --user restart openclaw-gateway.service
```

3. Vérifier que la règle est relue après restart (question de contrôle via WhatsApp + vérification des fichiers memU).

### 12.2 Si memU indisponible

Basculer automatiquement en mode dégradé:

- read-only,
- aucune action externe,
- aucune suggestion intrusive d’exécution,
- notifier admin que memU est indisponible.

---

## 13) Permissions et isolation dossiers

Appliquer:

```bash
chmod 700 ~/assistant ~/.openclaw ~/assistant/quarantine ~/assistant/approved ~/assistant/logs
```

Option `noexec`:

- Possible sur certains montages Linux dédiés.
- Sous WSL2 standard, c’est souvent limité/non pertinent sur `/home`.
- Si indisponible: compenser avec `run_safe.sh`, chemins stricts, et séparation runtime/staging.

---

## 14) Tests de validation / smoke tests

### 14.1 `run_safe` accepte `git status`

```bash
mkdir -p ~/assistant/workspaces/demo && cd ~/assistant/workspaces/demo
git init -q
~/assistant/core/run_safe.sh git status
```

### 14.2 `run_safe` refuse injection `; rm -rf`

```bash
~/assistant/core/run_safe.sh git status ';' rm -rf /
# attendu: DENY
```

### 14.3 staging `fetch -> scan -> promote`

```bash
QDIR=$(~/assistant/core/fetch_to_quarantine.sh https://example.com testpkg | awk '{print $2}')
# si URL exemple inutilisable, remplacer par source réelle interne validée
~/assistant/core/scan_quarantine.sh "${QDIR#OK: }"
~/assistant/core/promote_approved.sh "${QDIR#OK: }" proposal-001
```

### 14.4 runtime ne peut pas installer

```bash
~/assistant/core/run_safe.sh npm install left-pad
# attendu: DENY
```

### 14.5 memU persiste après restart

```bash
systemctl --user restart openclaw-gateway.service
systemctl --user status openclaw-gateway.service --no-pager
# puis vérifier une donnée memU écrite avant restart
```

---

## 15) Mise en production (checklist finale, 10 points max)

1. systemd actif dans WSL2 (PID1 = systemd).
2. Une seule instance OpenClaw active (`openclaw-gateway.service`).
3. Dossiers `~/assistant/*` créés et permissions `700` appliquées.
4. Runner unique `~/assistant/core/run_safe.sh` actif.
5. Pipeline quarantine -> scan -> promote opérationnel.
6. WhatsApp canal unique + allowlist admin stricte.
7. Double confirmation active: `/approve <id>` puis `CONFIRM <id>`.
8. Audit log actif + rotation hebdo configurée.
9. Budgets tokens stricts (150k/j, 30k/tâche, stop à 80%).
10. memU validé (persistance OK) ou mode dégradé activé.

---

## 16) Exploitation quotidienne (WhatsApp-first, zéro shell au quotidien)

Une fois installé:

- Vous pilotez via WhatsApp.
- L’IA peut lire/analyser/préparer en staging.
- L’IA ne promeut ni n’applique en runtime sans votre approbation explicite.
- Le shell sert uniquement à l’installation initiale, maintenance planifiée et audit.
