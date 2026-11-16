# Pipeline de Revue Systématique de la Littérature (LangGraph & RAG Hybride)

Ce projet implémente un pipeline automatisé d'aide à la revue systématique de la littérature (conforme au cadre PRISMA) orchestré avec **LangGraph**. Il combine une collecte documentaire multi-sources, un moteur de recherche hybride local (dense + lexical), une double sélection par LLM avec mesure d'accord inter-juges, ainsi que des points de validation humaine (*Human-in-the-Loop*).

## Description du projet

L'objectif est d'automatiser les étapes répétitives d'une revue de littérature tout en maintenant un contrôle humain sur le protocole et la sélection finale des articles :

* **Planification et requêtes :** Génération structurée du protocole de recherche (questions PICO, critères d'inclusion/exclusion) et des requêtes booléennes.

* **Collecte et déduplication :** Récupération des métadonnées et résumés depuis des bases académiques ouvertes (OpenAlex, arXiv) avec filtrage et déduplication par DOI et titre normalisé.

* **Indexation RAG hybride locale :** Indexation des résumés retenus sans consommation de quota API externe, combinant des embeddings denses (`all-MiniLM-L6-v2` via ChromaDB) et une recherche lexicale (`BM25Okapi`).

* **Double screening croisé :** Évaluation indépendante des articles par deux modèles distincts (un profil strict et un profil inclusif) et calcul du coefficient d'accord $\kappa$ de Cohen.

* **Validation humaine (HITL) :** Deux interruptions explicites du graphe permettent de valider ou d'ajuster le protocole avant la recherche, puis de trancher sur les désaccords de sélection avant l'extraction.

* **Extraction et synthèse auto-critiquée :** Extraction structurée des données par article appuyée par le RAG hybride, suivie d'une rédaction contrôlée par une boucle itérative entre un agent rédacteur (`synthesis`) et un agent relecteur (`critic`).

* **Résilience API et traçabilité :** Gestion des limites de débit (*Token Bucket*, gestion des erreurs HTTP 429), cache local isolé par sujet (empreinte SHA-1) et journalisation complète des latences.

### Livrables générés

À l'issue de l'exécution, le pipeline produit un dossier de résultats (également exporté sous forme d'archive `.zip`) contenant :

* `report.md` : le rapport complet de la revue systématique au format Markdown.

* `extraction.csv` : le tableau structuré des données extraites pour chaque étude incluse.

* `prisma.json` : les comptages à chaque étape du flux PRISMA.

* `screening.json` : le détail des décisions des deux modèles et le calcul de concordance.

* `trace.jsonl` : le journal d'exécution et le profilage des temps de réponse par nœud.

## Architecture

Le flux d'exécution repose sur une machine à états **LangGraph** intégrant deux points d'arrêt humains et une boucle de révision qualité :

```
flowchart TD
    START([Début]) --> planner[planner<br/>Protocole & critères]
    planner --> human_protocol[[human_protocol<br/>Validation humaine HITL]]
    human_protocol --> search_strategy[search_strategy<br/>Requêtes booléennes]
    search_strategy --> retrieval[retrieval<br/>Collecte OpenAlex / arXiv<br/>& Indexation ChromaDB + BM25]
    retrieval --> screening[screening<br/>Double évaluation LLM A/B<br/>& Calcul kappa de Cohen]
    screening --> human_screening[[human_screening<br/>Arbitrage humain HITL]]
    human_screening --> extraction[extraction<br/>Extraction structurée + RAG]
    extraction --> synthesis[synthesis<br/>Rédaction du rapport]
    synthesis --> critic{critic<br/>Contrôle qualité}
    critic -- Révision requise --> synthesis
    critic -- Validé --> END([Fin & Export livrables])

```

### Composants principaux

| Fichier / Module | Rôle | 
| ----- | ----- | 
| `review_graph.py` | Définition de l'état du graphe, des nœuds LangGraph, du RAG hybride et de la gestion des quotas Groq. | 
| `*.ipynb` | Notebook Jupyter / Google Colab permettant d'exécuter le pipeline pas à pas et d'inspecter les métriques. | 
| `.cache/` | Stockage local des réponses API et des index vectoriels/BM25 par empreinte de sujet. | 

## Installation

Prérequis : **Python** $\ge$ **3.10**.

### 1. En local

```
git clone https://github.com/<votre-utilisateur>/<votre-depot>.git
cd <votre-depot>

python -m venv .venv
source .venv/bin/activate  # Sous Windows : .venv\Scripts\activate

pip install -U langgraph langchain-groq langchain-core chromadb sentence-transformers rank-bm25 requests pandas scikit-learn

```

### 2. Sur Google Colab

1. Ouvrez le notebook `.ipynb` directement dans Google Colab.

2. Exécutez la première cellule d'installation des dépendances (`pip install ...`) et placez `review_graph.py` dans le répertoire courant (`/content/`).

3. En fin d'exécution, téléchargez l'archive générée (`/content/projet_revue_systematique.zip`) depuis l'explorateur de fichiers Colab.

## Configuration

Le pipeline utilise l'API **Groq** pour l'inférence des modèles de langage. Les modèles d'embeddings (`all-MiniLM-L6-v2`) s'exécutent localement sur CPU/GPU sans clé externe.

### Variables d'environnement

| Variable | Obligatoire | Description | 
| ----- | ----- | ----- | 
| `GROQ_API_KEY` | Oui | Clé d'API obtenue sur [console.groq.com](https://console.groq.com). | 
| `MODEL_A` | Non | Modèle principal et screener strict (par défaut : `openai/gpt-oss-20b`). | 
| `MODEL_B` | Non | Second modèle pour le screening inclusif (par défaut : `qwen/qwen3-32b`). | 

### Définition des clés

* **En local (Linux/macOS) :**

  ```
  export GROQ_API_KEY="gsk_votre_cle_api"
  
  ```

* **En local (Windows PowerShell) :**

  ```
  $env:GROQ_API_KEY="gsk_votre_cle_api"
  
  ```

* **Sur Google Colab :**
  Ajoutez `GROQ_API_KEY` dans l'onglet **Secrets** (icône clé dans le panneau latéral gauche) et activez l'accès pour le notebook, ou définissez-la via `os.environ["GROQ_API_KEY"]`.

## Utilisation

### Exécution via le script Python

Lancez le pipeline depuis le terminal :

```
python review_graph.py

```

#### Déroulement de l'exécution

1. **Saisie du sujet :** Indiquez la question de recherche lorsque le script vous y invite (ou utilisez le sujet configuré par défaut dans `review_graph.py`).

2. **Checkpoint 1 (`human_protocol`) :** Le protocole généré s'affiche dans le terminal. Validez-le ou saisissez vos modifications pour orienter la recherche documentaire.

3. **Collecte et double screening :** Le script interroge les bases académiques, indexe les résumés en local et exécute les deux passes de sélection.

4. **Checkpoint 2 (`human_screening`) :** Examinez le résumé du screening et les éventuels désaccords entre les modèles A et B avant de lancer l'extraction.

5. **Génération finale :** Une fois la boucle `synthesis` / `critic` terminée, consultez le rapport dans le dossier de sortie (`output/report.md`) ainsi que les fichiers `extraction.csv` et `prisma.json`.

### Reprise après interruption (cache)

En cas de dépassement de quota journalier sur un modèle, les étapes déjà exécutées sont conservées dans le répertoire de cache local. Il suffit de relancer `python review_graph.py` (éventuellement en changeant `MODEL_A` ou `MODEL_B`) pour reprendre l'exécution sans réinterroger les API déjà sollicitées.