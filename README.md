# agentic-test (nom provisoire)

Plateforme de **tests E2E agentiques** : les tests sont des scénarios en langage naturel,
exécutés par un agent IA qui pilote un vrai navigateur.

- **BYOK** : l'utilisateur apporte sa clé LLM (Claude, Gemini, Mistral, OpenAI…).
- **BYO compute** : les navigateurs tournent chez lui (GitHub Actions en premier).
- La plateforme fait l'orchestration, la gestion des scénarios et le reporting.

## Documents

- [docs/BRIEF.md](docs/BRIEF.md) — brief de reprise : concept, marché, risques, modèle économique.
- [docs/MVP.md](docs/MVP.md) — cadrage du MVP : jalons, format de scénario, format de rapport,
  architecture du runner, GitHub Action, protocole runner ↔ control plane, benchmark.

## Statut

Cadrage terminé, rien n'est codé. Prochaine étape : benchmark Stagehand vs Midscene
(voir `docs/MVP.md` §8 et §11).
