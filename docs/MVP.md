# Cadrage du MVP

> Suite de [BRIEF.md](./BRIEF.md). Ce document transforme le brief en décisions
> (ou en propositions explicitement marquées **à valider**), découpe le travail en
> jalons, et fixe les deux contrats techniques qui structurent tout le reste :
> le **format de scénario** et le **protocole runner ↔ control plane**.
>
> Statut : proposition, octobre 2026. Rien n'est codé.

---

## 0. Résumé en 10 lignes

1. On commence par le **jalon zéro** : une GitHub Action open source, autonome, sans SaaS.
2. Les scénarios sont des **fichiers YAML dans le repo du client** (versioning et revue de PR gratuits).
3. Le runner est en **TypeScript** (même langage que le control plane) → benchmark de 3 options : **Playwright MCP + notre propre boucle d'agent**, **Stagehand**, **Midscene** ; browser-use écarté du MVP (Python).
4. Le code agent est caché derrière une interface `AgentDriver` : changer de lib ne doit toucher qu'un fichier.
5. Secrets de l'app cible et clé LLM : **uniquement dans les secrets GitHub du client**. Les secrets ne sont jamais envoyés au LLM (substitution au moment de l'action).
6. Verdicts à quatre valeurs : `passed` / `failed` (l'app est KO, preuve à l'appui) / `inconclusive` (l'agent n'a pas abouti, sans preuve contre l'app) / `error` (l'outil ou l'infra est KO). C'est la distinction qui économise le support et préserve la confiance.
7. Pour le MVP, le runner GitHub Action est **déclenché par la CI**, pas par polling → **pas de file de jobs** à construire au début.
8. Le commentaire de PR est posté **par l'Action elle-même** avec le `GITHUB_TOKEN` → pas d'App GitHub à créer pour le MVP.
9. Le SaaS (jalon 1) commence comme un **simple récepteur de rapports** : historique, captures, étape en échec.
10. L'édition des scénarios dans l'UI arrive au jalon 2, avec le même schéma que les fichiers.

---

## 1. Découpage en jalons

| Jalon | Contenu | Livrable | Critère de passage au suivant |
|---|---|---|---|
| **J-1 Benchmark** | 2 libs × 3 scénarios × 10 runs | Tableau de résultats + choix de lib | Une lib atteint ≥ 90 % de réussite sur les scénarios « nominaux » |
| **J0 Action OSS** | GitHub Action + rapport HTML + commentaire PR | Repo public, publié sur le Marketplace | Signal d'intérêt (voir §7) |
| **J1 Reporting SaaS** | Comptes, projets, token runner, ingestion des runs, historique | App web en ligne, plan gratuit | ≥ 5 projets actifs qui envoient des runs chaque semaine |
| **J2 Scénarios SaaS + paiement** | Édition des scénarios dans l'UI, runs à la demande, facturation | Plan payant | Premiers clients payants |

Le périmètre MVP du brief (§7) = **J0 + J1 + J2**. J0 seul est le « jalon zéro » du brief.

### Hors MVP (confirmé)
Cache/replay sans LLM, détection de flaky, runner Docker générique, GitLab CI,
runners managés, export vers bucket client, **diagnostic et correctif automatique
en cas d'échec** (§12). Mais **le format de rapport doit déjà contenir ce qu'il faudra
pour ces fonctions** (actions résolues par étape, historique par scénario, erreurs
console et réseau) — voir §3.

---

## 2. Format de scénario (décision proposée)

### Choix : un fichier YAML par scénario, dans le repo

Pourquoi YAML plutôt que du texte libre ou du Markdown :

- **étapes et critères de validation séparés** : l'agent *agit* sur les `steps`, un
  vérificateur *juge* les `expect`. Mélanger les deux dans un texte libre est la
  première source de faux positifs (l'agent « croit » avoir vérifié) ;
- parsable et validable (schéma Zod partagé entre runner et plateforme) ;
- reste lisible par un non-dev ; les étapes restent du langage naturel.

Emplacement par défaut : `.agentic/scenarios/*.yaml` (nom de dossier provisoire tant que le produit n'a pas de nom).

### Exemple

```yaml
name: Achat d'un article
description: Parcours d'achat nominal, utilisateur existant
start_url: /                     # relatif à base_url (défini dans la config ou l'Action)
tags: [smoke, checkout]
timeout: 180                     # secondes, pour tout le scénario

steps:
  - Se connecter avec l'email {{secrets.TEST_USER_EMAIL}} et le mot de passe {{secrets.TEST_USER_PASSWORD}}
  - Rechercher "T-shirt bleu" et ouvrir la fiche produit
  - Ajouter l'article au panier en taille M
  - Ouvrir le panier

expect:
  - Le panier contient exactement 1 article, "T-shirt bleu", taille M
  - Le total affiché est 19,90 €
```

### Règles

- `steps` : liste ordonnée d'instructions. Chaque étape est exécutée et reportée séparément
  (c'est ce qui permet d'afficher « étape en échec : 3 »).
- `expect` : assertions vérifiées **à la fin** par le juge (§4, « Juge »), qui répond
  vrai / faux / indéterminable en citant une preuve tirée de la page. Une étape peut aussi
  porter un `expect` local :
  ```yaml
  steps:
    - do: Valider le formulaire sans remplir l'email
      expect: Un message d'erreur indique que l'email est obligatoire
  ```
- Variables : `{{vars.X}}` (valeurs non sensibles, visibles dans le rapport) et
  `{{secrets.X}}` (lues dans l'env du runner, jamais affichées, jamais envoyées au LLM).
- Pas de `if`, pas de boucles, pas d'import en MVP. Un scénario = un parcours linéaire.

### Fichier de configuration : `agentic-test.ts`

Un fichier TypeScript à la racine du repo du client, sur le modèle de `playwright.config.ts` :
typé (autocomplétion dans l'éditeur), et capable de lire l'environnement
(`process.env`) pour adapter la config à la branche ou à l'environnement visé.

```ts
// agentic-test.ts
import { defineConfig } from "@agentic/cli";

export default defineConfig({
  baseUrl: process.env.STAGING_URL ?? "http://localhost:3000",
  scenarios: ".agentic/scenarios/**/*.yaml",
  model: "anthropic/claude-sonnet-5-5",        // format fournisseur/modèle ; la clé vient de l'env
  viewport: { width: 1280, height: 800 },

  agent: {
    maxActionsPerStep: 15,                     // budget d'actions avant le bilan forcé
  },

  // Relances automatiques (jamais pour `failed`).
  retries: {
    inconclusive: 1,
    error: 1,
  },

  // Quand la CI doit-elle échouer ? Comptage APRÈS relances.
  ci: {
    failOnFailed: true,                        // au moins un `failed` → CI rouge
    inconclusiveThreshold: 1,                  // CI rouge à partir de N scénarios `inconclusive` ; false = jamais
    errorThreshold: false,                     // idem pour `error` ; false = avertissement seulement
  },

  judge: {
    model: undefined,                          // optionnel : un modèle plus solide que l'agent (défaut : `model`)
    votes: 1,                                  // 3 = trois avis indépendants ; s'ils divergent → inconclusive
  },

  artifacts: { video: "on-failure", trace: "on-failure", screenshots: "every-step" },
});
```

Valeurs par défaut si le fichier est absent : celles ci-dessus (seul `baseUrl` doit
alors être fourni, par l'Action). Le runner charge le fichier TypeScript sans étape de
compilation côté client.

---

## 3. Résultat d'un run (contrat de sortie du runner)

Le runner produit **un seul `report.json`** + un dossier d'artefacts. Ce JSON est à la fois :
la source du rapport HTML (J0), le payload envoyé au SaaS (J1), et la matière première
du futur cache/replay et de la détection de flaky.

```jsonc
{
  "schema_version": 1,
  "run": {
    "id": "local-uuid",
    "started_at": "...", "finished_at": "...",
    "git": { "sha": "...", "ref": "...", "pr": 42, "repo": "owner/name" },
    "runner": { "kind": "github-action", "version": "0.1.0" },
    "agent": { "driver": "stagehand", "driver_version": "...", "model": "anthropic/claude-sonnet-5-5" }
  },
  "scenarios": [
    {
      "file": ".agentic/scenarios/checkout.yaml",
      "content_hash": "sha256:...",          // identifie la version du scénario
      "status": "failed",                    // passed | failed | inconclusive | error | skipped
      "duration_ms": 48210,
      "attempt": 1,
      "usage": { "input_tokens": 51234, "output_tokens": 2310, "llm_calls": 14 },
      "steps": [
        {
          "index": 0, "text": "Se connecter avec ...{{secrets.TEST_USER_PASSWORD}}",
          "status": "passed", "duration_ms": 9100,
          "actions": [                        // actions Playwright résolues → futur replay
            { "type": "fill", "selector": "#email", "value": "{{secrets.TEST_USER_EMAIL}}" },
            { "type": "click", "selector": "button[type=submit]" }
          ],
          "screenshot": "artifacts/checkout/step-0.png"
        }
      ],
      "expectations": [
        { "text": "Le total affiché est 19,90 €", "status": "failed",
          "reason": "Le total affiché est 24,90 € (frais de port inclus)",
          "screenshot": "artifacts/checkout/expect-1.png" }
      ],
      "error": null,                          // rempli si status = error (timeout, crash, quota LLM…)
      "page_signals": {                       // indices pour le futur diagnostic (§12)
        "url_at_failure": "https://staging.example.com/cart",
        "console_errors": ["TypeError: cannot read properties of undefined (reading 'price')"],
        "failed_requests": [{ "method": "POST", "url": "/api/cart", "status": 500 }]
      },
      "artifacts": { "video": "artifacts/checkout/video.webm", "trace": "artifacts/checkout/trace.zip" }
    }
  ]
}
```

Points importants :

- **Les quatre statuts d'échec possibles** :

  | Situation | Statut | Relancé ? | Bloque la CI ? |
  |---|---|---|---|
  | Étape impossible, **preuve trouvée dans la page** | `failed` | non | oui (`ci.failOnFailed`) |
  | Critère final jugé faux | `failed` | non | oui (`ci.failOnFailed`) |
  | Agent bloqué, tourne en rond, ou preuve introuvable dans la page | `inconclusive` | `retries.inconclusive` | à partir de `ci.inconclusiveThreshold` |
  | Clé invalide, quota, navigateur planté, site injoignable | `error` | `retries.error` | à partir de `ci.errorThreshold` |

  `failed` dit « l'app est en cause, et voici la preuve ». `inconclusive` dit « l'agent
  n'a pas réussi cette étape, sans pouvoir prouver que l'app est en cause ». Les
  confondre produit soit de faux rouges (perte de confiance), soit des bugs cachés.
  Le rapport et le commentaire PR les affichent différemment (rouge / orange / gris).
- Les valeurs de secrets sont **masquées** partout dans le rapport (texte, actions, logs).
- `actions` + `content_hash` = de quoi faire du replay sans LLM plus tard sans changer le format.
- `page_signals` (erreurs console, requêtes HTTP en échec, URL au moment de l'échec) :
  peu coûteux à capturer avec Playwright, et c'est le meilleur indice pour remonter
  d'un test en échec à une ligne de code (§12). À capturer dès J0.
- `usage` = argument commercial (« voilà ce que vous coûte chaque test ») et base pour
  mesurer les économies du futur cache.

---

## 4. Runner : architecture interne

```
@agentic/scenario      schéma Zod + parseur YAML + résolution des variables   (partagé runner/SaaS)
@agentic/runner-core   orchestration : charge les scénarios, lance Playwright,
                       appelle l'AgentDriver, juge les expect, écrit report.json
@agentic/driver-*      adaptateurs : driver-playwright-mcp (boucle maison), driver-stagehand, driver-midscene
@agentic/report-html   report.json → rapport HTML statique autonome
@agentic/cli           `agentic run` en local (sert aussi au debug support)
github-action          wrapper : installe Chromium, appelle le CLI, upload artefacts, commente la PR
```

Monorepo **pnpm + TypeScript** ; le control plane (`apps/web`) viendra s'y ajouter en J1
et réutilisera `@agentic/scenario` et les types du rapport.

### Interface `AgentDriver` (esquisse)

```ts
interface AgentDriver {
  init(page: Page, opts: { model: string }): Promise<void>;
  // Exécute une étape en langage naturel ; renvoie les actions concrètes effectuées.
  act(instruction: string, ctx: { secrets: Record<string, string> }): Promise<StepOutcome>;
  // Juge une assertion sur l'état courant de la page.
  check(assertion: string): Promise<{ ok: boolean; reason: string }>;
  usage(): TokenUsage;
}
```

`check` peut être implémenté une seule fois dans `runner-core` (appel LLM direct avec
capture + texte de la page) plutôt que dans chaque driver : cela garantit des verdicts
comparables quel que soit le driver.

### Boucle d'agent (driver Playwright MCP)

Pas de framework d'orchestration (LangGraph & co.) : un seul agent, un parcours
linéaire, quelques minutes. Le Vercel AI SDK fournit les appels multi-fournisseurs et
les appels d'outils ; la logique ci-dessous est notre code, parce que c'est là que se
jouent le coût et la fiabilité.

**1. Message reconstruit à chaque tour** (au lieu d'empiler la conversation) :

```
┌─ 1. Consignes + scénario complet          (identique à chaque tour)
├─ 2. Journal des actions déjà faites        (une ligne ajoutée par action)
│      Étape 1/4 ✓ Connexion — "connecté en tant que Marie"
│      Étape 2/4 : clic "Rechercher" → champ rempli "T-shirt bleu" → page /search
└─ 3. État ACTUEL de la page                 (seul bloc entièrement remplacé)
      + « Étape en cours : 2/4 — Ouvrir la fiche produit. Quelle action ? »
```

- Une seule version de la page est envoyée, pas les 30 précédentes : le coût suit le
  nombre d'actions au lieu d'exploser.
- Les blocs 1 et 2 ne font que s'allonger : ils profitent de la mise en cache des
  prompts des fournisseurs (tarif réduit sur un début de message déjà vu).
- Le journal sert aussi de « fil des actions » dans le rapport.
- C'est l'approche retenue par défaut ; le benchmark la compare à la conversation
  complète pour **chiffrer le gain** et vérifier qu'elle ne dégrade pas la fiabilité (§8).

**2. Outils donnés à l'IA** : ceux de Playwright MCP (cliquer, remplir, naviguer…), plus :
- `regarder_ecran` : capture d'écran **à la demande** seulement (une image coûte cher) ;
- `etape_reussie(resume)` : termine l'étape en cours ;
- `etape_impossible(raison, preuve)` : déclare l'étape irréalisable, **en citant un
  texte visible** dans la page (« Stock épuisé », « aucun bouton "Ajouter au panier" »).

**3. Règles appliquées par la boucle, pas par l'IA** :
- **Page stable avant lecture** : après chaque action, attendre la fin du chargement
  avant de lire la page (sinon l'IA lit une page à moitié chargée : cause classique
  d'instabilité).
- **Preuve vérifiée** : sur `etape_impossible`, la boucle vérifie que le texte cité
  existe dans la page. Trouvé → `failed`. Introuvable → `inconclusive`. Vérification
  sans appel LLM, donc gratuite.
- **Détection de boucle** : même action sur le même élément 3 fois, ou page inchangée
  après 4 actions → on arrête l'étape sans attendre la fin du budget.
- **Bilan forcé** : budget (`agent.maxActionsPerStep`) épuisé ou boucle détectée → un
  dernier appel court oblige l'IA à choisir entre « impossible, avec preuve » (→ règle
  de la preuve) et « je n'y arrive pas » (→ `inconclusive`).
- **Juge séparé** : les critères `expect` sont vérifiés par un appel distinct (voir
  « Juge » ci-dessous). L'agent qui a agi ne se note pas lui-même.

```ts
for (const [i, step] of scenario.steps.entries()) {
  for (let turn = 0; turn < config.agent.maxActionsPerStep; turn++) {
    const page = await stableSnapshot();
    const call = await llm(buildPrompt(scenario, journal, i, page),
                           { tools: { ...browserTools, regarder_ecran, etape_reussie, etape_impossible } });

    if (call.name === "etape_reussie") { journal.done(i, call.args.resume); break; }
    if (call.name === "etape_impossible")
      return pageContains(page, call.args.preuve) ? failed(i, call.args) : inconclusive(i, "preuve introuvable");

    const out = await execute(call, { injectSecrets: true });
    journal.add(i, describe(call, out));               // une ligne, secrets masqués
    if (isLooping(journal, i)) break;
  }
  if (!journal.isDone(i)) return await finalReview(i); // → failed (preuve trouvée) ou inconclusive
}
return judge(scenario.expect, await stableSnapshot(), await screenshot());
```

### Juge

Le juge décide si un critère `expect` est rempli. Ses deux erreurs possibles n'ont pas
le même coût : un **faux vert** (dit OK alors que le site est cassé) rend le test
inutile sans que personne ne s'en aperçoive ; un **faux rouge** fait perdre confiance.
L'objectif n°1 est **zéro faux vert**.

**Ce qu'il reçoit** : le texte de la page entière (pas seulement la partie visible),
une capture de la page entière, l'URL. **Pas le journal de l'agent** : il juge la page,
pas ce que l'agent affirme avoir fait.

**Nombre d'appels** : un appel par *moment* de vérification, pas par critère.
- Tous les `expect` de fin de scénario sont jugés **ensemble, en un seul appel** (même
  page, envoyée une fois), avec une réponse et une preuve par critère.
- Chaque `expect` attaché à une étape donne un appel, juste après `etape_reussie`. S'il est
  faux, le scénario s'arrête en `failed` à cette étape.
- `judge.votes: 3` multiplie le tout par 3.

**Réponse imposée, par critère** :

```jsonc
{
  "critere": "Le total affiché est 19,90 €",
  "verdict": "faux",                       // vrai | faux | indeterminable
  "valeur_attendue": "19,90 €",
  "valeur_observee": "24,90 €",
  "preuve": "Total TTC : 24,90 €",         // texte copié tel quel depuis la page
  "raison": "Le total inclut 5 € de frais de port"
}
```

**Contrôles faits par notre code, sans IA** :
- **La preuve existe dans la page** ; sinon → `inconclusive` (le juge a inventé).
- **Comparaison des valeurs** : si `valeur_attendue` et `valeur_observee` sont des nombres
  ou des montants, le code les normalise (« 19,90 € » = « 19.90€ » = « 19,9 EUR ») et les
  compare lui-même. Désaccord entre le code et le verdict du juge → `inconclusive`.
  Principe : l'IA **repère** l'information, le code **compare**.
- **Votes** (si `votes > 1`) : verdicts divergents → `inconclusive`.

**Correspondance** : vrai (preuve trouvée) → `passed` ; faux (preuve trouvée) → `failed` ;
indéterminable → `inconclusive`. La réponse « indéterminable » évite qu'un LLM forcé de
choisir réponde « vrai » par défaut. Les consignes du juge lui demandent aussi de
**chercher d'abord ce qui contredit le critère**, avant ce qui le confirme.

**Réglages** (`agentic-test.ts`) : `judge.model` (un modèle plus solide que l'agent,
abordable car le juge est appelé peu de fois) et `judge.votes`. Si le benchmark montre
que le regroupement des critères dégrade la fiabilité, on ajoutera
`judge.groupExpectations: false` (un appel par critère).

### Secrets jamais envoyés au LLM

Le LLM voit `{{secrets.TEST_USER_PASSWORD}}` (ou un placeholder équivalent) ; la valeur
réelle n'est injectée qu'au moment où l'action `fill` est exécutée par Playwright.
Stagehand propose un mécanisme de variables pour cela — **à vérifier** pour la version
retenue et pour Midscene pendant le benchmark ; à défaut, on l'implémente dans le driver.
Avec Playwright MCP, c'est forcément à nous : la boucle remplace le placeholder par la
valeur juste avant d'exécuter l'outil `fill`, et masque la valeur dans tout ce qui revient au LLM.

### Multi-provider LLM

Ne pas écrire de SDK maison. S'appuyer sur ce que supporte la lib d'agent retenue
(Stagehand et Midscene acceptent plusieurs providers — **liste exacte à vérifier**),
et utiliser le **Vercel AI SDK** pour l'appel `check` fait par `runner-core` ainsi que
pour la boucle du driver Playwright MCP (le SDK sait consommer les outils d'un serveur MCP).
Convention de config : `model: <provider>/<modèle>`, clé lue dans la variable d'env
standard du provider (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `MISTRAL_API_KEY`, …).
OpenRouter reste possible comme provider parmi d'autres, pas comme dépendance.

---

## 5. GitHub Action (J0)

```yaml
# .github/workflows/agentic-e2e.yml (côté client)
name: Agentic E2E
on: [pull_request]
permissions:
  contents: read
  pull-requests: write          # pour le commentaire
jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: <org>/agentic-e2e-action@v0
        with:
          base_url: ${{ vars.STAGING_URL }}
          # J1 : api_token: ${{ secrets.AGENTIC_TOKEN }}  → envoie aussi le rapport au SaaS
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          TEST_USER_EMAIL: ${{ secrets.TEST_USER_EMAIL }}
          TEST_USER_PASSWORD: ${{ secrets.TEST_USER_PASSWORD }}
```

Ce que fait l'Action :
1. installe Chromium (avec cache entre runs) ;
2. exécute les scénarios (parallélisme configurable, défaut 1 pour la lisibilité des coûts) ;
3. génère `report.json` + `report.html`, les publie en artefact du workflow ;
4. poste **ou met à jour** un commentaire unique sur la PR (tableau des scénarios, étape en
   échec, raison, lien vers l'artefact) ;
5. sort en code ≠ 0 selon les règles `ci` de `agentic-test.ts` (par défaut : au moins un `failed` ou un `inconclusive` après relance).

Exemple de commentaire PR :

> **Agentic E2E** — 4 scénarios · 3 ✅ · 1 ❌ · 0 🟠 · 0 ⚠️ · 2 min 41 · ~68k tokens
>
> | Scénario | Résultat | Détail |
> |---|---|---|
> | Achat d'un article | ❌ failed | Attente « Le total affiché est 19,90 € » : 24,90 € affiché |
> | Connexion | ✅ | |
>
> Rapport complet : artefact `agentic-report`

**Anti-support dès J0** : le premier check du runner est un « preflight » (clé LLM valide ?
`base_url` joignable ? scénarios valides selon le schéma ?) qui échoue avec un message
actionnable avant de lancer quoi que ce soit.

Le preflight **signale aussi les critères `expect` vagues**, qu'un juge ne peut pas
vérifier de façon fiable (« la page fonctionne bien », « l'expérience est fluide »).
Un appel LLM court, sans navigateur, vérifie que chaque critère décrit quelque chose
d'**observable sur la page**, et sinon affiche un avertissement avec une suggestion :
« Précisez ce qui doit être visible, par ex. "le message 'Commande confirmée' s'affiche" ».
Avertissement seulement : le run n'est pas bloqué. Le résultat est mis en cache par
`content_hash` du scénario pour ne pas repayer cet appel à chaque run.

---

## 6. Protocole runner ↔ control plane (J1–J2)

### Simplification clé pour le MVP

Avec une GitHub Action, **c'est la CI qui déclenche le runner**. Le runner n'a donc pas
besoin de poller une file de jobs : il *pousse* ses résultats. Le polling (et donc la
file de jobs côté plateforme) n'est nécessaire que pour les runners persistants
(Docker, hors MVP). Le bouton « lancer maintenant » de J2 peut passer par un
`workflow_dispatch` GitHub plutôt que par une file.

### Authentification

- Un **token runner par projet**, format `agt_<random>`, affiché une seule fois, stocké
  **haché** (SHA-256) côté plateforme. Révocable, plusieurs tokens actifs possibles (rotation).
- Header : `Authorization: Bearer agt_...`. Toutes les connexions sont **sortantes depuis le runner**, en HTTPS.

### Endpoints (v1)

| Méthode | Route | Rôle | Jalon |
|---|---|---|---|
| `POST` | `/api/v1/runs` | Ouvre un run (métadonnées git, runner, agent) → `run_id` | J1 |
| `POST` | `/api/v1/runs/:id/artifacts` | Demande des URLs d'upload présignées (R2) pour une liste de fichiers | J1 |
| `PUT` | *URL présignée R2* | Upload direct runner → R2 (la plateforme ne proxifie pas les octets) | J1 |
| `POST` | `/api/v1/runs/:id/complete` | Envoie `report.json` final, clôt le run | J1 |
| `GET` | `/api/v1/scenarios?ref=<branch>` | Récupère les scénarios gérés dans l'UI | J2 |

Règles :
- **Idempotence** : `POST /runs` accepte une clé `idempotency_key` (ex. `github.run_id` +
  `run_attempt`) pour qu'un job relancé ne crée pas de doublon.
- **Le SaaS n'est jamais bloquant** : si la plateforme est injoignable, l'Action logue un
  warning, garde le rapport HTML local et ne fait pas échouer la CI du client.
- Limites de taille par run (artefacts) côté plateforme, rétention 14 j par défaut.
- Versionner l'API (`/v1`) et `schema_version` du rapport dès le premier jour.

### Source de vérité des scénarios (J2)

À trancher en J2, proposition : **le repo reste la source de vérité par défaut** ;
l'UI permet d'éditer et de tester un scénario, puis de l'**exporter en PR** (ou de le
garder « géré par la plateforme » pour les utilisateurs non-dev). Le schéma est le même
dans les deux cas.

---

## 7. Control plane (J1)

**Stack** : Next.js (App Router) + TypeScript, Postgres + Drizzle, auth **GitHub OAuth**
(la cible est déjà sur GitHub ; pas de mot de passe à gérer), Cloudflare R2.
Hébergement : un VPS (Hetzner + Docker/Coolify) ou Vercel + Postgres managé — **à
trancher selon les tarifs du moment**, le code ne doit pas en dépendre.

### Modèle de données minimal

```
users            id, github_id, email, created_at
projects         id, owner_id, name, repo_full_name, created_at
project_members  project_id, user_id, role                 -- peut attendre J2
runner_tokens    id, project_id, name, token_hash, last_used_at, revoked_at
runs             id, project_id, idempotency_key, git_sha, git_ref, pr_number,
                 runner_kind, agent_driver, model, status, started_at, finished_at,
                 totals jsonb (passed/failed/error, tokens)
scenario_results id, run_id, file, content_hash, status, duration_ms, attempt,
                 failed_step_index, failure_reason, usage jsonb, detail jsonb
artifacts        id, run_id, scenario_result_id, kind, r2_key, size_bytes, expires_at
-- J2 :
scenarios        id, project_id, name, current_version_id
scenario_versions id, scenario_id, yaml, content_hash, created_by, created_at
```

`scenario_results.content_hash` + historique = détection de flaky plus tard (même
version de scénario, verdicts différents sur le même commit).

### Écrans J1
1. Connexion GitHub → liste des projets → créer un projet → générer un token (avec snippet YAML prêt à copier).
2. Liste des runs d'un projet (branche, PR, commit, verdict, durée, tokens).
3. Détail d'un run : scénarios, étape en échec mise en évidence, captures par étape, raison de l'échec, vidéo/trace téléchargeables.
4. Page « santé du runner » : dernier run reçu, version du runner, dernières erreurs `error` (anti-support).

---

## 8. Benchmark des libs d'agent (J-1)

**Candidats** (tous TypeScript, au-dessus de Playwright) :

| Option | Principe | Ce qu'on mesure en particulier |
|---|---|---|
| **Playwright MCP + boucle maison** (référence) | Le LLM reçoit les outils officiels de Playwright (cliquer, remplir, lire la page via l'arbre d'accessibilité) et décide lui-même de chaque action ; notre code fait tourner la boucle | Fiabilité « brute » d'un LLM avec les outils officiels ; coût en tokens (la structure de la page est renvoyée à chaque action) |
| **Stagehand** | La lib reçoit une instruction et choisit elle-même l'action | Ce qu'apporte une couche plus intelligente (cache, auto-réparation) |
| **Midscene** | Idem, approche par vision | Idem, et intérêt de la vision vs l'arbre d'accessibilité |

Pourquoi Playwright MCP sert de référence :
- outil officiel de l'équipe Playwright → pas de dépendance à une startup ou à un éditeur tiers ;
- fonctionne avec tout LLM qui sait appeler des outils → BYOK naturel ;
- chaque décision et chaque action sont visibles → rapports et diagnostic plus clairs ;
- c'est l'outil auquel les clients nous compareront (« Claude Code + Playwright MCP »).

Ce qu'il faut écrire soi-même avec cette option : la boucle d'agent (conception à détailler), la
gestion des secrets, et la récupération des actions effectuées pour le futur replay
(Playwright MCP semble afficher le code Playwright équivalent à chaque action — **à vérifier**).
À terme, on pourra aussi se passer du serveur MCP et appeler Playwright directement avec
nos propres définitions d'outils ; MCP est juste le moyen le plus rapide de tester l'approche.

**Règle de décision** : si Playwright MCP fait jeu égal en fiabilité et en coût, on le
retient (moins de dépendances, plus de contrôle).

browser-use (Python) est écarté du MVP pour ne pas avoir un runner dans un langage
différent du reste ; à reconsidérer seulement si les trois options échouent nettement.

**App cible** : une app de démo publique de type e-commerce (ex. SauceDemo) + une app
de démo locale contrôlée (pour pouvoir **introduire volontairement des bugs** et vérifier
que le test échoue — un agent qui passe toujours est pire que pas de test).

**Scénarios** :
1. Connexion + vérification du nom affiché (simple).
2. Ajout au panier + vérification du total (multi-étapes, assertion chiffrée).
3. Formulaire avec validation d'erreur (assertion négative).

**Protocole** : 10 exécutions par (lib × scénario × modèle), 2 modèles (un gros, un
petit/peu cher), sur l'app saine **et** sur l'app avec bug injecté.

**Mesures** :
| Mesure | Pourquoi |
|---|---|
| Taux de vrais positifs (passe sur app saine) | fiabilité |
| Taux de vrais négatifs (échoue sur app buguée) | le test sert à quelque chose |
| Variance entre les 10 runs | flakiness intrinsèque |
| Taux d'`inconclusive` | l'agent se perd-il souvent ? |
| Tokens : message reconstruit vs conversation complète (Playwright MCP) | valider le choix de la boucle (§4) |
| Durée médiane | expérience CI |
| Tokens / coût moyen par scénario | argument BYOK |
| Possibilité d'extraire les actions Playwright résolues | futur cache/replay |
| Support des variables/secrets non transmis au LLM | sécurité |

**Évaluation du juge, à part** : grâce à l'app où l'on injecte des bugs, constituer une
collection de cas (page, critère, bonne réponse connue) et mesurer :
- le **taux de faux verts** (objectif : zéro — c'est le chiffre le plus important du produit) ;
- le taux de faux rouges ;
- critères groupés en un appel vs un appel par critère ;
- l'apport réel de `judge.model` plus solide et de `judge.votes: 3`, rapporté à leur coût.

Le livrable est un tableau dans `docs/benchmark.md` et un choix argumenté.

---

## 9. Validation marché (en parallèle de J0)

- Landing page minimaliste (promesse : « vos tests E2E en langage naturel, votre clé IA, votre CI, vos données ») + liste d'attente.
- Publication de l'Action J0 (Marketplace GitHub, Show HN, Reddit r/QualityAssurance / r/webdev, communautés FR).
- 10 entretiens : agences web, PME avec app métier, freelances. Questions clés : comment testez-vous aujourd'hui ? qui écrit les tests ? combien paieriez-vous, 5–9 € ou 30–50 € ?
- **Signal pour lancer J1** (proposition) : ≥ 50 stars ou ≥ 20 repos utilisant l'Action, ou ≥ 3 équipes qui demandent un historique/dashboard.

---

## 10. Décisions à prendre par le porteur du projet

| # | Question | Proposition par défaut |
|---|---|---|
| 1 | Nom du produit (et donc préfixes `agentic`, `agt_`) | provisoire, à choisir avant publication de l'Action |
| 2 | Licence du runner | MIT ou Apache-2.0 (adoption max) ; le control plane reste fermé |
| 3 | Cible prioritaire | agences web (plusieurs projets clients, staging souvent derrière auth, sensibles au coût) — à confirmer par les entretiens |
| 4 | Paiement | merchant of record (Paddle / Lemon Squeezy) pour la TVA UE en solo |
| 5 | Hébergement | VPS si à l'aise avec l'ops, sinon Vercel + Postgres managé |

---

## 11. Prochaines actions concrètes

1. Initialiser le monorepo (pnpm, TS, lint, tests) avec `@agentic/scenario` (schéma Zod + tests).
2. Écrire les 3 scénarios de benchmark dans le format ci-dessus (ils servent de premiers fixtures).
3. Implémenter `driver-playwright-mcp` (avec sa boucle), `driver-stagehand` et `driver-midscene` + un `runner-core` minimal → lancer le benchmark.
4. Choisir la lib, puis construire `report-html` et la GitHub Action (J0).
5. En parallèle : landing page + premiers entretiens.

---

## 12. Après le MVP : diagnostic et correctif automatique en cas d'échec

Idée : quand un test échoue, proposer une **analyse du code** qui explique la cause
probable, puis, si l'utilisateur le demande, **une PR de correctif**.

### Pourquoi c'est cohérent avec le modèle BYOK + BYO compute

Le runner tourne déjà dans la CI du client, avec le code du repo sous la main
(`actions/checkout`) et la clé LLM du client. L'analyse peut donc se faire **chez
le client, sans que la plateforme voie jamais le code**. C'est un différenciateur fort
face aux SaaS qui devraient demander l'accès au code.

### Ne pas écrire notre propre agent de code

Comme pour le navigateur, on s'appuie sur un agent de code existant, lancé dans la CI
avec les mêmes contraintes multi-fournisseurs : par exemple un agent open source
multi-fournisseurs (type OpenCode ou Aider), ou le Claude Agent SDK pour les
utilisateurs de Claude. **Choix à faire par un mini-benchmark**, le jour venu. Notre
valeur ajoutée est **le dossier d'enquête** qu'on lui fournit, pas l'agent.

### Le dossier d'enquête fourni à l'agent de code

1. Le scénario, l'étape en échec et la raison (`report.json`).
2. Les `page_signals` : erreurs console, requêtes HTTP en échec, URL.
3. Les captures avant/après l'échec.
4. **Le diff de la PR testée** : quand le test passait sur la branche principale et
   échoue sur la PR, la cause est presque toujours dans ce diff. C'est l'indice n°1.
5. L'historique du scénario (il passait au commit X, échoue au commit Y).

### Étape 1 : trier avant de corriger

L'analyse commence par classer l'échec, car la bonne réponse n'est pas toujours
« corriger le code » :

| Cause | Exemple | Proposition |
|---|---|---|
| **Bug dans l'app** | Le total oublie la remise | Correctif du code |
| **Test obsolète** (changement voulu) | Le bouton « Valider » s'appelle désormais « Payer » | **Mise à jour du scénario**, pas du code |
| **Instabilité / agent perdu** | Rien dans le diff n'explique l'échec | Aucun correctif ; marquer comme suspect |
| **Environnement** | Staging en panne, données de test absentes | Aucun correctif ; message clair |

La deuxième ligne est très utile et peu risquée : c'est le premier correctif à proposer.

### Déroulé en trois niveaux (à livrer dans cet ordre)

1. **Diagnostic, en lecture seule.** Dans le commentaire de PR : la cause probable,
   les fichiers et lignes suspects, un diff proposé **en texte**. Aucun droit
   d'écriture nécessaire. Peu risqué, très démonstratif.
2. **PR de correctif, à la demande.** L'utilisateur répond `/agentic fix` dans la PR ;
   l'agent crée une branche et ouvre une PR **vers la branche de la PR en échec**
   (jamais vers `main`, jamais en poussant directement sur la branche du développeur).
3. **Correctif vérifié.** Le test qui échouait est relancé sur la PR de correctif ; le
   commentaire indique si le correctif le fait passer.

### Points techniques à ne pas oublier

- **Vérifier un correctif suppose de pouvoir lancer l'app corrigée.** Si les tests
  visent un staging fixe, le correctif ne peut pas être testé avant d'être déployé.
  Ça marche bien si le client a des **environnements de prévisualisation par PR**
  (Vercel, Netlify, Render…) ou si l'app peut être démarrée dans la CI (`base_url: http://localhost:3000`).
  Sinon, s'arrêter au niveau 2 et le dire clairement.
- **Limite GitHub** : une PR ouverte avec le `GITHUB_TOKEN` du workflow ne déclenche
  pas de nouveau workflow. Pour que les tests tournent sur la PR de correctif (niveau 3),
  il faudra une **App GitHub** (ou un token fourni par le client), ou un déclenchement
  explicite par `workflow_dispatch`.
- **Coût** : un agent de code consomme beaucoup plus de tokens qu'un test. Toujours
  **activé explicitement** par l'utilisateur, avec un plafond configurable et le coût
  affiché dans le commentaire.
- **Sécurité** :
  - job séparé du job de test, avec les **permissions minimales** (`contents: write` et
    `pull-requests: write` seulement au niveau 2) et sans les secrets de l'app testée ;
  - le contenu des pages testées est **non fiable** (texte saisi par des utilisateurs
    sur le staging, par exemple) : il ne doit jamais pouvoir faire exécuter des
    commandes à l'agent de code. On lui passe des données résumées, et on n'autorise
    ni l'accès réseau libre ni le push sans action humaine ;
  - désactivé par défaut sur les PR venant de forks.

### Place dans la feuille de route

Prévu **après J2**. Le niveau 1 (diagnostic) est assez simple pour être avancé si les
entretiens montrent que c'est un argument d'achat : il ferait une très bonne démo
(« le test échoue, et voici pourquoi, ligne 42 »). Ce qu'il faut faire **dès J0** :
capturer les `page_signals` et le contexte git dans `report.json` (§3).
