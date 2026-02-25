# Analyse QQOQCP : Automatisation du Système de Recherche de Mots-Clés et Création de Contenu SEO

> **Projet** : Keyword Automatisation - Data Marketing Labs
> **Méthodologie** : Triangle Thématique + SGE Simulation
> **Plateforme cible** : n8n (self-hosted ou cloud)

---

## Tableau Synthétique QQOQCP (1 page)

| Dimension | Synthèse | Détails clés |
|-----------|----------|-------------|
| **QUI ?** | Équipe marketing SEO (rédacteurs, analystes, responsables contenu) + APIs (Gemini, OpenAI, Google Search) | Rédacteurs BOFU/TOFU, Analyste SEO, Admin n8n, APIs tierces |
| **QUOI ?** | Automatiser le pipeline complet : recherche mots-clés → extraction entités → génération contenu → optimisation E-E-A-T → suivi performance | 7 workflows couvrant le cycle de vie complet du contenu |
| **OÙ ?** | n8n self-hosted/cloud + Google Sheets + Google Drive + Vertex AI + OpenAI API + Google Search API | Intégrations : Sheets, Drive, Gmail, Vertex AI, OpenAI, Custom Search |
| **QUAND ?** | Sprints de 120 jours ; recherche quotidienne ; génération à la demande ; monitoring hebdomadaire | Cron, webhooks, triggers Google Sheets |
| **COMMENT ?** | Pipeline orchestré par n8n : HTTP nodes, Function nodes (JS), Google Sheets nodes, conditionals, loops | 5 workflows JSON exportables avec scripts JS custom |
| **COMBIEN ?** | ~50-200 articles/sprint ; ~$50-200/mois APIs ; scalable jusqu'à 1000+ articles | Coûts détaillés par API et volume |
| **POURQUOI ?** | Réduire 80% du temps manuel, augmenter la qualité sémantique, accélérer l'indexation SGE | ROI : 10x en productivité, revenus prévisibles |

---

## QUI ? (Acteurs, rôles, utilisateurs, dépendances)

### Acteurs humains

| Rôle | Responsabilités | Interaction avec n8n |
|------|----------------|---------------------|
| **Responsable SEO / Stratège** | Définit les catégories de marque, valide le calendrier de contenu, analyse les SERP | Lance le Workflow 1 (recherche), valide les résultats |
| **Rédacteur de contenu** | Rédige et édite les articles BOFU/TOFU/LA | Reçoit les briefs générés, soumet les brouillons |
| **Analyste de données** | Surveille les performances GSC, analyse les entités | Consulte les dashboards, configure les alertes |
| **Admin n8n** | Maintient les workflows, gère les credentials, monitore les erreurs | Configure et déploie les workflows |

### Acteurs systèmes (APIs et Services)

| Service | Rôle dans le pipeline | Credentials requis |
|---------|----------------------|-------------------|
| **Google Vertex AI (Gemini 1.5 Pro)** | Génération de contenu, simulation SGE, topical maps | Service Account GCP (JSON) |
| **OpenAI API (GPT-3.5 Turbo / GPT-4)** | Extraction d'entités nommées, analyse sémantique | Clé API OpenAI |
| **Google Custom Search API** | Récupération des résultats de recherche, titres, snippets | Clé API + Search Engine ID |
| **Google Sheets** | Calendrier de contenu, stockage des résultats, base d'entités | OAuth2 Google |
| **Google Drive** | Stockage des brouillons générés | OAuth2 Google |
| **Gmail** | Notifications aux rédacteurs et analystes | OAuth2 Google |
| **Google Search Console** | Monitoring indexation et performances | OAuth2 Google |

### Dépendances techniques

- **n8n** >= v1.20 (self-hosted Docker ou n8n Cloud)
- **Node.js** >= 18 (pour les nœuds Function)
- Librairie `fuzzywuzzy` → réimplémentée en JS natif dans les nœuds Function
- Accès réseau aux APIs Google, OpenAI

---

## QUOI ? (Essence du concept/stratégie/modèle)

### Définition précise

Le projet **Keyword Automatisation** est un système d'automatisation end-to-end qui transpose le pipeline Python existant (Google Colab) en workflows n8n orchestrés. Il couvre :

1. **Recherche de mots-clés par IA** : Génération multi-requêtes → fusion données Google + LLM → extraction d'entités → analyse de fréquence
2. **Stratégie Triangle Thématique** : Classification BOFU/TOFU/Linkable Assets avec sprints de 120 jours
3. **Génération de contenu** : Pipeline en 4 étapes (brouillon → vérification factuelle → détection redondances → suppression fluff)
4. **Optimisation E-E-A-T** : Ajout automatique de métadonnées auteur, sources, FAQ
5. **Monitoring** : Suivi indexation GSC + alertes de performance

### Composants clés du pipeline

```
┌──────────────────────────────────────────────────────────────┐
│                    PIPELINE N8N COMPLET                        │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  WF1: RECHERCHE        WF2: GÉNÉRATION      WF3: OPTIMISATION │
│  ┌─────────────┐       ┌──────────────┐     ┌──────────────┐  │
│  │ Google Search│       │ Gemini Draft │     │ E-E-A-T Meta │  │
│  │ Multi-Query  │──────▶│ 4-Step Refine│────▶│ FAQ Generate │  │
│  │ Entity Freq  │       │ Topic Score  │     │ Source Cite  │  │
│  └─────────────┘       └──────────────┘     └──────────────┘  │
│         │                                          │           │
│         ▼                                          ▼           │
│  WF4: CALENDRIER       WF5: MONITORING                        │
│  ┌─────────────┐       ┌──────────────┐                       │
│  │ Sprint Plan │       │ GSC Tracking │                       │
│  │ BOFU/TOFU   │       │ Perf Alerts  │                       │
│  │ Sheets Sync │       │ Cost Monitor │                       │
│  └─────────────┘       └──────────────┘                       │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Résultats attendus et métriques de succès

| Métrique | Objectif | Méthode de mesure |
|----------|----------|-------------------|
| Temps de recherche mots-clés | Réduction de 80% (de 4h à 45min) | Comparaison avant/après |
| Entités extraites par sujet | >50 entités uniques | Compteur dans Workflow 1 |
| Topic Score Frase | ≥80% par article | Score sémantique |
| Taux d'indexation | <48h après publication | Google Search Console |
| Articles produits/sprint | 50-200 articles/120 jours | Calendrier de contenu |
| Coût par article | <$2 en API | Tracking des tokens |

---

## OÙ ? (Contextes, environnements, intégrations)

### Environnement de déploiement n8n

| Option | Configuration | Coût estimé |
|--------|--------------|-------------|
| **n8n Self-hosted (Docker)** | VPS 4GB RAM, 2 vCPU (ex: Hetzner, DigitalOcean) | ~$10-20/mois |
| **n8n Cloud Starter** | 5 workflows actifs, 2 500 exécutions/mois | $24/mois |
| **n8n Cloud Pro** | Workflows illimités, 10 000 exécutions/mois | $60/mois |

### Points d'intégration

```
                    ┌─────────────────┐
                    │     n8n          │
                    │  (Orchestrateur) │
                    └────────┬────────┘
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
    ┌─────▼─────┐     ┌─────▼─────┐     ┌─────▼─────┐
    │ ENTRÉES   │     │ TRAITEMENT│     │ SORTIES   │
    ├───────────┤     ├───────────┤     ├───────────┤
    │ Webhook   │     │ Gemini API│     │ G. Sheets │
    │ G. Sheets │     │ OpenAI API│     │ G. Drive  │
    │ Cron      │     │ G. Search │     │ Gmail     │
    │ Manuel    │     │ JS Custom │     │ GSC       │
    └───────────┘     └───────────┘     └───────────┘
```

### Flux de données

1. **Entrée** : Sujet/topic (string) → Google Sheets ou Webhook
2. **Recherche** : Topic → Multi-requêtes → Google Search API (titres, snippets) + Gemini (réponses LLM)
3. **Extraction** : Textes combinés → OpenAI GPT (entités nommées) → DataFrame
4. **Analyse** : Entités → Fuzzy matching → Fréquences → Top entités triées
5. **Génération** : Brief (type, titre, mots-clés, contexte) → Gemini → Brouillon 800+ mots
6. **Raffinement** : Brouillon → 4 passes LLM (factuel, redondances, fluff, structure)
7. **Sortie** : Article optimisé → Google Drive/Sheets + Notification Gmail

---

## QUAND ? (Timing, déclencheurs, fréquences)

### Matrice des déclencheurs

| Workflow | Déclencheur | Fréquence | Durée estimée |
|----------|------------|-----------|---------------|
| **WF1 : Recherche mots-clés** | Manuel ou Webhook | À la demande, 2-5x/semaine | 5-15 min/topic |
| **WF2 : Génération contenu** | Nouvelle ligne "En production" dans Sheets | À la demande | 3-8 min/article |
| **WF3 : Optimisation E-E-A-T** | Article généré (chaîné à WF2) | Automatique après WF2 | 1-2 min/article |
| **WF4 : Gestion calendrier** | Cron quotidien (08:00) | Quotidien | 30 sec |
| **WF5 : Monitoring & alertes** | Cron hebdomadaire (lundi 09:00) | Hebdomadaire | 2-5 min |

### Timeline d'un sprint type (120 jours)

```
Jour 1-7    : Configuration n8n + credentials + test workflows
Jour 8-14   : WF1 x20 topics → Calendrier rempli avec 50+ sujets validés
Jour 15-90  : WF2+WF3 en continu → 2-3 articles/jour générés et optimisés
Jour 30+    : WF5 activé → Monitoring hebdomadaire des performances
Jour 90-120 : Analyse des résultats, ajustement de la stratégie, nouveau sprint
```

---

## COMMENT ? (Mécanismes, processus, implémentation)

### Architecture des 5 Workflows n8n

#### Workflow 1 : Recherche de Mots-Clés et Extraction d'Entités

```
[Manual Trigger / Webhook]
        │
        ▼
[Set] topic, n_queries=10, n_responses=5, language="fr"
        │
        ▼
[HTTP Request → Gemini] Générer N requêtes liées au topic
        │
        ▼
[Function] Parser les requêtes en tableau
        │
        ▼
[SplitInBatches] Pour chaque requête :
    │
    ├──▶ [HTTP Request → Google Search API] Récupérer snippets
    │
    ├──▶ [HTTP Request → Gemini] Générer N réponses avec context snippets
    │
    └──▶ [HTTP Request → OpenAI] Extraire entités de chaque réponse
        │
        ▼
[Function] Fuzzy matching (seuil 90%) + comptage fréquences
        │
        ▼
[Sort] Trier par fréquence décroissante
        │
        ▼
[Google Sheets] Écrire les résultats (entité, mentions, prominence)
        │
        ▼
[Gmail] Notifier avec résumé des top 20 entités
```

#### Workflow 2 : Génération de Contenu (Pipeline 4 étapes)

```
[Google Sheets Trigger] Nouvelle ligne avec Statut="En production"
        │
        ▼
[Set] Extraire : TYPE, TITLE, GOAL, KEYWORDS, CONTEXT
        │
        ▼
[HTTP Request → Gemini] Étape 1 : Générer brouillon (800+ mots)
        │
        ▼
[HTTP Request → Gemini] Étape 2 : Vérification factuelle
        │
        ▼
[HTTP Request → Gemini] Étape 3 : Détection redondances
        │
        ▼
[HTTP Request → Gemini] Étape 4 : Suppression fluff
        │
        ▼
[HTTP Request → Gemini] Étape 5 : Restructuration paragraphes
        │
        ▼
[Google Drive] Créer document avec le contenu final
        │
        ▼
[Google Sheets] Mettre à jour Statut="Brouillon prêt" + lien Drive
        │
        ▼
[Gmail] Notifier le rédacteur assigné
```

#### Workflow 3 : Optimisation E-E-A-T

```
[Google Sheets Trigger] Statut passe à "En édition"
        │
        ▼
[Google Drive] Lire le contenu de l'article
        │
        ▼
[Function] Ajouter métadonnées E-E-A-T :
    ├── Bloc auteur (nom, bio, expertise)
    ├── Date de dernière mise à jour
    ├── Section citations/sources
    └── Bloc reviewer/fact-checker
        │
        ▼
[HTTP Request → Gemini] Générer section FAQ (5 questions)
        │
        ▼
[Function] Assembler article final avec E-E-A-T + FAQ
        │
        ▼
[Google Drive] Mettre à jour le document
        │
        ▼
[Google Sheets] Statut="Prêt à publier"
```

#### Workflow 4 : Gestion du Calendrier de Contenu

```
[Cron Trigger] Tous les jours à 08:00
        │
        ▼
[Google Sheets] Lire toutes les lignes du calendrier
        │
        ▼
[Function] Calculer :
    ├── Articles en retard (date publication < aujourd'hui & statut ≠ Publié)
    ├── Articles à produire cette semaine
    ├── Statistiques du sprint (% complété, vélocité)
    └── Prochaines deadlines
        │
        ▼
[IF] Articles en retard ?
    │
    ├── OUI → [Gmail] Alerte urgente au responsable + rédacteur
    │
    └── NON → [Continue]
        │
        ▼
[Gmail] Résumé quotidien du calendrier
```

#### Workflow 5 : Monitoring et Alertes de Performance

```
[Cron Trigger] Chaque lundi à 09:00
        │
        ▼
[HTTP Request → GSC API] Récupérer données des 7 derniers jours
    ├── Impressions, clics, position moyenne par URL
    └── Nouvelles pages indexées
        │
        ▼
[Google Sheets] Lire les URLs du calendrier avec statut "Publié"
        │
        ▼
[Merge] Croiser données GSC avec calendrier
        │
        ▼
[Function] Analyser :
    ├── Articles indexés vs non-indexés
    ├── Performance vs objectifs (position, CTR)
    ├── Tendances (amélioration/dégradation)
    └── Coûts API cumulés du sprint
        │
        ▼
[IF] Articles non-indexés après 7 jours ?
    │
    ├── OUI → [Gmail] Alerte avec recommandations
    │
    └── NON → [Continue]
        │
        ▼
[Gmail] Rapport hebdomadaire de performance
```

### Outils n8n recommandés par workflow

| Nœud n8n | Usage | Workflows |
|-----------|-------|-----------|
| **Manual Trigger** | Déclenchement à la demande | WF1 |
| **Webhook** | Réception de requêtes externes | WF1 |
| **Cron / Schedule Trigger** | Planification temporelle | WF4, WF5 |
| **Google Sheets Trigger** | Détection de changements | WF2, WF3 |
| **HTTP Request** | Appels API (Gemini, OpenAI, Google Search, GSC) | WF1-5 |
| **Function (Code)** | Logique JS custom (fuzzy matching, parsing, assemblage) | WF1-5 |
| **Google Sheets** | Lecture/écriture calendrier et résultats | WF1-5 |
| **Google Drive** | Création/lecture de documents | WF2, WF3 |
| **Gmail** | Notifications et rapports | WF1-5 |
| **IF** | Logique conditionnelle (retards, alertes) | WF4, WF5 |
| **SplitInBatches** | Itération sur les requêtes/entités | WF1 |
| **Merge** | Fusion de données (GSC + calendrier) | WF5 |
| **Set** | Définition de variables | WF1, WF2 |
| **Sort** | Tri des entités par fréquence | WF1 |

---

## COMBIEN ? (Quantités, coûts, scalabilité)

### Volumes de données estimés

| Métrique | Par exécution | Par sprint (120j) | Par an |
|----------|--------------|-------------------|--------|
| Topics recherchés (WF1) | 1 | 50-200 | 150-600 |
| Requêtes Gemini (WF1) | ~60 (10 queries x 5 gen + overhead) | 3 000-12 000 | 9 000-36 000 |
| Appels OpenAI (WF1) | ~50 (extraction entités) | 2 500-10 000 | 7 500-30 000 |
| Appels Google Search (WF1) | ~10 | 500-2 000 | 1 500-6 000 |
| Articles générés (WF2) | 1 | 50-200 | 150-600 |
| Appels Gemini (WF2) | 5 (par article) | 250-1 000 | 750-3 000 |

### Coûts estimés mensuels

| Service | Utilisation | Coût estimé/mois |
|---------|-----------|-----------------|
| **n8n Cloud Pro** | Workflows illimités | $60 |
| **n8n Self-hosted** (VPS) | Docker sur VPS | $10-20 |
| **Gemini 1.5 Pro** (Vertex AI) | ~5 000 requêtes/mois | $15-40 (input: $1.25/M tokens, output: $5/M) |
| **OpenAI GPT-3.5 Turbo** | ~3 000 requêtes/mois | $5-15 ($0.50/M input, $1.50/M output) |
| **Google Custom Search API** | ~600 requêtes/mois | $0-5 (100 gratuites/jour) |
| **Google Sheets/Drive/Gmail** | Inclus dans Google Workspace | $0 (si déjà abonné) |
| **TOTAL estimé** | | **$30-140/mois** |

### Seuils de scalabilité

| Seuil | Limitation | Solution |
|-------|-----------|----------|
| **100 requêtes/jour Google Search** | Quota gratuit API | Passer au plan payant ($5/1000 requêtes) |
| **60 requêtes/min Gemini** | Rate limit Vertex AI | SplitInBatches avec délai (Wait node) |
| **3 RPM OpenAI** (tier gratuit) | Rate limit tier 1 | Passer au tier 2+ ($50 crédits) |
| **5 workflows n8n Cloud Starter** | Limite du plan | Passer au plan Pro ou self-host |
| **Mémoire n8n** (gros datasets) | Items >1000 par exécution | Pagination avec SplitInBatches |

---

## POURQUOI ? (Raisons, bénéfices, risques, alternatives)

### Motivations et valeur ajoutée

1. **Productivité x10** : Le pipeline Python Colab actuel nécessite une exécution manuelle cellule par cellule. n8n automatise l'ensemble en un flux continu.
2. **Reproductibilité** : Chaque exécution produit des résultats cohérents et traçables.
3. **Collaboration** : Les rédacteurs reçoivent des briefs automatiques, les managers voient les dashboards en temps réel.
4. **Scalabilité** : Passer de 5 articles/semaine à 50+ sans effort humain supplémentaire.
5. **ROI mesurable** : Tracking des coûts API vs revenus générés par les articles BOFU.

### Risques et mitigations

| Risque | Impact | Probabilité | Mitigation |
|--------|--------|-------------|-----------|
| Rate limits API dépassés | Workflows bloqués | Moyenne | Retry avec backoff exponentiel, SplitInBatches avec Wait |
| Qualité contenu IA insuffisante | Contenu non publiable | Moyenne | Pipeline 4 étapes + revue humaine obligatoire |
| Coûts API imprévus | Dépassement budget | Faible | Monitoring des tokens dans WF5, alertes seuil |
| Changement API Google/OpenAI | Workflows cassés | Faible | Abstraction des appels HTTP, versioning des prompts |
| Données sensibles (clés API) | Fuite de credentials | Faible | Credentials n8n chiffrées, jamais en clair |
| Contenu dupliqué/plagiat | Pénalité Google | Faible | Vérification unicité dans le pipeline de raffinement |

### Alternatives considérées

| Solution | Avantages | Inconvénients | Verdict |
|----------|-----------|---------------|---------|
| **n8n** | Open-source, flexible, self-hosted, 400+ intégrations | Courbe d'apprentissage JS | **RETENU** |
| **Zapier** | Plus simple d'utilisation | Cher à grande échelle, moins flexible | Non retenu |
| **Make (Integromat)** | Bon rapport qualité/prix | Moins de contrôle sur le code custom | Alternative acceptable |
| **Python scripts (actuel)** | Déjà existant, contrôle total | Manuel, non collaboratif, non scalable | À migrer vers n8n |
| **LangChain/LangFlow** | Puissant pour LLM orchestration | Complexe, nécessite développeur | Complémentaire, pas un remplacement |
