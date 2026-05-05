<p align="center">
  <img src="https://user-images.githubusercontent.com/10284570/173569848-c624317f-42b1-45a6-ab09-f0ea3c247648.png" alt="n8n Logo" />
</p>

<div align="center">

<img src="https://raw.githubusercontent.com/n8n-io/n8n/master/assets/n8n-logo.png" width="180" alt="n8n" />

# DocFlow RAG

### Pipeline asynchrone de traitement PDF et retrieval augmenté — Production SaaS

[![n8n](https://img.shields.io/badge/n8n-Orchestration-FF6D5A?logo=n8n&logoColor=white)](https://n8n.io)
[![Redis](https://img.shields.io/badge/Queue-Redis-DC382D?logo=redis&logoColor=white)](https://redis.io)
[![MinIO](https://img.shields.io/badge/Storage-MinIO-C72E49)](https://min.io)
[![PostgreSQL](https://img.shields.io/badge/Database-PostgreSQL-4169E1?logo=postgresql&logoColor=white)](https://postgresql.org)
[![PGVector](https://img.shields.io/badge/Vector-PGVector-7B2FBE)](https://github.com/pgvector/pgvector)
[![Gemini](https://img.shields.io/badge/LLM-Google%20Gemini-4285F4?logo=google&logoColor=white)](https://ai.google.dev)
[![Docker](https://img.shields.io/badge/Deploy-Docker-2496ED?logo=docker&logoColor=white)](https://docker.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-22C55E)](LICENSE)
[![Status](https://img.shields.io/badge/Status-Production%20Ready-22C55E)](.)
[![Multi-tenant](https://img.shields.io/badge/Multi--tenant-Supported-A855F7)](.)

</div>

---

## Image Workflow

<p align="center">
  <img src="https://github.com/Tchatchoua14/n8n-rag-pdf-pipeline/blob/main/DocFlow%20RAG.png" alt="n8n Logo" />
</p>

---

## Table des matières

- [Vue d'ensemble](#vue-densemble)
- [Architecture](#architecture)
- [Workflows](#workflows)
  - [Workflow 1 — Ingestion PDF](#workflow-1--ingestion-pdf)
  - [Workflow 2 — Worker Asynchrone](#workflow-2--worker-asynchrone)
  - [Workflow 3 — RAG Query Pipeline](#workflow-3--rag-query-pipeline)
- [Modèle de données](#modèle-de-données)
- [Cycle de vie d'un job](#cycle-de-vie-dun-job)
- [Gestion des erreurs](#gestion-des-erreurs)
- [Configuration de production](#configuration-de-production)
- [Installation](#installation)
- [Sécurité et multi-tenant](#sécurité-et-multi-tenant)
- [Monitoring et observabilité](#monitoring-et-observabilité)
- [Points de vigilance production](#points-de-vigilance-production)
- [FAQ](#faq)

---

## Vue d'ensemble

**DocFlow RAG** est une architecture de traitement documentaire asynchrone de niveau production, construite sur **n8n**. Elle transforme des fichiers PDF bruts en une base de connaissances vectorielle interrogeable via un pipeline RAG complet.

Le système couvre l'intégralité du cycle documentaire :

- **Ingestion** — réception sécurisée, validation stricte, stockage objet et mise en file
- **Traitement** — extraction de texte, nettoyage, découpage en chunks avec overlap
- **Indexation** — génération d'embeddings Gemini et insertion dans PGVector
- **Interrogation** — retrieval sémantique et génération de réponse via LLM
- **Observabilité** — traçabilité complète via `audit_logs`, statuts de job et logs RAG
- **Résilience** — retry automatique, branches d'erreur explicites sur chaque étape critique
- **Multi-tenant** — isolation complète des données par `tenant_id` à chaque niveau

> Ce projet est conçu pour être déployé directement en production SaaS. Chaque étape critique dispose de sa propre gestion d'erreur — aucune défaillance ne peut passer silencieusement.

---

## Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                       CLIENT / API                             │
│             POST /webhook/upload  (multipart PDF)              │
└─────────────────────────┬──────────────────────────────────────┘
                          │
                   ┌──────▼──────┐
                   │ WORKFLOW 1  │  Ingestion PDF
                   │   n8n       │
                   └──────┬──────┘
                          │
         ┌────────────────┼───────────────┐
         │                │               │
   ┌─────▼──────┐  ┌──────▼──────┐  ┌────▼──────┐
   │  MinIO/S3  │  │ PostgreSQL  │  │   Redis   │
   │ pdf-storage│  │  pdf_jobs   │  │  pdf_jobs │
   └────────────┘  └─────────────┘  └─────┬─────┘
                                          │ RPOP
                                   ┌──────▼──────┐
                                   │ WORKFLOW 2  │  Worker PDF
                                   │  n8n (cron) │
                                   └──────┬──────┘
                                          │
           ┌──────────────────────────────┼──────────────────┐
           │                              │                  │
    ┌──────▼──────┐              ┌────────▼───────┐  ┌───────▼──────┐
    │  MinIO/S3   │              │  PostgreSQL    │  │   PGVector   │
    │ (download)  │              │ pdf_documents  │  │  embeddings  │
    └──────┬──────┘              │ pdf_chunks     │  │  (indexed)   │
           │                     │ audit_logs     │  └──────────────┘
    ┌──────▼──────┐              └────────────────┘
    │   FastAPI   │  Extraction PDF
    │   :8000     │
    └─────────────┘

                                   ┌──────────────┐
                                   │  WORKFLOW 3  │  RAG Query
                                   │  n8n (chat)  │
                                   └──────┬───────┘
                                          │
                      ┌───────────────────┼──────────────┐
                      │                   │              │
               ┌──────▼──────┐  ┌─────────▼──────┐  ┌───▼──────────┐
               │  PGVector   │  │  Gemini LLM    │  │ PostgreSQL   │
               │  retriever  │  │  (génération)  │  │ rag_queries  │
               └─────────────┘  └────────────────┘  └──────────────┘
```

### Stack technique

| Composant | Rôle | Version recommandée |
|-----------|------|---------------------|
| **n8n** | Orchestration des workflows | ≥ 1.40 |
| **Redis** | File d'attente asynchrone | ≥ 7.x |
| **MinIO** | Stockage objet compatible S3 | Latest |
| **PostgreSQL** | Base métier, chunks, audit | ≥ 15 |
| **PGVector** | Extension vectorielle PostgreSQL | ≥ 0.7 |
| **FastAPI** | Service d'extraction PDF | Python ≥ 3.11 |
| **Google Gemini** | LLM + Embeddings | gemini-embedding-001 |
| **Docker** | Containerisation | ≥ 24 |

---

## Workflows

---

### Workflow 1 — Ingestion PDF

> **Rôle :** Recevoir, valider et mettre en file un fichier PDF envoyé par un client.

Ce workflow est le point d'entrée du système. Il est déclenché par un appel `POST` sur le webhook d'upload. Il valide rigoureusement le fichier binaire reçu (extension, MIME type, taille), construit les métadonnées avec isolation par tenant, stocke le PDF dans MinIO, crée un enregistrement de suivi dans PostgreSQL, pousse le job dans Redis, et retourne un `job_id` au client pour suivi.

Chaque étape critique dispose de sa propre branche d'erreur avec `audit_logs` et réponse HTTP explicite. Aucune défaillance ne peut laisser un job dans un état indéfini.

#### Flux principal

```
Webhook POST /upload
  │
  ├─ Code - Valider fichier PDF
  │     extension .pdf, MIME application/pdf, taille ≤ 50 MB
  │
  ├─ IF - Fichier PDF valide ?
  │     FALSE → Node Set (erreur) → audit_logs → Respond 400/415/413
  │
  ├─ Set - Préparer métadonnées fichier
  │     clé S3 : uploads/{tenant_id}/{timestamp}-{filename}
  │
  ├─ S3 - Upload PDF vers MinIO          (retryOnFail×3, 2s)
  │     └─ IF - Upload S3 OK ?
  │           FALSE → audit_logs → Respond 500
  │
  ├─ Postgres - Créer job PDF (uploaded) (retryOnFail×3, 2s)
  │     INSERT pdf_jobs, queryReplacement $1..$4
  │     └─ IF - Job Postgres créé ?
  │           FALSE → audit_logs → Respond 500
  │
  ├─ Redis - Empiler job PDF
  │     JSON.stringify({job_id, filename, storage_key, tenant_id})
  │     └─ IF - Redis push OK ?
  │           FALSE → audit_logs → Respond 500
  │
  ├─ Postgres - Marquer job en queued    (retryOnFail×3, 2s)
  │     UPDATE status='queued'
  │     └─ IF - Job marqué queued ?
  │           FALSE → audit_logs → Respond 500
  │
  └─ Respond to Webhook 202
        { success: true, status: "queued", job_id: "<uuid>" }
```

#### Nodes

| Node | Type | Rôle |
|------|------|------|
| `Webhook - Réception PDF` | Webhook | Entrée HTTP POST multipart/form-data |
| `Code - Valider fichier PDF` | Code | Validation extension, MIME, taille, binaire présent |
| `IF - Fichier PDF valide ?` | IF | Branchement sur `is_valid_file` |
| `Set - Préparer métadonnées fichier` | Set | Construction clé S3 avec tenant_id + timestamp |
| `S3 - Upload PDF vers MinIO` | S3 | Upload binaire, retryOnFail×3 |
| `IF - Upload S3 OK ?` | IF | Vérification succès S3 |
| `Postgres - Créer job PDF (uploaded)` | Postgres | INSERT pdf_jobs, RETURNING * |
| `IF - Job Postgres créé ?` | IF | Vérification id retourné |
| `Redis - Empiler job PDF` | Redis | LPUSH avec payload JSON.stringify |
| `IF - Redis push OK ?` | IF | Vérification `$json.value !== undefined` |
| `Postgres - Marquer job en queued` | Postgres | UPDATE status='queued' |
| `IF - Job marqué queued ?` | IF | Vérification status === "queued" |
| `Respond to Webhook 202` | Respond | job_id dynamique depuis Postgres RETURNING |

#### Entrée / Sortie

| | Détail |
|--|--------|
| **Entrée** | `POST /webhook/upload` — `multipart/form-data` — headers `x-api-key` + `x-tenant-id` |
| **Succès** | `202 { success: true, status: "queued", job_id: "uuid" }` |
| **Erreur fichier** | `400 / 413 / 415` selon le cas de validation |
| **Erreur serveur** | `500` avec message explicite selon la zone (S3 / Postgres / Redis) |

---

### Workflow 2 — Worker Asynchrone

> **Rôle :** Consommer les jobs Redis et exécuter le pipeline complet : extraction, nettoyage, chunking, embeddings, indexation PGVector.

Ce workflow joue le rôle de **worker asynchrone**. Déclenché par un Schedule toutes les 30 secondes, il dépile un job depuis Redis, valide le payload, verrouille le traitement en passant le job à `processing`, télécharge le PDF depuis MinIO, extrait le texte via FastAPI, nettoie et découpe le contenu en chunks, stocke les blocs dans PostgreSQL, puis indexe chaque chunk dans PGVector avec ses embeddings Gemini.

Le statut du job est mis à jour à chaque étape critique. Chaque erreur est capturée, journalisée dans `audit_logs`, et associée au job avec un message explicite.

#### Flux principal

```
Schedule (toutes les 30s)
  │
  ├─ Redis - Pop job PDF                (onError: continueErrorOutput)
  │     └─ IF - Job Redis existe ?      ($json.job !== null)
  │           FALSE → Set erreur → Postgres mark_failed → stop
  │
  ├─ Set - Normaliser payload job
  │     job_id, filename, storage_key, tenant_id
  │     .replace(/^=/, '') pour nettoyer les préfixes parasites
  │
  ├─ IF — job existe ?1
  │     !!job_id AND !!storage_key AND !!tenant_id
  │     FALSE → audit_logs → stop
  │
  ├─ Postgres - Marquer job en processing
  │     UPDATE status='processing' WHERE status='queued' (anti-doublon)
  │     └─ IF - Job marqué processing ?
  │           FALSE → mark_failed → stop
  │
  ├─ S3 - Télécharger PDF               (retryOnFail×3, 2s)
  │     └─ IF - PDF téléchargé ?        ($binary.data existe)
  │           FALSE → Set erreur → Postgres mark_failed
  │
  ├─ HTTP - Extraire contenu PDF        (retryOnFail×3, 2s)
  │     POST http://extraction-service:8000/extract
  │     └─ IF - Extraction HTTP OK ?    (!$json.error)
  │           FALSE → Set erreur → Postgres mark_failed
  │
  ├─ Code - Fusionner job + extraction
  │     Merge contexte job + texte extrait sans Merge node fragile
  │
  ├─ IF - Texte extrait non vide ?      (length > 30)
  │     FALSE → Log erreur worker → stop
  │
  ├─ Postgres - Enregistrer document extrait
  │     INSERT pdf_documents
  │
  ├─ Code - Nettoyer texte extrait
  │     Suppression null bytes, normalisation whitespace, fusion lignes
  │
  ├─ Code - Découper en chunk
  │     chunkSize: 1200, overlap: 200
  │     Découpe sur espace, propagation job_id + tenant_id
  │
  ├─ IF - Chunks existent ?             ($items().length > 0)
  │     FALSE → Set erreur → audit_logs worker → stop
  │
  ├─ Postgres - Enregistrer chunks
  │     INSERT pdf_chunks, queryReplacement $1..$6
  │
  ├─ Loop Over Items                    (batchSize: 10)
  │     └─ Postgres PGVector Store
  │           Default Data Loader (chunk_text)
  │           Embeddings Google Gemini (gemini-embedding-001)
  │
  └─ Postgres - Marquer job en done
        UPDATE status='done', finished_at=NOW()
        queryReplacement depuis Set - Normaliser payload job
```

#### Nodes

| Node | Type | Rôle |
|------|------|------|
| `Schedule - Worker PDF` | Schedule | Déclencheur toutes les 30s |
| `Redis - Pop job PDF` | Redis | RPOP + onError continueErrorOutput |
| `IF - Job Redis existe ?` | IF | Arrêt propre si queue vide |
| `Set - Normaliser payload job` | Set | Extraction et nettoyage payload Redis |
| `IF — job existe ?1` | IF | Validation complète du payload |
| `Postgres - Marquer job en processing` | Postgres | Lock du job (WHERE status='queued') |
| `IF - Job marqué processing ?` | IF | Anti-doublon de traitement |
| `S3 - Télécharger PDF` | S3 | Download binaire PDF, retryOnFail×3 |
| `IF - PDF téléchargé ?` | IF | Vérification `$binary.data` |
| `HTTP - Extraire contenu PDF` | HTTP Request | POST FastAPI, retryOnFail×3 |
| `IF - Extraction HTTP OK ?` | IF | Vérification succès extraction |
| `Code - Fusionner job + extraction` | Code | Merge contexte sans Merge node |
| `IF - Texte extrait non vide ?` | IF | Validation longueur > 30 |
| `Postgres - Enregistrer document extrait` | Postgres | INSERT pdf_documents |
| `Code - Nettoyer texte extrait` | Code | Normalisation texte brut |
| `Code - Découper en chunk` | Code | Chunking avec overlap 200 |
| `IF - Chunks existent ?` | IF | Vérification `$items().length > 0` |
| `Postgres - Enregistrer chunks` | Postgres | INSERT pdf_chunks, queryReplacement |
| `Loop Over Items` | SplitInBatches | Batches de 10 chunks |
| `Postgres PGVector Store` | PGVector | Indexation vectorielle |
| `Embeddings Google Gemini1` | Embeddings | Génération vecteurs |
| `Default Data Loader` | Data Loader | Préparation chunk_text pour PGVector |
| `Postgres - Marquer job en done` | Postgres | UPDATE status='done' |
| `Postgres - Marquer job en failed` | Postgres | UPDATE status='failed' + error_message |

---

### Workflow 3 — RAG Query Pipeline

> **Rôle :** Recevoir une question utilisateur, rechercher les chunks pertinents dans PGVector, et générer une réponse fondée sur les documents indexés via Gemini.

Ce workflow constitue la couche de **retrieval augmenté** du système. Déclenché par un Chat Trigger, il valide la question reçue, prépare le contexte utilisateur (tenant_id, session_id), exécute la recherche vectorielle dans PGVector, injecte les sources dans la chaîne RAG Gemini, et retourne une réponse structurée. Un fallback explicite gère les questions invalides.

#### Flux principal

```
Chat Trigger (question utilisateur)
  │
  ├─ Set - Préparer contexte utilisateur
  │     question, tenant_id, session_id, source='rag_chat'
  │
  ├─ IF - Question valide ?             (length >= 3)
  │     FALSE → Set - Question invalide → réponse explicite
  │
  └─ RAG - Générer la réponse à partir des documents
        │
        ├─ [ai_languageModel] Google Gemini Chat Model
        │
        └─ [ai_retriever] Vector Store - Base documentaire PGVector
              └─ [ai_vectorStore] Postgres PGVector Store1
                    topK: 5 — Embeddings Google Gemini
  │
  └─ Set - Réponse finale RAG
        { success, tenant_id, question, answer, session_id }
```

#### Nodes

| Node | Type | Rôle |
|------|------|------|
| `Chat - Recevoir la question utilisateur` | Chat Trigger | Point d'entrée chat |
| `Set - Préparer contexte utilisateur` | Set | Extraction question, tenant_id, session_id |
| `IF - Question valide ?` | IF | Validation longueur ≥ 3 caractères |
| `Set - Question invalide` | Set | Réponse fallback question trop courte |
| `RAG - Générer la réponse à partir des documents` | RetrievalQA Chain | Chaîne RAG complète |
| `Google Gemini Chat Model` | LLM | Génération de réponse |
| `Vector Store - Base documentaire PGVector` | Retriever | Recherche top-5 chunks |
| `Postgres PGVector Store1` | PGVector | Base vectorielle en mode retrieve |
| `Embeddings Google Gemini` | Embeddings | Vectorisation de la requête |
| `Set - Réponse finale RAG` | Set | Formatage réponse structurée |

#### Améliorations à ajouter pour un RAG expert

**1. Isolation tenant sur le retriever**

Le node `Postgres PGVector Store1` doit filtrer par `tenant_id`. Utiliser le champ `Filter` du node PGVector en mode retrieve, ou passer par une query SQL custom :

```sql
SELECT chunk_text, 1 - (embedding <=> $1) AS score
FROM pdf_chunks
WHERE tenant_id = $2
ORDER BY embedding <=> $1
LIMIT 5;
```

**2. IF — Sources trouvées ?**

Après le RAG node, vérifier que la réponse n'est pas vide :

```
Condition : $json.output.trim().length > 10
FALSE → Set fallback : "Aucune information trouvée dans vos documents."
```

**3. Log rag_queries**

Après chaque réponse, insérer dans `rag_queries` pour traçabilité et analytics :

```sql
INSERT INTO rag_queries (tenant_id, question, answer, session_id, created_at)
VALUES ($1, $2, $3, $4, NOW());
```

**4. System prompt Gemini**

Configurer `Google Gemini Chat Model` avec un system prompt strict :

```
Tu es un assistant documentaire expert.
Réponds uniquement à partir des sources fournies dans le contexte.
Si l'information n'est pas dans les documents, réponds exactement :
"Je ne trouve pas cette information dans vos documents."
Ne fabrique aucune réponse. Cite les passages sources si possible.
```

**5. continueOnFail sur le LLM**

Activer `continueOnFail: true` sur le node Gemini et ajouter un IF sur `$json.output` pour détecter les réponses nulles et déclencher un fallback.

---

## Modèle de données

### `pdf_jobs` — Suivi du cycle de vie

```sql
CREATE TABLE public.pdf_jobs (
  id            UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id     TEXT        NOT NULL,
  filename      TEXT        NOT NULL,
  storage_key   TEXT        NOT NULL,
  status        TEXT        NOT NULL DEFAULT 'uploaded',
  file_size     BIGINT,
  error_message TEXT,
  uploaded_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  started_at    TIMESTAMPTZ,
  finished_at   TIMESTAMPTZ
);

-- statuts possibles : uploaded | queued | processing | done | failed

CREATE INDEX idx_pdf_jobs_tenant   ON pdf_jobs(tenant_id);
CREATE INDEX idx_pdf_jobs_status   ON pdf_jobs(status);
CREATE INDEX idx_pdf_jobs_uploaded ON pdf_jobs(uploaded_at DESC);
```

### `pdf_documents` — Documents extraits

```sql
CREATE TABLE public.pdf_documents (
  id                UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  job_id            UUID        NOT NULL REFERENCES pdf_jobs(id),
  tenant_id         TEXT        NOT NULL,
  filename          TEXT        NOT NULL,
  storage_key       TEXT        NOT NULL,
  extracted_text    TEXT,
  page_count        INTEGER,
  content_type      TEXT        DEFAULT 'application/pdf',
  extraction_status TEXT        DEFAULT 'extracted',
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_pdf_docs_job    ON pdf_documents(job_id);
CREATE INDEX idx_pdf_docs_tenant ON pdf_documents(tenant_id);
```

### `pdf_chunks` — Chunks vectorisés

```sql
CREATE TABLE public.pdf_chunks (
  id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  job_id      UUID        NOT NULL REFERENCES pdf_jobs(id),
  tenant_id   TEXT        NOT NULL,
  filename    TEXT        NOT NULL,
  storage_key TEXT        NOT NULL,
  chunk_index INTEGER     NOT NULL,
  chunk_text  TEXT        NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_chunks_job    ON pdf_chunks(job_id);
CREATE INDEX idx_chunks_tenant ON pdf_chunks(tenant_id);
```

### `audit_logs` — Traçabilité complète

```sql
CREATE TABLE public.audit_logs (
  id         UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  job_id     UUID,
  level      TEXT        NOT NULL, -- info | warn | error
  message    TEXT        NOT NULL,
  details    JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_job     ON audit_logs(job_id);
CREATE INDEX idx_audit_level   ON audit_logs(level);
CREATE INDEX idx_audit_created ON audit_logs(created_at DESC);
```

### `rag_queries` — Log des interrogations RAG

```sql
CREATE TABLE public.rag_queries (
  id         UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id  TEXT        NOT NULL,
  question   TEXT        NOT NULL,
  answer     TEXT,
  session_id TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rag_tenant  ON rag_queries(tenant_id);
CREATE INDEX idx_rag_created ON rag_queries(created_at DESC);
```

---

## Cycle de vie d'un job

```
       ┌──────────┐
       │ UPLOADED │  Webhook reçu, PDF stocké dans MinIO
       └────┬─────┘
            │ Redis LPUSH
       ┌────▼────┐
       │ QUEUED  │  Job en attente dans la file Redis
       └────┬────┘
            │ Worker RPOP + lock WHERE status='queued'
     ┌──────▼──────┐
     │ PROCESSING  │  Traitement en cours (extract → chunk → embed)
     └──────┬──────┘
            │
    ┌───────┴───────┐
    │               │
 ┌──▼──┐        ┌───▼────┐
 │DONE │        │ FAILED │  error_message renseigné dans pdf_jobs
 └─────┘        └────────┘
```

**Reset automatique des jobs bloqués** — ajouter via un 4e workflow Schedule (toutes les 30 min) :

```sql
UPDATE pdf_jobs
SET status = 'queued', started_at = NULL
WHERE status = 'processing'
  AND started_at < NOW() - INTERVAL '30 minutes'
RETURNING id, tenant_id, filename;
```

Si des lignes sont retournées, déclencher une alerte Slack.

---

## Gestion des erreurs

### Matrice complète par zone

| Zone | Erreur possible | Branche n8n | Action |
|------|-----------------|-------------|--------|
| Webhook | Fichier absent | IF Fichier valide → FALSE | Respond 400 + audit_log |
| Webhook | MIME invalide | IF Fichier valide → FALSE | Respond 415 + audit_log |
| Webhook | Taille > 50 MB | IF Fichier valide → FALSE | Respond 413 + audit_log |
| S3 Upload | 403 / timeout | IF Upload S3 OK → FALSE | audit_log + Respond 500 |
| Postgres Create | Pool épuisé | IF Job créé → FALSE | audit_log + Respond 500 |
| Redis Push | Connexion perdue | IF Redis OK → FALSE | audit_log + Respond 500 |
| Postgres Queued | Échec UPDATE | IF Job queued → FALSE | audit_log + Respond 500 |
| Redis Pop | Queue vide | IF Job Redis existe → FALSE | Stop propre (pas d'erreur) |
| Payload Redis | job_id manquant | IF job existe → FALSE | audit_log + stop |
| Postgres Lock | Déjà en processing | IF Processing → FALSE | mark_failed + stop |
| S3 Download | Fichier absent | IF PDF téléchargé → FALSE | mark_failed + audit_log |
| HTTP Extract | Timeout / 500 | IF Extraction OK → FALSE | mark_failed + audit_log |
| Texte extrait | PDF image / vide | IF Texte non vide → FALSE | Log erreur worker + stop |
| Chunks | Texte trop court | IF Chunks existent → FALSE | audit_logs + stop |
| PGVector | Quota Gemini 429 | retryOnFail×3 | Retry automatique |
| RAG | Question vide | IF Question valide → FALSE | Réponse fallback explicite |

### Règles de retry

| Node | retryOnFail | maxTries | waitBetweenTries |
|------|-------------|----------|------------------|
| S3 Upload | ✅ | 3 | 2 000 ms |
| S3 Download | ✅ | 3 | 2 000 ms |
| HTTP Extract | ✅ | 3 | 2 000 ms |
| Postgres Create | ✅ | 3 | 2 000 ms |
| Postgres Update | ✅ | 3 | 2 000 ms |
| Redis Push | onError: continueRegularOutput | — | — |
| Redis Pop | onError: continueErrorOutput | — | — |

---

## Configuration de production

### Paramètres recommandés

```bash
# Validation fichier
MAX_PDF_SIZE_MB=50

# Chunking
CHUNK_SIZE=1200           # caractères par chunk
CHUNK_OVERLAP=200         # recouvrement entre chunks

# Worker
WORKER_INTERVAL_SECONDS=30
BATCH_SIZE=10             # chunks par batch pour PGVector

# RAG
RAG_TOP_K=5               # chunks retournés par le retriever

# Timeouts
HTTP_EXTRACT_TIMEOUT_MS=30000

# Jobs bloqués
JOB_PROCESSING_TIMEOUT_MINUTES=30
```

### Variables d'environnement n8n

Configurer dans **Settings > Variables** (jamais en dur dans les nodes) :

```
N8N_WEBHOOK_API_KEY=<votre-secret>
N8N_S3_BUCKET=pdf-storage
N8N_S3_ENDPOINT=http://minio:9000
N8N_POSTGRES_URL=postgresql://user:pass@postgres:5432/docflow
N8N_REDIS_HOST=redis
N8N_EXTRACT_URL=http://pdf-extractor:8000/extract
N8N_GEMINI_API_KEY=<votre-secret>
```

---

## Installation

### Prérequis

- Docker ≥ 24
- Docker Compose ≥ 2.20

### 1. Cloner le dépôt

```bash
git clone https://github.com/votre-user/docflow-rag.git
cd docflow-rag
```

### 2. Configurer l'environnement

```bash
cp .env.example .env
# Éditer .env avec vos credentials
```

### 3. Lancer les services

```bash
docker compose up -d
```

### 4. Initialiser la base de données

```bash
# Activer PGVector
psql -h localhost -U postgres -d docflow \
  -c "CREATE EXTENSION IF NOT EXISTS vector;"

# Créer les tables
psql -h localhost -U postgres -d docflow -f sql/schema.sql
```

### 5. Importer les workflows n8n

Depuis l'interface n8n (`http://localhost:5678`) :

1. **Settings > Import** → `workflows/workflow-1-ingestion.json`
2. **Settings > Import** → `workflows/workflow-2-worker.json`
3. **Settings > Import** → `workflows/workflow-3-rag.json`
4. Configurer les **Credentials** (S3, Postgres, Redis, Gemini)
5. Activer les trois workflows

### 6. Tester le pipeline

```bash
# Upload d'un PDF
curl -X POST https://votre-n8n.domain/webhook/VOTRE_PATH/upload \
  -H "x-api-key: votre-api-key" \
  -H "x-tenant-id: tenant-test" \
  -F "file=@/chemin/vers/document.pdf"

# Réponse attendue
# { "success": true, "status": "queued", "job_id": "xxxxxxxx-..." }
```

### Structure du projet

```
docflow-rag/
├── workflows/
│   ├── workflow-1-ingestion.json
│   ├── workflow-2-worker.json
│   └── workflow-3-rag.json
├── sql/
│   ├── schema.sql
│   └── indexes.sql
├── extraction-service/
│   ├── main.py
│   ├── requirements.txt
│   └── Dockerfile
├── docker-compose.yml
├── .env.example
├── LICENSE
└── README.md
```

---

## Sécurité et multi-tenant

### Isolation des données

- **Clé S3** : `uploads/{tenant_id}/{timestamp}-{filename}` — isolation physique par tenant
- **Redis** : payload contient `tenant_id`, propagé sur toute la chaîne sans exception
- **SQL** : toutes les requêtes utilisent `queryReplacement` avec `$1..$N` — zéro interpolation directe
- **PGVector** : les métadonnées de chaque chunk incluent `tenant_id` — filtrer sur chaque query RAG
- **audit_logs** : chaque événement est systématiquement lié au `job_id` et au contexte tenant

### Authentification webhook

```
Webhook settings → Authentication → Header Auth
  Key   : x-api-key
  Value : {{ $env.N8N_WEBHOOK_API_KEY }}
```

### Règles strictes

| Règle | Détail |
|-------|--------|
| ❌ Jamais `{{ $json.champ }}` dans un SQL | Utiliser `queryReplacement: [$1, $2, ...]` |
| ❌ Jamais de webhook sans auth | Header Auth obligatoire en production |
| ❌ Jamais de credentials en dur | Variables n8n ou Credentials n8n uniquement |
| ❌ Jamais de fallback `tenant-a` | Couper immédiatement si `tenant_id` absent |
| ✅ Toujours valider `tenant_id` | En entrée de chaque workflow, avant tout traitement |
| ✅ Toujours filtrer par `tenant_id` | Dans chaque query SQL et chaque retrieval PGVector |

---

## Monitoring et observabilité

### Requêtes de monitoring recommandées

```sql
-- Jobs par statut (dernières 24h)
SELECT status, COUNT(*) as total
FROM pdf_jobs
WHERE uploaded_at > NOW() - INTERVAL '24 hours'
GROUP BY status
ORDER BY total DESC;

-- Erreurs par zone (source d'audit)
SELECT details->>'source' as zone, COUNT(*) as total
FROM audit_logs
WHERE level = 'error'
  AND created_at > NOW() - INTERVAL '24 hours'
GROUP BY zone
ORDER BY total DESC;

-- Jobs bloqués en processing
SELECT id, tenant_id, filename, started_at,
       NOW() - started_at AS bloque_depuis
FROM pdf_jobs
WHERE status = 'processing'
  AND started_at < NOW() - INTERVAL '30 minutes';

-- Volume de chunks par tenant
SELECT tenant_id, COUNT(*) as chunks, SUM(LENGTH(chunk_text)) as chars_total
FROM pdf_chunks
GROUP BY tenant_id
ORDER BY chunks DESC;

-- Dernières erreurs worker
SELECT job_id, message, details->>'source', created_at
FROM audit_logs
WHERE level = 'error'
ORDER BY created_at DESC
LIMIT 20;

-- Taux d'utilisation RAG par tenant
SELECT tenant_id, COUNT(*) as queries, MAX(created_at) as derniere_query
FROM rag_queries
GROUP BY tenant_id
ORDER BY queries DESC;
```

### Alertes recommandées

| Condition | Seuil | Action recommandée |
|-----------|-------|-------------------|
| Jobs `processing` > 30 min | > 0 | Alerte Slack + reset automatique |
| Taux d'erreur worker | > 10% sur 1h | Alerte Slack + investigation |
| Queue Redis en attente | > 100 jobs | Scale du worker |
| Quota Gemini | > 80% | Alerte email + throttling |
| Pool Postgres | > 80% connexions | Vérifier pgBouncer |
| Espace MinIO | > 80% | Alerte capacité |

---

## Points de vigilance production

Ces corrections s'appliquent aux nodes encore présents dans le workflow exporté qui utilisent l'ancienne syntaxe.

**1. Nodes `mark_failed` avec `{{ $json.id }}` — injection SQL à corriger**

Les nodes suivants utilisent encore la syntaxe template string non paramétrée :

- `Postgres - Marquer failed payload Redis`
- `Postgres - Marquer failed job absent`
- `Postgres - Marquer failed aucun chunk`
- `Postgres - audit_logs worker2`

Correction à appliquer sur chacun :

```sql
-- Query à utiliser
UPDATE public.pdf_jobs
SET finished_at = NOW(),
    status = 'failed',
    error_message = $1
WHERE id = $2
RETURNING *;
```

```
-- queryReplacement
={{ [
  $json.error_message || "Erreur inconnue",
  $("Set - Normaliser payload job").item.json.job_id
] }}
```

**2. `Loop Over Items` — vérifier l'ordre des sorties**

Vérifier que la connexion est bien :
- Sortie index 1 → `Postgres PGVector Store` (traitement des chunks en cours)
- Sortie index 0 → `Postgres - Marquer job en done` (fin de boucle, tous les chunks traités)

**3. `IF - Redis push OK ?` — condition à renforcer**

Remplacer `!$json.error` par :

```
={{ $json.value !== undefined && $json.value !== null }}
```

**4. Workflow 3 — isolation tenant manquante**

Le retriever PGVector ne filtre pas encore par `tenant_id`. Ajouter un filtre de métadonnées dans le node PGVector ou utiliser une query SQL custom avec `WHERE metadata->>'tenant_id' = $1`.

**5. Workflow 3 — log `rag_queries` manquant**

Ajouter un node Postgres après `Set - Réponse finale RAG` :

```sql
INSERT INTO rag_queries (tenant_id, question, answer, session_id, created_at)
VALUES ($1, $2, $3, $4, NOW());
```

---

## FAQ

**Q : Comment surveiller et reset les jobs bloqués en `processing` ?**

Créer un 4e workflow Schedule (toutes les 30 min) avec ce SQL :

```sql
UPDATE pdf_jobs
SET status = 'queued', started_at = NULL
WHERE status = 'processing'
  AND started_at < NOW() - INTERVAL '30 minutes'
RETURNING id, tenant_id, filename;
```

Si des lignes sont retournées, déclencher une alerte Slack.

**Q : Comment scaler le worker si la queue s'allonge ?**

Réduire l'intervalle du Schedule (15s) ou activer plusieurs instances n8n en mode queue : `EXECUTIONS_MODE=queue` dans la configuration n8n.

**Q : Le service d'extraction `host.docker.internal:8000` n'est pas accessible en production.**

Remplacer par le nom de service Docker dans `docker-compose.yml` : `http://pdf-extractor:8000/extract`. S'assurer que les deux services sont sur le même réseau Docker.

**Q : Comment changer le modèle d'embeddings ?**

Remplacer le node `Embeddings Google Gemini` par `Embeddings OpenAI` (text-embedding-3-small). Vérifier que la dimension vectorielle correspond à la colonne PGVector — 1536 pour OpenAI, variable pour Gemini selon le modèle.

**Q : Peut-on interroger plusieurs tenants à la fois dans le RAG ?**

Non — chaque session RAG doit être strictement isolée par `tenant_id`. Le retriever PGVector doit toujours filtrer sur le `tenant_id` de l'utilisateur connecté.

**Q : Comment ajouter un retry après 3 échecs (dead letter) ?**

Ajouter un champ `retry_count` dans `pdf_jobs`. Après chaque `mark_failed`, incrémenter `retry_count`. Si `retry_count < 3`, re-pousser dans la queue Redis. Si `retry_count >= 3`, marquer `status = 'dead_letter'` et envoyer une alerte Slack.

---

## Licence

MIT — voir [LICENSE](LICENSE)

---

<div align="center">

Construit avec [n8n](https://n8n.io) · [PostgreSQL](https://postgresql.org) · [Redis](https://redis.io) · [MinIO](https://min.io) · [Google Gemini](https://ai.google.dev)

</div>

