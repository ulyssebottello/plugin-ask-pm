# 🏷️ Ask-PM — Product Decision Escalation Plugin

> Quand Claude tombe sur une ambiguïté produit, il pose la question au PM
> dans Linear. Il attend la réponse (ou continue sur le reste), l'applique,
> et met à jour la description de l'issue comme PRD vivant.

## Quickstart

**Prérequis :** Claude Code + [MCP Linear](https://linear.app/docs/mcp) configuré.

### Installer

Dans Claude Code :

```
/plugin marketplace add ulyssebottello/plugin-ask-pm
```

```
/plugin install ask-pm@ask-pm-marketplace
```

### Utiliser — session interactive

Donne un ticket Linear à Claude et code avec lui. Quand il tombe sur une ambiguïté, il pose la question au PM automatiquement.

```
Implémente https://linear.app/team/issue/ENG-456
```

Si le PM a répondu entre-temps et que tu veux que Claude applique la décision :

```
/ask-pm check
```

### Utiliser — mode autonome (headless)

Claude travaille seul, poll les réponses du PM toutes les 2 min (max 30 min) :

```bash
claude --plugin-dir ~/Desktop/plugin-ask-pm -p "Implémente https://linear.app/team/issue/ENG-456" --allowedTools '*'
```

Ou en mode interactif avec le plugin chargé manuellement (utile pour tester) :

```bash
claude --plugin-dir ~/Desktop/plugin-ask-pm
```

### Mettre à jour

```bash
claude plugin update "ask-pm@ask-pm-marketplace" --scope project
```

---

## Le flow complet

```
 CLAUDE CODE (dev / autonome)                    PM
 ─────────────────────────────              ─────────────
                                            
 1. Claude détecte une ambiguïté             
    pendant l'implémentation                 
           │                                 
           ▼                                 
 2. Crée un commentaire structuré            
    sur le ticket Linear                     
    (options, tradeoffs, contexte)           
           │                                 
           ├─── Continue sur les     ───►   📩 Le PM voit le
           │    tâches non bloquées          commentaire dans
           │                                 sa notif Linear
           │   ┌──────────────────┐               │
           │   │ MODE AUTONOME:   │               ▼
           │   │ poll toutes les  │          
           │   │ 2 min (max 30m)  │          Le PM répond en
           │   │ pour vérifier si │          commentaire
           │   │ le PM a répondu  │               │
           │   └──────────────────┘               │
           │                                      │
           ▼                                      │
 3. Claude lit la réponse         ◄──────────     │
    du PM dans les commentaires              
           │                                 
           ▼                                 
 4. Applique la décision dans le code        
           │                                 
           ▼                                 
 5. Confirme dans les commentaires           
    "✅ Decision applied"                    
           │                                 
           ▼                                 
 6. Met à jour la DESCRIPTION                
    de l'issue pour refléter                 
    la décision (PRD vivant)                 
```

## Quand est-ce que Claude va checker la réponse ?

C'est LA question. Trois cas :

### Cas 1 — Le dev est là (session interactive)

Claude poste la question, continue sur le reste, et **c'est le dev qui dit**
"le PM a répondu" ou "check Linear". Claude va lire et applique. Simple.

### Cas 2 — Agent autonome (headless / background)

Claude poste la question, épuise les tâches non bloquées, puis **poll en boucle** :
- Lit les commentaires du ticket via le MCP Linear
- Cherche une réponse après son commentaire Ask-PM
- Si rien : `sleep 120` → recommence
- **Max 15 tentatives** (~30 minutes)
- Si toujours rien après 30min : s'arrête proprement et laisse un résumé
  de ce qui est fait et ce qui reste bloqué

### Cas 3 — Nouvelle session sur le même ticket

La skill détecte automatiquement le ticket Linear (via la branche git, le contexte,
CLAUDE.md) et vérifie s'il y a des décisions PM non appliquées pour briefer le dev.

## Mise à jour de la description (PRD vivant)

Après chaque décision du PM, Claude **met à jour la description de l'issue Linear**
en ajoutant un blockquote contextuel au bon endroit. L'objectif : n'importe qui lit
la description et voit le spec complet à jour, avec la trace des décisions PM.

Exemple :

```markdown
## Archive
Les projets archivés sont masqués du dashboard.

> **🏷️ PM Decision** _(2026-03-12)_ — Modified:
> Les tâches restent visibles avec un badge "archivé" mais sont exclues
> des filtres par défaut.
> _(See thread on issue comments)_
```

Les blockquotes indiquent **Added**, **Modified** ou **Removed** pour tracer
le type de décision.

## Composants du plugin

```
plugin-ask-pm/
├── .claude-plugin/
│   ├── plugin.json            # Manifeste du plugin
│   └── marketplace.json       # Catalogue marketplace
├── skills/
│   └── ask-pm/
│       └── SKILL.md           # Quand/comment escalader, poll, appliquer, MAJ description
├── hooks/
│   └── hooks.json             # Stop: flag unconfirmed product assumptions
└── README.md
```

4 fichiers actifs. Zéro dépendance. Zéro infra. Le MCP Linear fait tout le travail.

## Installation avancée

### Pour toute l'équipe (recommandé)

Ajoutez au `.claude/settings.json` du projet (versionné) :

```jsonc
{
  "enabledPlugins": {
    "ask-pm@ask-pm-marketplace": true
  },
  "extraKnownMarketplaces": {
    "ask-pm-marketplace": {
      "source": "github",
      "repo": "ulyssebottello/plugin-ask-pm"
    }
  }
}
```

Quand un dev trust le repo → le plugin est actif automatiquement.

### Test local (dev)

```bash
claude --plugin-dir ./plugin-ask-pm
```

## Exemple concret

**Ce que Claude poste sur ENG-456 :**

```markdown
🏷️ **Ask-PM** — Decision needed

**Context:**
Implementing project archive. The spec says "archived projects are hidden
from the dashboard" but doesn't address what happens to associated tasks.

**Question:**
When a project is archived, what happens to its tasks?

**Options:**
→ **A) Soft-delete** — Tasks disappear, restored with the project
  · _Pro:_ Clean UX · _Con:_ Orphan refs in "My Tasks"
→ **B) Badge** — Tasks stay with "archived" badge, hidden from default filter
  · _Pro:_ No data loss · _Con:_ Badge adds visual noise
→ **C) New status** — Tasks get a filterable "Archived" status
  · _Pro:_ Explicit · _Con:_ One more status to manage

**Blocked?** Yes — archive PR. Continuing on dashboard filters.

---
_🤖 Ask-PM · awaiting decision_
```

**Le PM répond en commentaire :**

> Option B. On ne perd jamais de tâches. Le badge suffit, mais masquez-les
> du filtre par défaut.

**Claude confirme (en réponse dans le thread du commentaire original) :**

```markdown
✅ **Ask-PM** — Decision applied

Applied option B: tasks remain visible with "archived" badge,
hidden from default filters. Issue description updated.
```

**Claude met à jour la description de ENG-456 :**

```markdown
## Archive
Les projets archivés sont masqués du dashboard.

> **🏷️ PM Decision** _(2026-03-12)_ — Modified:
> Les tâches restent visibles avec un badge "archivé" mais sont
> exclues du filtre par défaut.
> _(See thread on issue comments)_
```

## FAQ

**Claude va pas spammer le PM ?**
Non. La skill a des critères stricts : les choix techniques ne sont jamais escaladés.
Seules les décisions qui affectent l'expérience utilisateur remontent.

**Et si le PM met plus de 30 min en mode autonome ?**
L'agent s'arrête proprement et liste ce qui est fait vs bloqué.
Le dev reprend plus tard avec `/ask-pm check` et Claude continue.

**La description de l'issue ne va pas devenir un bordel ?**
Claude intègre les décisions au bon endroit dans la description existante,
il n'ajoute pas bêtement à la fin. Le format est préservé.

**Ça marche avec Cursor ?**
Le SKILL.md suit le standard Agent Skills. Le contenu est réutilisable.
Il faudrait adapter les instructions d'outils à l'écosystème Cursor.

---

*Pour les PMs qui veulent du signal et les devs qui veulent moins d'hypothèses.*
