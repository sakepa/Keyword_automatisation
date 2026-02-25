# Plan d'Automatisation n8n - Keyword Automatisation

> Système complet d'automatisation SEO basé sur la méthodologie du Triangle Thématique
> et la simulation SGE (Search Generative Experience)

---

## Vue d'ensemble

Ce plan transforme le pipeline Python existant (Google Colab) en **5 workflows n8n**
orchestrés et interconnectés, couvrant l'intégralité du cycle de vie du contenu SEO :

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE DES WORKFLOWS                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐                       │
│  │   WF1    │   │   WF2    │   │   WF3    │                       │
│  │Recherche │──▶│Génération│──▶│  E-E-A-T │                       │
│  │Mots-clés │   │ Contenu  │   │   + FAQ  │                       │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘                       │
│       │              │              │                               │
│       ▼              ▼              ▼                               │
│  ┌──────────────────────────────────────┐                          │
│  │         Google Sheets                 │                          │
│  │    (Calendrier de Contenu)            │                          │
│  └──────────────┬───────────────────────┘                          │
│                 │                                                    │
│       ┌─────────┴─────────┐                                        │
│       ▼                   ▼                                        │
│  ┌──────────┐       ┌──────────┐                                   │
│  │   WF4    │       │   WF5    │                                   │
│  │Calendrier│       │Monitoring│                                   │
│  │ (Daily)  │       │ (Weekly) │                                   │
│  └──────────┘       └──────────┘                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## Contenu du dossier

| Fichier | Description |
|---------|-------------|
| `01_ANALYSE_QQOQCP.md` | Analyse complète QQOQCP (QUI, QUOI, OÙ, QUAND, COMMENT, COMBIEN, POURQUOI) |
| `02_GUIDE_DEPLOIEMENT.md` | Guide pas-à-pas pour installer n8n, configurer les credentials et importer les workflows |
| `03_SCRIPTS_JS_CUSTOM.md` | 7 scripts JavaScript documentés pour les nœuds Function n8n |
| `04_TESTS_ET_MONITORING.md` | Scénarios de test, plan de monitoring, checklist de validation |
| `workflows/WF1_recherche_mots_cles.json` | Workflow 1 : Recherche de mots-clés + extraction d'entités |
| `workflows/WF2_generation_contenu.json` | Workflow 2 : Génération de contenu en 5 étapes |
| `workflows/WF3_optimisation_eeat.json` | Workflow 3 : Optimisation E-E-A-T + génération FAQ |
| `workflows/WF4_calendrier_contenu.json` | Workflow 4 : Gestion du calendrier de contenu (quotidien) |
| `workflows/WF5_monitoring_performance.json` | Workflow 5 : Monitoring GSC + alertes (hebdomadaire) |

## Démarrage rapide

1. **Installer n8n** (voir `02_GUIDE_DEPLOIEMENT.md` pour les options)
2. **Configurer les credentials** (Google Cloud, OpenAI, Google Sheets)
3. **Créer le Google Sheet** avec les 3 onglets (Calendrier, Entités, Performance)
4. **Importer les 5 workflows JSON** dans n8n
5. **Tester WF1** avec un topic simple
6. **Activer les workflows** Cron (WF4, WF5)

## Technologies utilisées

| Composant | Usage |
|-----------|-------|
| **n8n** | Orchestration des workflows |
| **Gemini 1.5 Pro** (Vertex AI) | Génération de contenu, requêtes, FAQ |
| **OpenAI GPT-3.5 Turbo** | Extraction d'entités nommées |
| **Google Custom Search API** | Récupération des résultats de recherche |
| **Google Sheets** | Calendrier de contenu, stockage des données |
| **Google Drive** | Stockage des articles générés |
| **Gmail** | Notifications et rapports |
| **Google Search Console** | Monitoring de performance SEO |

## Coût estimé

| Service | Coût/mois |
|---------|----------|
| n8n (self-hosted) | $10-20 |
| APIs (Gemini + OpenAI + Search) | $20-60 |
| **Total** | **$30-80/mois** |

## Stratégie sous-jacente

Le système implémente la **Méthodologie du Triangle Thématique** :
- **BOFU** (Bottom-of-Funnel) : Contenu transactionnel pour revenus directs
- **TOFU** (Top-of-Funnel) : Contenu informationnel pour construire l'audience
- **Linkable Assets** : Contenu de référence pour gagner des backlinks

Organisé en **sprints de 120 jours** avec la "Ski Slope Strategy" :
priorité aux revenus (BOFU d'abord), puis audience (TOFU), puis autorité (LA).
