# docrag — chatbot RAG sur documentation technique (Mistral)

Un assistant qui répond aux questions sur **ta** documentation technique (Markdown, texte, PDF)
en citant ses sources. Il fonctionne avec l'**API Mistral** ou avec un **modèle local**
(Ollama, vLLM, llama.cpp…) sans changer une ligne de code.

```
 documents ──► chunker ──► embeddings ──► index (numpy)
 (.md/.txt/.pdf)  (par titres)  (mistral-embed)      │
                                                      ▼
 question ──► (réécriture si suivi) ──► embedding ──► top-k passages
                                                      │
                                     prompt = consignes + passages [1..k] + question
                                                      ▼
                                       LLM (mistral-small-latest) ──► réponse + sources
```

## Démarrage rapide

```bash
python -m venv .venv && source .venv/bin/activate      # Windows : .venv\Scripts\activate
pip install -e ".[dev]"

cp .env.example .env        # puis renseigne MISTRAL_API_KEY (clé sur console.mistral.ai)

docrag ingest               # indexe data/docs/
docrag ask "Que signifie le code erreur E-214 ?"
docrag chat                 # mode conversation avec mémoire
```

Le dossier `data/docs/` contient une documentation **fictive** (« SeaLink ») pour tester.
Pour utiliser tes documents : `docrag ingest --docs chemin/vers/tes/docs`
(ou `DOCS_DIR` dans `.env`).

Questions à essayer sur l'exemple :
- « Après combien de temps le système bascule-t-il sur la 4G ? »
- « Comment sauvegarder la configuration ? »
- « Quelle est la différence entre les classes C0 et C3 ? »
- « Quelle est la capitale de la France ? » → doit répondre qu'il ne trouve pas l'information.

## Utiliser un modèle local (Ollama)

```bash
ollama pull mistral && ollama pull bge-m3
```

Dans `.env` :

```
LLM_BASE_URL=http://localhost:11434/v1
CHAT_MODEL=mistral
EMBED_MODEL=bge-m3
```

Puis relance `docrag ingest` (l'index est lié au modèle d'embedding ; le programme refuse
de mélanger deux modèles). Aucune donnée ne quitte alors ta machine.

## Structure

```
src/docrag/
  config.py     réglages (variables d'environnement / .env)
  loader.py     lecture des fichiers .md .txt .rst .pdf
  chunker.py    découpage par titres Markdown, paragraphes, blocs de code préservés
  llm.py        client HTTP compatible OpenAI : embeddings, chat, streaming SSE, retries
  store.py      magasin de vecteurs numpy (similarité cosinus), sauvegarde JSON + .npy
  pipeline.py   ingestion, recherche, réécriture des questions de suivi, génération
  cli.py        commandes ingest / ask / chat
tests/          suite de tests, sans réseau ni clé API
```

## Choix de conception

- **Pas de framework (LangChain, etc.)** : moins de 800 lignes lisibles (commentaires compris), chaque étape du RAG est visible.
- **HTTP direct plutôt que SDK** : Mistral et Ollama exposent les mêmes endpoints
  (`/v1/chat/completions`, `/v1/embeddings`), donc un seul client pour le cloud et le local.
- **Chunking structurel** : découper par titres garde le contexte (« Architecture > Bascule »),
  et ce fil d'Ariane est ajouté au texte embeddé pour améliorer la recherche.
- **Anti-hallucination** : le prompt impose de répondre uniquement à partir des extraits,
  de citer `[n]`, et d'avouer quand l'information est absente. `MIN_SCORE` permet en plus de
  court-circuiter le LLM quand rien de pertinent n'est trouvé.
- **Questions de suivi** : « Et pour la 4G ? » est réécrite en question autonome avant la
  recherche, sinon la recherche vectorielle n'a aucun contexte.
- **Injection de prompt** : le contenu des documents est présenté comme de la *donnée*,
  et le prompt système demande d'ignorer les consignes qu'il contiendrait.
- **Robustesse** : retries avec backoff sur 429/5xx, erreurs claires (clé manquante,
  index absent, modèle d'embedding différent).

## Tests

```bash
pytest
```

Les tests couvrent le chunker, le store, le pipeline (avec un faux modèle déterministe) et
le client HTTP (contre un faux serveur compatible OpenAI, y compris streaming et retries).

## Limites et pistes d'amélioration

- Recherche **hybride** (BM25 + vecteurs) : meilleure sur les identifiants exacts (`E-214`).
- **Reranking** des passages avec un modèle dédié avant la génération.
- **Évaluation** : jeu de questions/réponses de référence, mesure du taux de bonnes sources
  retrouvées (hit rate @k) et de la fidélité des réponses.
- Index **incrémental** (ne ré-embedder que les fichiers modifiés) et base vectorielle
  (FAISS, Qdrant, pgvector) au-delà de quelques dizaines de milliers de passages.
- Exposer la recherche comme **serveur MCP** pour l'utiliser depuis un assistant de code.
- Ajouter une **transcription audio (ASR)** pour indexer des enregistrements de réunions.

## Git

Un commit par étape logique (voir `git log`). Ne commite jamais `.env` (déjà dans `.gitignore`).
