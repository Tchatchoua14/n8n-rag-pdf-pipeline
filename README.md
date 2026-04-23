<p align="center">
  <img src="https://user-images.githubusercontent.com/10284570/173569848-c624317f-42b1-45a6-ab09-f0ea3c247648.png" alt="n8n Logo" />
</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/n8n-io/n8n/master/assets/n8n-logo.png" width="280" alt="n8n Logo" />
</p>

<h1 align="center">Async PDF Processing & RAG Pipeline</h1>

<p align="center">
  Pipeline asynchrone de traitement de PDF avec n8n, Redis, MinIO, PostgreSQL, PGVector et extraction documentaire.
</p>

<p align="center">
  <img src="https://img.shields.io/badge/n8n-Workflow-ff6d5a" alt="n8n Workflow" />
  <img src="https://img.shields.io/badge/Architecture-Async-green" alt="Architecture Async" />
  <img src="https://img.shields.io/badge/Queue-Redis-red" alt="Redis" />
  <img src="https://img.shields.io/badge/Storage-MinIO-orange" alt="MinIO" />
  <img src="https://img.shields.io/badge/Database-PostgreSQL-blue" alt="PostgreSQL" />
  <img src="https://img.shields.io/badge/Vector-PGVector-purple" alt="PGVector" />
  <img src="https://img.shields.io/badge/License-MIT-lightgrey" alt="License MIT" />
</p>

---

## Overview

Ce projet fournit une architecture complète de traitement asynchrone de PDF construite avec **n8n**.

Le système permet de :

- recevoir un PDF depuis un formulaire ou un webhook
- stocker le fichier dans **MinIO**
- créer un suivi de traitement dans **PostgreSQL**
- empiler le travail dans **Redis**
- traiter le PDF dans un workflow worker séparé
- extraire le texte
- découper le contenu en chunks
- indexer les chunks dans **PGVector**
- interroger les documents via un pipeline **RAG**

L’objectif est d’obtenir une base robuste, modulaire et prête à être étendue.

---

## Architecture

Le système repose sur une architecture orientée événements :

- **n8n** : orchestration
- **Redis** : file d’attente
- **MinIO** : stockage objet compatible S3
- **PostgreSQL** : suivi métier et stockage documentaire
- **PGVector** : recherche vectorielle
- **Python / FastAPI** : extraction du texte PDF
- **LLM / Embeddings** : couche RAG

---

## Workflows

### Workflow 1 — PDF Ingestion

Ce workflow gère l’entrée des documents.

#### Étapes
- Réception du PDF
- Génération des métadonnées du fichier
- Upload dans MinIO
- Création d’un job dans PostgreSQL
- Push du job dans Redis
- Mise à jour du statut à `queued`

#### Nœuds recommandés
- `Webhook - Réception PDF`
- `Set - Préparer métadonnées fichier`
- `S3 - Upload PDF vers MinIO`
- `Postgres - Créer job PDF (uploaded)`
- `Redis - Empiler job PDF`
- `Postgres - Marquer job en queued`

---

### Workflow 2 — Async Worker

Ce workflow consomme les jobs et exécute le traitement documentaire.

#### Étapes
- Lecture d’un job depuis Redis
- Passage du job à `processing`
- Téléchargement du PDF depuis MinIO
- Extraction du texte
- Stockage du document extrait
- Nettoyage du texte
- Découpage en chunks
- Sauvegarde des chunks
- Génération des embeddings
- Indexation dans PGVector
- Mise à jour finale à `done` ou `failed`

#### Nœuds recommandés
- `Schedule - Déclencheur worker`
- `Redis - Récupérer job PDF`
- `Set - Normaliser payload job`
- `Postgres - Marquer job en processing`
- `S3 - Télécharger PDF`
- `HTTP - Extraire contenu PDF`
- `Merge - Fusionner job + extraction`
- `Postgres - Enregistrer document extrait`
- `Code - Nettoyer texte extrait`
- `Code - Découper en chunks`
- `Postgres - Enregistrer chunks`
- `Default Data Loader - Préparer chunks pour vector store`
- `Embeddings - Générer vecteurs`
- `PGVector - Indexer chunks`
- `Postgres - Marquer job en done`
- `Postgres - Marquer job en failed`

---

### Workflow 3 — RAG Query Pipeline

Ce workflow permet d’interroger les documents déjà indexés.

#### Étapes
- Réception d’une question utilisateur
- Recherche des chunks les plus pertinents dans PGVector
- Injection des chunks dans la chaîne RAG
- Génération d’une réponse finale par le modèle

#### Nœuds recommandés
- `Chat - Recevoir la question utilisateur`
- `Retriever - Rechercher les chunks pertinents`
- `Vector Store - Base documentaire PGVector`
- `RAG - Générer la réponse à partir des documents`
- `LLM - Modèle de réponse`

---

## Data Model

### `pdf_jobs`
Table de suivi du cycle de vie de chaque PDF.

Champs typiques :
- `id`
- `filename`
- `storage_key`
- `status`
- `tenant_id`
- `uploaded_at`
- `started_at`
- `finished_at`
- `error_message`
- `file_size`

---

### `pdf_documents`
Table de stockage du document extrait.

Champs typiques :
- `id`
- `job_id`
- `tenant_id`
- `filename`
- `storage_key`
- `extracted_text`
- `page_count`
- `content_type`
- `extraction_status`

---

### `pdf_chunks`
Table des chunks préparés pour l’indexation.

Champs typiques :
- `id`
- `job_id`
- `document_id`
- `tenant_id`
- `filename`
- `storage_key`
- `chunk_index`
- `chunk_text`
- `created_at`

---

## Job Lifecycle

Les jobs suivent un cycle simple :

```text
uploaded → queued → processing → done
```

## En cas d’erreur :

```text
uploaded → queued → processing → failed
```

## `Setup`

### Prérequis
1. Docker
2. n8n
3. Redis
4. PostgreSQL
5. MinIO
6. PostgreSQL avec PGVector
7. Service d’extraction PDF

## Example Architecture Flow

```text
[PDF Upload]
   ↓
[MinIO]
   ↓
[PostgreSQL: pdf_jobs]
   ↓
[Redis Queue]
   ↓
[Worker Workflow]
   ↓
[PDF Extraction]
   ↓
[PostgreSQL: pdf_documents]
   ↓
[Chunking]
   ↓
[PostgreSQL: pdf_chunks]
   ↓
[Embeddings]
   ↓
[PGVector]
   ↓
[RAG Query Workflow]

```

## Image Workflow

<p align="center">
  <img src="https://github.com/Tchatchoua14/n8n-rag-pdf-pipeline/blob/main/End-to-End%20Async%20PDF%20Processing.png" alt="n8n Logo" />
</p>

