# Projet : plateforme SaaS de tests E2E agentiques (BYOK + BYO compute)

> Brief de reprise. Résume une discussion exploratoire (octobre 2026).
> Statut : idée validée sur le papier, rien n'est codé. Prochaine étape : cadrer le MVP
> (voir [MVP.md](./MVP.md)).

## 1. Le concept

Une plateforme web permettant de lancer des **tests E2E agentiques** sur une app web :
les tests sont une **liste de prompts en langage naturel** ("Se connecter avec X, ajouter un produit au panier, vérifier que le total est correct"), exécutés par un agent IA qui pilote un vrai navigateur.

Particularité du modèle :

- **BYOK (Bring Your Own Key)** : chaque utilisateur fournit sa propre clé API LLM (Claude, Gemini, Mistral, OpenAI…). Il gère et paie ses coûts IA directement.
- **BYO compute** : l'utilisateur fournit l'infra qui exécute les navigateurs (runner chez lui).
- **La plateforme ne gère que** : l'orchestration des runs, la gestion des scénarios, le reporting.

### Positionnement retenu

**Produit indie pas cher**, pas une startup à lever des fonds. Objectif : couvrir l'hébergement + une marge.
Le modèle BYOK + BYO compute supprime les deux seuls postes de coût qui pourraient exploser (tokens et flotte de navigateurs) → coûts très prévisibles.

## 2. État du marché (recherche faite)

### Open source / auto-hébergé avec BYOK

| Outil | Type | Notes |
|---|---|---|
| **SkyTest Agent** (oursky) | Plateforme avec UI (Next.js + Midscene + Playwright) | Le plus proche du concept. Clé OpenRouter perso. Mélange étapes IA + code Playwright. |
| **QA Agent** (jimmytoan) | Plateforme (FastAPI + React + browser-use) | Suites, tests, captures, GIFs, historique. Multi-provider. |
| **AWT (AI Watch Tester)** | Outil + cloud bêta | BYOK (OpenAI/Anthropic/Ollama). Génère des scénarios depuis URL/spec. Cloud = plutôt une démo. |
| **e2e** (TesterArmy) | Framework TS | Agent + record/replay sans appel LLM tant que l'UI ne change pas. Jeune (pré-1.0). |
| **Midscene.js** (ByteDance) | Lib au-dessus de Playwright | Vision, cache de planification. |
| **Magnitude** | Framework de test | Vision, double agent (planner + executor). |
| **Stagehand** (Browserbase) | SDK navigateur IA | Le plus mature, mais pas un framework de test. |
| **browser-use**, **Skyvern** | Agents navigateur | Généralistes. |

### SaaS existants

- **Aucun SaaS de test clé en main avec BYOK sérieux** trouvé. TesterArmy, QA Wolf, Momentic, testRigor, Testsigma incluent le modèle dans leur prix (ex. TesterArmy : 99 $/mois pour 250 runs).
- SaaS d'**infra navigateur** BYOK (Browserbase, Scrapfly) : on paie l'exécution, pas l'IA — mais ce n'est pas une plateforme de test (pas de gestion de scénarios ni de reporting).

### Précédent du modèle "control plane"

**Currents.dev**, ancien **Cypress Dashboard**, **Buildkite**, runners self-hosted GitHub Actions : le client exécute chez lui, la plateforme vend orchestration + reporting. Modèle éprouvé, mais pas encore appliqué au test agentique.

→ **Il y a un trou dans le marché** : SaaS de test agentique avec BYOK + BYO compute.

## 3. Arguments forts

1. **Coûts maîtrisés** : pas de risque sur les tokens ni sur le compute.
2. **Confidentialité / souveraineté** (meilleur argument commercial) :
   - le runner tourne chez le client → peut tester un staging derrière firewall, les données ne sortent pas ;
   - **la clé API peut rester dans l'env du runner** : la plateforme ne voit jamais ni les clés ni le trafic LLM ;
   - possibilité d'utiliser Mistral → argument RGPD / Europe face aux SaaS US.

## 4. Risques identifiés

1. **Capture de valeur** : si le client apporte IA + compute, la plateforme doit apporter plus qu'un outil OSS gratuit ou que Claude Code + Playwright MCP. Leviers : gestion/versioning des scénarios, rapports exploitables, intégration PR, détection de tests instables, **cache/replay** (réduit la facture de tokens du client = ROI mesurable).
2. **Friction d'onboarding** : "ta clé + ton infra" = beaucoup de config avant le 1er test.
   → **Parade : GitHub Actions (puis GitLab CI) comme premier runner.** Le client a déjà ce compute ; installation = un fichier YAML + un secret.
3. **Le produit, c'est la qualité de l'agent** : ne pas réécrire l'agent, construire sur Stagehand / Midscene / browser-use et se concentrer sur la fiabilité (assertions, retries, cache, prompts de vérification).
4. **Marché qui bouge vite** (TesterArmy YC, outils de code agentiques qui intègrent le test navigateur).
5. **Coût caché principal = le temps de support** ("mon runner ne se connecte pas", "l'agent ne clique pas"). À minimiser dès la conception.
6. **Distribution** : le plus dur ne sera pas la rentabilité par client mais se faire connaître.

## 5. Architecture envisagée

### Control plane (la plateforme)
- App **Next.js / TypeScript**
- **Postgres**
- File de jobs
- Stockage artefacts : **Cloudflare R2** (pas de frais de sortie), rétention courte (7–14 j), option "envoyer vers le bucket S3 du client"

### Runner (chez le client)
- CLI npm / image Docker / **GitHub Action** (prioritaire)
- **Connexion sortante uniquement** (polling) : jamais de connexion entrante chez le client
- Récupère les scénarios → exécute (Playwright + lib d'agent) → renvoie verdicts, captures, vidéos, traces
- Clé LLM lue depuis l'env du runner (jamais stockée côté plateforme)

### Points d'attention
- Les vidéos/traces Playwright pèsent lourd → le stockage est le poste à surveiller.
- Un token de runner par projet.

## 6. Modèle économique

- **Hébergement estimé** : ~10–30 €/mois tant qu'il y a < quelques centaines d'utilisateurs actifs (VPS Hetzner 5–15 €/mois, ou Vercel + Postgres managé en free tier au début). *À re-vérifier avec les tarifs actuels.*
- **Prix envisagé** : 5–9 €/mois par compte ou petit forfait par projet ; **pousser l'annuel** (~60 €/an) car les frais fixes par transaction pèsent sur les petits montants.
- **Paiement** : Stripe, ou un *merchant of record* (Paddle, Lemon Squeezy) pour déléguer la TVA européenne — commission plus élevée mais beaucoup moins de gestion en solo.
- **Acquisition** : runner open source et/ou plan gratuit limité.
- **Évolution possible** : option "runners managés" payante pour ceux qui ne veulent rien gérer.

## 7. Périmètre MVP

**Dans le MVP :**
- Comptes + projets
- Édition des scénarios en langage naturel
- Token de runner par projet
- Runner sous forme de **GitHub Action**
- Historique des runs : verdict, captures, étape en échec
- Commentaire automatique sur les PR

**Plus tard :**
- Cache / replay sans appel LLM
- Analyse d'instabilité (flaky tests)
- Runner Docker générique, GitLab CI
- Runners managés
- Export artefacts vers bucket client

**Jalon zéro possible :** GitHub Action seule, open source, qui publie un rapport HTML en artefact → mesurer l'intérêt avant de construire la couche SaaS.

## 8. Questions ouvertes

- Quelle lib d'agent sous le capot : Stagehand, Midscene, browser-use ? (à benchmarker sur 2–3 scénarios réels)
- Format des scénarios : texte libre, YAML, Markdown structuré (étapes + critères de validation) ?
- Comment gérer les identifiants de test de l'app cible (secrets côté runner aussi ?)
- Abstraction multi-provider LLM : SDK maison, Vercel AI SDK, OpenRouter ?
- Nom du produit, cible prioritaire (agences web ? PME avec app métier ? devs freelances ?)
- Validation marché : interroger ~10 équipes — paieraient-elles 30–50 €/mois… ou 5–9 € ?

## 9. Prochaines étapes suggérées

1. Benchmarker 2 libs d'agent sur une app de démo (fiabilité, vitesse, coût en tokens).
2. Définir le format de scénario.
3. Prototyper la GitHub Action (jalon zéro).
4. Concevoir le protocole runner ↔ control plane (polling, auth par token, upload d'artefacts).
5. Landing page + collecte d'intérêt en parallèle.
