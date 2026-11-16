# Architecture Technique — ASRP (Agentic Systematic Review Pipeline)

Ce document détaille l'architecture interne, les garanties de fiabilité et les métriques de référence du pipeline **ASRP**. Pour l'installation, la configuration des clés API et l'exécution, se référer au fichier [`README.md`](README.md).

## Flux d'exécution (Schéma détaillé)

Le graphe d'états coordonne 7 agents spécialisés, 2 points d'interruption humains (`INTERRUPT`) et une boucle de correction bornée :

```text
                    ┌─────────────┐
                    │   START     │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │   planner   │  protocole : objectif + critères
                    └──────┬──────┘
                           │
              ┌────────────▼─────────────┐
              │    human_protocol  ⏸     │  INTERRUPT #1 (validation)
              └────────────┬─────────────┘
                           │
                 ┌─────────▼──────────┐
                 │  search_strategy   │  3 à 5 requêtes en anglais
                 └─────────┬──────────┘
                           │
                 ┌─────────▼──────────┐
                 │     retrieval      │  OpenAlex + arXiv + index RAG
                 └─────────┬──────────┘
                           │
                 ┌─────────▼──────────┐
                 │     screening      │  Modèle A (strict) + Modèle B (inclusif)
                 └─────────┬──────────┘
                           │
              ┌────────────▼─────────────┐
              │   human_screening   ⏸    │  INTERRUPT #2 (arbitrage)
              └────────────┬─────────────┘
                           │
                 ┌─────────▼──────────┐
                 │     extraction     │  tableau structuré enrichi par RAG
                 └─────────┬──────────┘
                           │
                 ┌─────────▼──────────┐
                 │     synthesis      │  Markdown + Références (Python)
                 └─────────┬──────────┘
                           │
                 ┌─────────▼──────────┐       ┌─────┐
                 │       critic       │──────▶│ END │
                 └─────────┬──────────┘       └─────┘
                           │ (anomalies détectées, révisions < 2)
                           └──────► synthesis
```

## Garanties techniques et mécanismes internes

* **Références bibliographiques déterministes :** Contrairement au corps de la synthèse rédigé par le LLM, la section `References` finale est générée de manière déterministe en Python directement à partir des enregistrements OpenAlex et arXiv validés. Chaque citation est ainsi rattachée à un DOI ou identifiant réel, éliminant tout risque d'hallucination bibliographique.
* **Règles d'audit de l'agent `critic` :** Avant la clôture du graphe, le `critic` inspecte le brouillon produit par `synthesis` pour détecter trois types d'anomalies :
  1. Citations numériques `[n]` absentes du corpus inclus ;
  2. Divergence entre les comptages PRISMA annoncés et les données réelles du pipeline ;
  3. Écarts par rapport aux critères définis dans le protocole initial.
  La boucle de correction est strictement plafonnée (`revisions < 2`) pour garantir la terminaison du graphe.
* **Isolation des exécutions (`runs/<timestamp>/`) :** Chaque lancement horodaté isole ses artefacts et consigne l'ensemble des décisions et justifications des agents dans un journal structuré au format JSON Lines (`trace.jsonl`).

## Exemple de référence (Benchmark)

Performance mesurée lors d'une exécution complète de démonstration sur le palier gratuit de l'API Groq :

| Paramètre | Valeur observée |
| :--- | :--- |
| **Sujet de test** | *"Effects of coffee on cognitive performance"* |
| **Articles identifiés** | 20 enregistrements (OpenAlex + arXiv) |
| **Études retenues** | 5 études incluses après double screening et arbitrage |
| **Temps d'exécution total** | ~7 minutes (synthèse narrative complète + références vérifiables par DOI) |