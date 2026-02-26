---
summary: "Guide production pour OpenClaw autonome sur Windows 11 Pro avec WSL2 Ubuntu sans Docker"
read_when:
  - Vous voulez un agent autonome discipliné avec OpenClaw sur WSL2
  - Vous cherchez une architecture sécurisée avec WhatsApp, memU et budgets tokens
  - Vous avez besoin d'un workflow auto-amélioration avec validation admin
title: "Windows 11 Pro plus WSL2 Ubuntu pour agent autonome discipliné"
---

# Guide complet OpenClaw sur Windows 11 Pro avec WSL2 sans Docker

Ce guide décrit une architecture production-ready pour un agent autonome avancé avec OpenClaw sur Windows 11 Pro, Ubuntu WSL2 et systemd activé, sans Docker.

## 1. Architecture cible

- Orchestrateur: OpenClaw.
- OS hôte: Windows 11 Pro.
- Runtime Linux: Ubuntu sous WSL2 avec systemd.
- Transport principal: WhatsApp.
- Mémoire longue durée: memU.
- Modèles: souscriptions officielles OpenAI Codex et Gemini.
- Politique skills: uniquement local (`~/assistant/skills/enabled`), marketplace interdit.
- Sécurité: écriture locale contrôlée et actions externes sous validation admin.

## 2. Pré-requis Windows 11 Pro

Ouvrir un terminal PowerShell en administrateur et exécuter:

```powershell
wsl --install -d Ubuntu
wsl --set-default-version 2
wsl --update
```

Redémarrer Windows, puis lancer Ubuntu une première fois.

Vérifier côté Ubuntu:

```bash
uname -a
wsl.exe --status
```

## 3. Activer systemd dans WSL2

Dans Ubuntu:

```bash
sudo tee /etc/wsl.conf >/dev/null <<'EOWSLCONF'
[boot]
systemd=true
EOWSLCONF
```

Depuis PowerShell Windows:

```powershell
wsl --shutdown
```

Relancer Ubuntu et vérifier:

```bash
systemctl is-system-running
ps -p 1 -o comm=
```

Résultat attendu: PID 1 = `systemd`.

## 4. Base système Ubuntu

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl git jq ca-certificates build-essential python3 python3-venv python3-pip cron ripgrep
```

## 5. Installer Node 22 et pnpm

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
node -v
npm -v
sudo npm install -g pnpm
pnpm -v
```

## 6. Installer OpenClaw globalement

```bash
sudo npm install -g openclaw@latest
openclaw --version
```

Créer la hiérarchie contrôlée:

```bash
mkdir -p ~/assistant/{core,skills/staging,skills/enabled,sandbox,logs,state,feeds}
chmod 700 ~/assistant ~/assistant/state ~/assistant/logs ~/assistant/sandbox
chmod 750 ~/assistant/skills ~/assistant/skills/enabled ~/assistant/skills/staging
```

## 7. Authentification providers par login officiel

Utiliser les flux de login supportés par OpenClaw sans stocker de clés en clair dans des fichiers partagés:

```bash
openclaw login
```

Puis configurer les providers et modèles par défaut:

```bash
openclaw config set model.provider openai
openclaw config set model.name codex-5.3
openclaw config set model.fallbackProvider gemini
openclaw config set model.fallbackName gemini-3.1-pro
```

Adapter les IDs exacts aux noms disponibles dans votre version OpenClaw.

## 8. Configuration OpenClaw sécurisée

Fichier: `~/.openclaw/openclaw.json`

```json
{
  "gateway": {
    "mode": "local",
    "bind": "loopback",
    "port": 18789
  },
  "routing": {
    "defaultChannel": "whatsapp"
  },
  "channels": {
    "whatsapp": {
      "enabled": true,
      "allowlist": ["+33600000001", "+33600000002"],
      "adminIdentity": "+33600000001"
    }
  },
  "skills": {
    "marketplace": {
      "enabled": false
    },
    "external": {
      "enabled": false
    },
    "local": {
      "enabled": true,
      "paths": ["/home/<user>/assistant/skills/enabled"],
      "stagingPath": "/home/<user>/assistant/skills/staging",
      "allowWrites": [
        "/home/<user>/assistant/skills",
        "/home/<user>/assistant/logs",
        "/home/<user>/assistant/state"
      ]
    }
  },
  "memory": {
    "provider": "memu",
    "path": "/home/<user>/assistant/state/memu",
    "compaction": {
      "enabled": true,
      "maxItems": 20000,
      "summaryIntervalHours": 24
    }
  },
  "security": {
    "requireAdminApprovalForExternalActions": true,
    "commandAllowlist": [
      "pwd",
      "ls",
      "cat",
      "jq",
      "rg",
      "diff",
      "git status",
      "git diff",
      "python3",
      "node",
      "openclaw"
    ],
    "denyNetworkCodeDownload": true,
    "auditLogPath": "/home/<user>/assistant/logs/audit.log"
  },
  "agent": {
    "primaryAdmin": {
      "id": "+33600000001",
      "name": "admin-principal"
    },
    "externalActionPolicy": "ask-admin-explicitly",
    "dailyResearch": {
      "maxSessions": 1,
      "maxNotifications": 1,
      "weeklyDigestDay": "sunday"
    }
  },
  "tokens": {
    "dailyBudget": 1200000,
    "perTaskBudget": 120000,
    "roiThreshold": 1.2,
    "cache": {
      "enabled": true,
      "path": "/home/<user>/assistant/state/cache"
    }
  }
}
```

### Points clés de sécurité

- `allowlist` WhatsApp stricte.
- `adminIdentity` unique.
- Marketplace et skills externes désactivés.
- Écritures limitées à `~/assistant/*`.
- Actions externes bloquées sans validation explicite.

## 9. Service systemd OpenClaw

Créer l'unité utilisateur:

```bash
mkdir -p ~/.config/systemd/user
cat > ~/.config/systemd/user/openclaw-gateway.service <<'EOSERVICE'
[Unit]
Description=OpenClaw Gateway
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/env openclaw gateway run --bind loopback --port 18789 --force
Restart=always
RestartSec=5
Environment=OPENCLAW_STATE_DIR=%h/.openclaw
Environment=HOME=%h
WorkingDirectory=%h
StandardOutput=append:%h/assistant/logs/gateway.log
StandardError=append:%h/assistant/logs/gateway.err.log

[Install]
WantedBy=default.target
EOSERVICE
```

Activer et démarrer:

```bash
systemctl --user daemon-reload
systemctl --user enable --now openclaw-gateway.service
systemctl --user status openclaw-gateway.service --no-pager
```

Activer le user-linger facultatif mais recommandé:

```bash
sudo loginctl enable-linger "$USER"
```

## 10. Configuration memU opérationnelle

Créer l'arborescence mémoire:

```bash
mkdir -p ~/assistant/state/memu/{identity,decisions,optimizations,errors,summaries}
```

Créer un bootstrap initial:

```bash
cat > ~/assistant/state/memu/identity/admin-principles.md <<'EOMEM'
# Identité administrateur
- Admin principal: +33600000001
- Priorité absolue: demandes explicites de l admin principal
- Aucune action externe sans validation explicite
- Curiosité autorisée mais disciplinée
EOMEM
```

## 11. Cron jobs discipline et maintenance

Installer crontab utilisateur:

```bash
crontab -l 2>/dev/null > /tmp/mycron || true
cat >> /tmp/mycron <<'EOCRON'
# Veille quotidienne une fois par jour
15 8 * * * /home/<user>/assistant/core/research-daily.sh >> /home/<user>/assistant/logs/research.log 2>&1

# Compression memoire quotidienne
30 8 * * * /home/<user>/assistant/core/memory-compact.sh >> /home/<user>/assistant/logs/memory.log 2>&1

# Digest hebdo dimanche
0 9 * * 0 /home/<user>/assistant/core/weekly-digest.sh >> /home/<user>/assistant/logs/digest.log 2>&1

# Rapport tokens hebdo
30 9 * * 0 /home/<user>/assistant/core/tokens-weekly-report.sh >> /home/<user>/assistant/logs/tokens.log 2>&1
EOCRON
crontab /tmp/mycron
rm -f /tmp/mycron
crontab -l
```

## 12. Scripts minimaux core

### 12.1 Veille quotidienne contrôlée

`~/assistant/core/research-daily.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

LOCK=~/assistant/state/research.lock
TODAY=$(date +%F)
STAMP=~/assistant/state/research.last

if [[ -f "$STAMP" && "$(cat "$STAMP")" == "$TODAY" ]]; then
  exit 0
fi

exec 9>"$LOCK"
flock -n 9 || exit 0

# Strategie offline first: collecter d abord les signaux locaux
rg -n "TODO|FIXME|debt|token|latency" ~/assistant ~/workspace/openclaw > ~/assistant/feeds/local-signals.txt || true

# Appel modele seulement si signaux actionnables detectes
if [[ -s ~/assistant/feeds/local-signals.txt ]]; then
  openclaw message send --channel whatsapp --to +33600000001 <<'EOMSG'
Veille quotidienne: signaux actionnables detectes.
Synthese disponible dans ~/assistant/feeds/local-signals.txt.
Souhaitez-vous une proposition de plan en 3 actions prioritaires?
EOMSG
fi

echo "$TODAY" > "$STAMP"
```

### 12.2 Compaction mémoire intelligente

`~/assistant/core/memory-compact.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

MEMROOT=~/assistant/state/memu
OUT="$MEMROOT/summaries/summary-$(date +%F).md"

{
  echo "# Resume memoire $(date +%F)"
  echo
  echo "## Decisions"
  tail -n 200 "$MEMROOT/decisions"/*.md 2>/dev/null || true
  echo
  echo "## Optimisations"
  tail -n 200 "$MEMROOT/optimizations"/*.md 2>/dev/null || true
  echo
  echo "## Erreurs"
  tail -n 200 "$MEMROOT/errors"/*.md 2>/dev/null || true
} > "$OUT"
```

### 12.3 Rapport tokens hebdomadaire

`~/assistant/core/tokens-weekly-report.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

LOG=~/assistant/logs/audit.log
OUT=~/assistant/logs/tokens-weekly-$(date +%F).md

{
  echo "# Rapport tokens hebdomadaire"
  echo
  echo "## Budget"
  echo "- Journalier: 1 200 000"
  echo "- Par tache: 120 000"
  echo
  echo "## Heuristique ROI"
  echo "- N appeler le modele que si gain attendu > cout"
  echo "- Priorite aux operations offline: rg, diff, logs"
  echo
  echo "## Top appels"
  rg -n "MODEL_CALL|TOKENS" "$LOG" | tail -n 200 || true
} > "$OUT"
```

Rendre exécutables:

```bash
chmod +x ~/assistant/core/*.sh
```

## 13. Budget tokens et discipline opérationnelle

Politique recommandée:

- Budget journalier global: 1.2M tokens.
- Budget par tâche: 120k tokens.
- Hard stop à 85 pourcent du budget journalier sauf override admin.
- Offline-first systématique:
  1. `rg` local.
  2. lecture logs.
  3. `git diff`.
  4. shortlist des hypothèses.
  5. appel LLM en dernier.
- Cache agressif des résultats stables.

## 14. Workflow auto-amélioration sécurisé

### Pipeline obligatoire

1. Idée d'amélioration capturée dans `~/assistant/skills/staging`.
2. Génération du skill local uniquement sans marketplace.
3. Tests unitaires et tests sandbox.
4. Rapport de risque et rollback plan.
5. Validation explicite admin principal.
6. Activation vers `~/assistant/skills/enabled`.
7. Monitoring 24 heures.
8. Rollback automatique si échec.

### Script promotion contrôlée

`~/assistant/core/promote-skill.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

SKILL_NAME="${1:?usage: promote-skill.sh <skill-name>}"
ADMIN_APPROVED_FILE=~/assistant/state/admin-approvals/${SKILL_NAME}.ok
STAGING=~/assistant/skills/staging/${SKILL_NAME}
ENABLED=~/assistant/skills/enabled/${SKILL_NAME}

[[ -d "$STAGING" ]] || { echo "Skill staging introuvable"; exit 1; }
[[ -f "$ADMIN_APPROVED_FILE" ]] || { echo "Validation admin manquante"; exit 2; }

if [[ -x "$STAGING/test.sh" ]]; then
  "$STAGING/test.sh"
fi

rm -rf "$ENABLED.bak" || true
if [[ -d "$ENABLED" ]]; then
  mv "$ENABLED" "$ENABLED.bak"
fi
cp -a "$STAGING" "$ENABLED"

echo "Promotion OK: $SKILL_NAME"
```

Rollback:

```bash
#!/usr/bin/env bash
set -euo pipefail
SKILL_NAME="${1:?usage: rollback-skill.sh <skill-name>}"
ENABLED=~/assistant/skills/enabled/${SKILL_NAME}
BACKUP=~/assistant/skills/enabled/${SKILL_NAME}.bak
[[ -d "$BACKUP" ]] || { echo "Backup absent"; exit 1; }
rm -rf "$ENABLED"
mv "$BACKUP" "$ENABLED"
```

## 15. Politique interne agent constitution

Créer `~/assistant/state/constitution.md`:

```markdown
# Constitution agent autonome

## Mission

Servir l administrateur principal avec discipline, securite et efficacite.

## Regles absolues

1. Lecture locale et distante autorisee.
2. Ecriture autorisee uniquement dans espaces controles.
3. Actions externes interdites sans validation explicite admin.
4. Priorite absolue aux demandes admin principal.
5. Maximum une session de veille par jour, maximum une notification veille par jour.
6. Jamais de marketplace skills.
7. Toujours proposer un plan de rollback avant activation d une amelioration.

## Gouvernance

- Admin principal: +33600000001
- Interlocuteurs secondaires: lecture et consultation uniquement.

## Discipline tokens

- Offline-first obligatoire.
- Budget journalier et budget par tache non depassables sans validation admin.
```

## 16. Mise à jour contrôlée OpenClaw

Workflow recommandé:

1. Snapshot config et état.
2. Mise à jour en staging.
3. Tests de non-régression.
4. Validation admin.
5. Bascule production.
6. Rollback si incident.

Exemple:

```bash
mkdir -p ~/assistant/state/backups
cp ~/.openclaw/openclaw.json ~/assistant/state/backups/openclaw.json.$(date +%F-%H%M%S)

openclaw --version
sudo npm i -g openclaw@latest
openclaw --version

openclaw channels status --probe
systemctl --user restart openclaw-gateway.service
systemctl --user status openclaw-gateway.service --no-pager
```

## 17. Vérifications finales production

```bash
systemctl --user is-active openclaw-gateway.service
openclaw channels status --probe
ss -ltnp | rg 18789
tail -n 120 ~/assistant/logs/gateway.log
crontab -l
```

Checklist:

- Service stable après redémarrage WSL2.
- WhatsApp limité à la allowlist.
- Admin principal reconnu.
- memU enregistre identité, décisions, erreurs, optimisations.
- Veille limitée à une notification par jour.
- Skills externes bloqués, marketplace désactivé.
- Workflow staging test validation activation rollback opérationnel.

## 18. Notes exploitation WSL2

- Stocker tous les états dans le système de fichiers Linux `/home/...` plutôt que dans `/mnt/c/...`.
- Synchroniser l'horloge hôte et vérifier timezone Ubuntu.
- Sauvegarder `~/assistant/state` et `~/.openclaw` quotidiennement.

Ce setup donne un agent autonome avancé, discipliné et contrôlable, sans Docker, adapté à WSL2.
