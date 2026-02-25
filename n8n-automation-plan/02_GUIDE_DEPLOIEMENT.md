# Guide de Déploiement n8n - Keyword Automatisation

> Ce guide couvre l'installation de n8n, l'import des 5 workflows, la configuration des credentials et les tests de validation.

---

## Table des matières

1. [Installation de n8n](#1-installation-de-n8n)
2. [Configuration des Credentials](#2-configuration-des-credentials)
3. [Préparation du Google Sheet](#3-préparation-du-google-sheet)
4. [Import des Workflows](#4-import-des-workflows)
5. [Configuration des Workflows](#5-configuration-des-workflows)
6. [Test de Validation](#6-test-de-validation)
7. [Mise en Production](#7-mise-en-production)

---

## 1. Installation de n8n

### Option A : n8n Cloud (Recommandé pour démarrer)

1. Créer un compte sur [n8n.io](https://n8n.io)
2. Choisir le plan **Pro** ($60/mois) pour les workflows illimités
3. L'interface est immédiatement accessible après inscription

### Option B : Self-hosted avec Docker (Recommandé pour la production)

```bash
# 1. Créer le répertoire de données
mkdir -p ~/.n8n

# 2. Lancer n8n avec Docker
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  -e N8N_SECURE_COOKIE=false \
  -e WEBHOOK_URL=https://votre-domaine.com/ \
  n8nio/n8n

# 3. Accéder à http://localhost:5678
```

### Option C : Docker Compose (Production avec PostgreSQL)

```yaml
# docker-compose.yml
version: '3.8'
services:
  n8n:
    image: n8nio/n8n
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=votre-mot-de-passe
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=n8n
      - WEBHOOK_URL=https://votre-domaine.com/
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      - postgres

  postgres:
    image: postgres:15
    restart: always
    environment:
      - POSTGRES_DB=n8n
      - POSTGRES_USER=n8n
      - POSTGRES_PASSWORD=n8n
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  n8n_data:
  postgres_data:
```

```bash
docker-compose up -d
```

---

## 2. Configuration des Credentials

Dans n8n, aller dans **Settings → Credentials** et configurer les credentials suivants :

### 2.1 Google Cloud / Vertex AI (pour Gemini)

| Paramètre | Valeur |
|-----------|--------|
| **Type** | Google Service Account |
| **Project ID** | `votre-project-id` (ex: `data-marketing-lab`) |
| **Region** | `us-central1` |
| **Service Account JSON** | Contenu du fichier JSON de votre service account |

**Comment obtenir :**
1. Aller dans [Google Cloud Console](https://console.cloud.google.com)
2. Créer un projet ou utiliser un existant
3. Activer l'API **Vertex AI**
4. Aller dans **IAM & Admin → Service Accounts**
5. Créer un service account avec le rôle **Vertex AI User**
6. Créer une clé JSON et la télécharger

### 2.2 OpenAI API

| Paramètre | Valeur |
|-----------|--------|
| **Type** | Header Auth ou OpenAI credential |
| **API Key** | `sk-...` |

**Comment obtenir :**
1. Aller sur [platform.openai.com](https://platform.openai.com)
2. **API Keys → Create new secret key**
3. Copier la clé (elle n'est montrée qu'une fois)
4. Ajouter des crédits ($10 minimum recommandé)

### 2.3 Google Custom Search API

| Paramètre | Valeur |
|-----------|--------|
| **API Key** | Clé API Google |
| **Search Engine ID** | ID du moteur de recherche personnalisé |

**Comment obtenir :**
1. Aller dans [Google Cloud Console → APIs](https://console.cloud.google.com/apis)
2. Activer **Custom Search API**
3. Créer une clé API dans **Credentials**
4. Aller sur [Programmable Search Engine](https://programmablesearchengine.google.com)
5. Créer un moteur de recherche (recherche sur tout le web)
6. Copier le **Search Engine ID**

### 2.4 Google Sheets / Drive / Gmail (OAuth2)

| Paramètre | Valeur |
|-----------|--------|
| **Type** | Google OAuth2 |
| **Client ID** | De votre projet GCP |
| **Client Secret** | De votre projet GCP |
| **Scopes** | Sheets, Drive, Gmail |

**Comment obtenir :**
1. Dans Google Cloud Console → **APIs & Services → OAuth consent screen**
2. Configurer l'écran de consentement (mode test)
3. **Credentials → Create OAuth Client ID → Web application**
4. Ajouter l'URL de callback de n8n : `https://votre-n8n.com/rest/oauth2-credential/callback`
5. Copier Client ID et Client Secret
6. Dans n8n, connecter le credential et autoriser l'accès

### 2.5 Google Search Console

| Paramètre | Valeur |
|-----------|--------|
| **Type** | Google OAuth2 (même que ci-dessus) |
| **Site URL** | `https://votre-site.com` |

**Pré-requis :** Votre site doit être vérifié dans Google Search Console.

---

## 3. Préparation du Google Sheet

### 3.1 Créer le Google Sheet "Calendrier de Contenu"

Créer un Google Sheet avec 3 onglets :

#### Onglet "Calendrier"

| URL du Sujet | Statut | Type | Propriétaire | Date de Publication | Volume | Traffic Potential | Minimum Authority Threshold | Number of Referring Domains | Keywords | Goal | Context | Lien Drive | E-E-A-T | FAQ |
|-------------|--------|------|-------------|-------------------|--------|------------------|---------------------------|---------------------------|----------|------|---------|-----------|---------|-----|
| best-tea-for-hangovers | Planifié | BOFU | jean@email.com | 2025-04-01 | 450 | 300 | 30 | 0 | best tea, hangover tea, herbal remedy | Guide readers to best tea choices | Research on herbal teas | | | |

**Statuts possibles :**
- `Planifié` → Article identifié, en attente de production
- `En production` → **Déclenche WF2** (génération de contenu)
- `Brouillon prêt` → Contenu IA généré, en attente de révision humaine
- `En édition` → **Déclenche WF3** (optimisation E-E-A-T)
- `Prêt à publier` → Article finalisé
- `Publié` → En ligne, suivi par **WF5**

#### Onglet "Entités"

| entity | totalMentions | queryCount | queries | prominence |
|--------|--------------|------------|---------|-----------|
| | | | | |

*Rempli automatiquement par WF1*

#### Onglet "Performance"

| Date | Total Clics | Total Impressions | Articles Indexés | Articles Non-Indexés |
|------|------------|-------------------|-----------------|---------------------|
| | | | | |

*Rempli automatiquement par WF5*

### 3.2 Noter l'ID du Google Sheet

L'ID se trouve dans l'URL : `https://docs.google.com/spreadsheets/d/`**`VOTRE_ID_ICI`**`/edit`

### 3.3 Créer un dossier Google Drive

Créer un dossier "Articles Générés" dans Google Drive et noter son ID (visible dans l'URL).

---

## 4. Import des Workflows

### Procédure d'import

Pour chaque fichier JSON dans le dossier `workflows/` :

1. Dans n8n, aller dans **Workflows → Add workflow**
2. Cliquer sur **⋮ (menu)** en haut à droite
3. Sélectionner **Import from file**
4. Sélectionner le fichier JSON correspondant
5. Le workflow s'affiche avec tous les nœuds

### Ordre d'import recommandé

1. **WF1_recherche_mots_cles.json** - Recherche de mots-clés
2. **WF4_calendrier_contenu.json** - Gestion calendrier (indépendant)
3. **WF2_generation_contenu.json** - Génération contenu
4. **WF3_optimisation_eeat.json** - Optimisation E-E-A-T
5. **WF5_monitoring_performance.json** - Monitoring

---

## 5. Configuration des Workflows

Après l'import, vous devez configurer chaque nœud avec vos credentials :

### Pour chaque workflow :

1. **Double-cliquer** sur chaque nœud HTTP Request
2. Dans **Authentication**, sélectionner le credential correspondant :
   - Nœuds Gemini → Google Service Account
   - Nœuds OpenAI → OpenAI API credential
   - Nœuds Google Search → Header Auth avec API Key
3. **Double-cliquer** sur chaque nœud Google Sheets/Drive/Gmail
4. Sélectionner le credential OAuth2 Google
5. Configurer le **Document ID** du Google Sheet

### Variables à personnaliser

Dans chaque workflow, les nœuds `Set` et `Code` contiennent des variables à adapter :

| Variable | Où la trouver | Valeur à mettre |
|----------|--------------|-----------------|
| `contentCalendarSheetId` | Nœuds Google Sheets | ID de votre Google Sheet |
| `contentDriveFolderId` | Nœud Google Drive (WF2) | ID du dossier Drive |
| `notificationEmail` | Nœuds Gmail | Votre email de notification |
| `vertexAiProject` | Nœuds HTTP (Gemini) | Votre Project ID GCP |
| `vertexAiRegion` | Nœuds HTTP (Gemini) | `us-central1` |
| `siteUrl` | WF5 (GSC) | URL de votre site |

---

## 6. Test de Validation

### Test WF1 : Recherche de Mots-Clés

1. Ouvrir WF1 dans l'éditeur
2. Cliquer sur **Test workflow**
3. Dans le nœud "Paramètres de Recherche", entrer :
   - `topic`: "meilleur thé pour la santé"
   - `n_queries`: 3 (réduit pour le test)
   - `n_responses`: 2
4. **Résultat attendu** :
   - Liste d'entités dans Google Sheets (onglet "Entités")
   - Email de notification reçu avec le résumé

### Test WF2 : Génération de Contenu

1. Dans le Google Sheet, ajouter une ligne :
   - URL du Sujet: `test-article`
   - Statut: `En production`
   - Type: `BOFU`
   - Propriétaire: `votre@email.com`
   - Keywords: `test keyword, example`
2. **Résultat attendu** :
   - Document créé dans Google Drive
   - Statut mis à jour dans Sheets → "Brouillon prêt"
   - Email reçu avec le lien du document

### Test WF3 : Optimisation E-E-A-T

1. Changer le statut de l'article test à `En édition`
2. **Résultat attendu** :
   - Document Drive mis à jour avec E-E-A-T + FAQ
   - Statut → "Prêt à publier"
   - Email de confirmation

### Test WF4 : Calendrier Quotidien

1. Exécuter manuellement le workflow
2. **Résultat attendu** :
   - Email résumé du calendrier reçu
   - Si articles en retard → alertes individuelles envoyées

### Test WF5 : Monitoring

1. Exécuter manuellement (nécessite des articles déjà publiés)
2. **Résultat attendu** :
   - Rapport hebdomadaire dans l'email
   - Nouvelle ligne dans l'onglet "Performance"
   - Alertes si articles non-indexés >7j

---

## 7. Mise en Production

### Checklist de déploiement

- [ ] n8n installé et accessible (cloud ou self-hosted)
- [ ] Tous les credentials configurés et testés
- [ ] Google Sheet créé avec les 3 onglets
- [ ] Dossier Google Drive créé
- [ ] WF1 importé et testé ✓
- [ ] WF2 importé et testé ✓
- [ ] WF3 importé et testé ✓
- [ ] WF4 importé et activé (cron quotidien) ✓
- [ ] WF5 importé et activé (cron hebdomadaire) ✓
- [ ] Emails de notification reçus correctement
- [ ] Rate limits vérifiés et délais configurés

### Activation des workflows

1. Pour chaque workflow, cliquer sur le toggle **Active** en haut à droite
2. Les workflows avec triggers Cron/Sheets commenceront automatiquement
3. WF1 reste en déclenchement manuel (ou via webhook)

### Monitoring de n8n

- Vérifier régulièrement **Executions** dans le menu n8n
- Configurer un **Error Workflow** pour recevoir les alertes d'erreur
- Surveiller l'utilisation mémoire si self-hosted

### Bonnes pratiques

1. **Commencer petit** : Tester avec 3 topics avant de lancer un sprint complet
2. **Revue humaine** : Toujours relire les articles générés avant publication
3. **Budget API** : Suivre les coûts dans les dashboards des providers
4. **Backups** : Exporter régulièrement les workflows (Settings → Export)
5. **Versioning** : Garder les fichiers JSON dans Git pour le suivi des modifications
