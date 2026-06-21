# oracle-ai-groundtruth

**La source de vérité pour l'IA Oracle.**

Deux outils pour deux moments différents :

- **Avant d'utiliser un plugin IA Oracle** → auditez-le avec `AUDIT.md`
- **Pendant que vous travaillez avec un LLM sur Oracle** → guidez-le avec `SKILL.md`

---

## Pourquoi ce repo existe

Les LLMs sont utiles pour travailler sur Oracle. Mais ils ont deux défauts majeurs en environnement critique.

**Ils hallucinent.** Une réponse fluide et crédible n'est pas une réponse correcte. Sur un environnement RAC + Data Guard, une recommandation inventée n'est pas un détail — c'est un risque opérationnel.

**Ils appliquent littéralement.** Un template conçu pour guider un développeur humain contient du jugement implicite. L'humain lit, comprend, adapte. Le LLM applique — à l'échelle, sur votre base de production, sans les questions qu'un expert aurait posées.

Ce repo répond aux deux problèmes.

---

## Les deux outils

### `SKILL.md` — Guidance documentaire Oracle

Un system prompt expert Oracle 19c HA/DR qui impose des références obligatoires sur chaque réponse.

Toute réponse doit citer :
- le document Oracle officiel exact
- le chapitre
- la page
- une citation vérifiable

Sinon, ce n'est pas une réponse fiable. C'est une hypothèse bien formulée.

**Périmètre couvert :**
Grid Infrastructure · RAC · Data Guard · GoldenGate · ASM · APEX · ORDS · SQLcl · OCI · Sécurité Oracle

**Usage :** chargez `SKILL.md` dans votre projet Claude, ChatGPT, ou tout autre LLM comme system prompt ou fichier de contexte.

---

### `AUDIT.md` — Prompt d'audit de sécurité IA

Un prompt d'audit offensif et défensif pour analyser n'importe quel repo de plugins IA Oracle avant de l'installer ou de l'utiliser comme base de génération.

Ce prompt identifie en une passe :
- les vulnérabilités d'exécution de code sans sandbox
- les angles morts entre domaines (ex : isolation compte CI/CD vs compte agent)
- les règles documentées mais non enforced techniquement
- les vecteurs de supply chain
- les fichiers `.md` utilisés comme instructions d'agents
- les mécanismes d'installation sans vérification d'intégrité

**Ce prompt a été utilisé pour auditer `oracle/skills`** — le repo officiel Oracle de plugins IA pour Claude Code. Le rapport complet est disponible dans `examples/oracle-skills-audit.md`.

**Usage :** chargez `AUDIT.md` dans Claude et fournissez le repo à auditer.

---

## Démonstration concrète

J'ai testé `oracle/skills` avec ce prompt d'audit. En partant de zéro — compte OCI gratuit, Claude Code installé — voici ce que j'ai observé en 3 minutes 47 secondes de génération :

**Installation sans intégrité**
```
Review skills before use; they run with full agent permissions.
```
Aucun hash. Aucune signature. Aucune vérification de ce qui vient d'être téléchargé.

**Un seul compte Oracle pour pipeline CI/CD et agent APEX**
```yaml
DB_USERNAME: ${{ secrets.DB_USERNAME }}
DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
DB_CONNECTION: ${{ secrets.DB_CONNECTION }}
```
Le même compte déploie le schéma DDL et exécute du PL/SQL via le chatbot. Sans avertissement.

**Outil d'exécution PL/SQL sans RBAC ni sandbox**
```
tool update-order (
    type: executeServersideCode
    settings {
        language: plsql
        plsqlCode: orders_mgmt.update_order(...)
    }
)
```
Pas d'`authorizationScheme`. Pas de validation humaine. Pas de restriction sur le périmètre d'exécution.

Ce n'est pas un reproche à Oracle. Le corpus recommande correctement le least privilege — par domaine. Ce qui manque, c'est la règle transversale qui dit que le compte du pipeline et le compte de l'agent ne doivent pas être le même. Cette règle n'existe nulle part dans le corpus. Et quand un LLM assemble les templates, personne ne la pose à sa place.

> Un template conçu pour guider un humain devient une instruction exécutée par une machine. La différence entre les deux, c'est le jugement. Et le jugement ne se documente pas dans un fichier `.md`.

---

## Installation

### SKILL.md — Guidance documentaire

**Option 1 — Claude Projects**
Copiez le contenu de `SKILL.md` dans les instructions de votre projet Claude.

**Option 2 — Claude Code**
```bash
npx skills add [votre-username]/oracle-ai-groundtruth
```

**Option 3 — Tout autre LLM**
Copiez le contenu de `SKILL.md` comme system prompt.

### AUDIT.md — Prompt d'audit

Copiez le contenu de `prompts/AUDIT.md` dans les instructions d'un projet Claude dédié à l'audit, puis fournissez le repo à analyser.

---

## Structure du repo

```
oracle-ai-groundtruth/
├── README.md                       ← ce fichier
├── SKILL.md                        ← guidance documentaire Oracle
├── prompts/
│   └── AUDIT.md                    ← prompt d'audit de sécurité IA
│   └── ...                         ← futurs prompts
└── examples/
    └── oracle-skills-audit.md      ← rapport d'audit oracle/skills
```

---

## Principes

**Toute réponse doit être traçable.** Pas juste "la doc Oracle". Le document exact, le chapitre, la page, la citation vérifiable.

**Les bonnes pratiques documentées ne sont pas des protections.** Un finding de sécurité peut exister même si le repo recommande la bonne pratique — si elle n'est pas enforced techniquement.

**Un LLM applique littéralement.** Ne jamais supposer qu'il fera preuve du même jugement qu'un expert humain qui lirait et adapterait.

---

## Sources de référence

| Source | URL |
|--------|-----|
| OWASP GenAI Security | https://genai.owasp.org |
| MITRE ATLAS | https://atlas.mitre.org |
| NIST AI RMF | https://www.nist.gov/itl/ai-risk-management-framework |
| CISA AI | https://www.cisa.gov/ai |
| ANSSI IA | https://cyber.gouv.fr/enjeux-technologiques/intelligence-artificielle/ |
| ENISA Threat Landscape | https://www.enisa.europa.eu/topics/cyber-threats/enisa-threat-landscape |
| CERT-FR | https://www.cert.ssi.gouv.fr |
| BSI IA | https://www.bsi.bund.de |

---

## Licence

MIT — libre d'utilisation, de modification, et de distribution.

---

*oracle-ai-groundtruth est un projet indépendant, non affilié à Oracle Corporation.*
