# Scénarios de Test & Plan de Monitoring

> Document définissant les tests de validation pour chaque workflow et le plan de surveillance en production.

---

## 1. Scénarios de Test par Workflow

### WF1 : Recherche de Mots-Clés & Extraction d'Entités

#### Test 1.1 : Exécution minimale

| Paramètre | Valeur |
|-----------|--------|
| **Topic** | `meilleur thé vert` |
| **n_queries** | 3 |
| **n_responses** | 2 |
| **search_limit** | 3 |
| **language** | `fr` |

**Résultats attendus :**
- [ ] Gemini retourne exactement 3 requêtes liées
- [ ] Google Search retourne des résultats pour chaque requête
- [ ] OpenAI extrait au moins 5 entités par réponse
- [ ] Le fuzzy matching réduit les doublons (ex: "thé vert" et "the vert")
- [ ] Les résultats sont écrits dans Google Sheets (onglet "Entités")
- [ ] Un email de notification est reçu avec le top 20

**Critères de succès :**
- Minimum 15 entités uniques extraites
- Aucune erreur dans les logs d'exécution
- Temps d'exécution < 5 minutes

#### Test 1.2 : Gestion d'erreurs API

| Scénario | Action | Résultat attendu |
|----------|--------|-----------------|
| Clé API Google invalide | Mettre une mauvaise clé | Retry 3x puis erreur claire |
| Rate limit OpenAI | Envoyer 20 requêtes simultanées | Backoff exponentiel, toutes traitées |
| Topic vide | Envoyer `""` comme topic | Erreur propre, pas de crash |
| Gemini timeout | Simuler avec un timeout court | Retry automatique (3 tentatives) |

#### Test 1.3 : Volume

| Paramètre | Valeur |
|-----------|--------|
| **n_queries** | 10 |
| **n_responses** | 5 |

**Résultats attendus :**
- [ ] 50+ entités uniques extraites
- [ ] Temps d'exécution < 15 minutes
- [ ] Mémoire n8n stable (pas de memory leak)

---

### WF2 : Génération de Contenu

#### Test 2.1 : Article BOFU

| Champ | Valeur |
|-------|--------|
| URL du Sujet | `best-herbal-tea-for-sleep` |
| Statut | `En production` |
| Type | `BOFU` |
| Propriétaire | `test@email.com` |
| Keywords | `herbal tea, sleep, chamomile, lavender` |
| Goal | `Help readers choose the best herbal tea for better sleep` |

**Résultats attendus :**
- [ ] Le trigger détecte la nouvelle ligne dans les 5 minutes
- [ ] Le brouillon contient ≥800 mots
- [ ] Structure H1/H2 présente
- [ ] Les 4 étapes de raffinement s'exécutent séquentiellement
- [ ] Un document Google Drive est créé dans le bon dossier
- [ ] Le statut passe à "Brouillon prêt" dans Sheets
- [ ] Email de notification envoyé au propriétaire

#### Test 2.2 : Article TOFU

| Champ | Valeur |
|-------|--------|
| Type | `TOFU` |
| URL du Sujet | `how-to-steep-tea-perfectly` |

**Vérifie que le system prompt change pour le mode éducatif/informationnel.**

#### Test 2.3 : Linkable Asset

| Champ | Valeur |
|-------|--------|
| Type | `Linkable Asset` |
| URL du Sujet | `tea-consumption-statistics-2025` |

**Vérifie que le system prompt oriente vers un contenu de référence/données.**

---

### WF3 : Optimisation E-E-A-T

#### Test 3.1 : Ajout E-E-A-T complet

**Pré-requis :** Un article avec statut "Brouillon prêt" et un lien Drive valide.

1. Changer le statut à `En édition`
2. **Résultats attendus :**
   - [ ] Article lu depuis Google Drive
   - [ ] Bloc auteur ajouté en en-tête
   - [ ] Date "Dernière mise à jour" = aujourd'hui
   - [ ] Section "Sources et Références" ajoutée en pied
   - [ ] 5 questions FAQ générées et pertinentes
   - [ ] Document Drive mis à jour
   - [ ] Statut → "Prêt à publier"
   - [ ] Colonnes E-E-A-T et FAQ → "Oui"
   - [ ] Email de confirmation reçu

#### Test 3.2 : FAQ pertinentes

**Vérifier manuellement que les 5 FAQ :**
- [ ] Sont en rapport avec le sujet de l'article
- [ ] Ne répètent pas le contenu existant
- [ ] Ajoutent de la valeur informative
- [ ] Sont dans la bonne langue

---

### WF4 : Gestion du Calendrier

#### Test 4.1 : Détection de retards

**Setup :** Ajouter dans le calendrier :
- 1 article avec Date de Publication = hier, Statut = "Planifié" (EN RETARD)
- 1 article avec Date de Publication = dans 3 jours, Statut = "En production"
- 1 article avec Date de Publication = il y a 1 semaine, Statut = "Publié" (OK)

**Résultats attendus :**
- [ ] Le rapport identifie 1 article en retard
- [ ] L'alerte urgente est envoyée au propriétaire de l'article en retard
- [ ] L'article dans 3 jours apparaît dans "Cette semaine"
- [ ] L'article publié est compté comme "Publié" dans les stats
- [ ] Le résumé quotidien est envoyé avec les bonnes métriques

#### Test 4.2 : Sprint vide

**Setup :** Calendrier sans aucune ligne.

**Résultat attendu :** Email avec "0 articles" et pas d'erreur.

---

### WF5 : Monitoring Performance

#### Test 5.1 : Avec données GSC

**Pré-requis :** Au moins 1 article publié et indexé depuis >7 jours.

**Résultats attendus :**
- [ ] Données GSC récupérées correctement
- [ ] Croisement avec le calendrier effectué
- [ ] Rapport avec clics, impressions, CTR, position
- [ ] Top 10 articles listés
- [ ] Données loggées dans l'onglet "Performance"

#### Test 5.2 : Détection non-indexation

**Setup :** 1 article avec statut "Publié" mais URL inexistante dans GSC.

**Résultats attendus :**
- [ ] Article identifié comme "non-indexé"
- [ ] Si publié >7 jours → alerte URGENT envoyée
- [ ] Recommandations incluses dans l'alerte

---

## 2. Matrice de Tests d'Intégration

| Test | WF1 → | WF2 → | WF3 → | WF4 → | WF5 |
|------|-------|-------|-------|-------|-----|
| Recherche → Sheets → Génération | Rechercher un topic | Le brief utilise les entités extraites | - | - | - |
| Génération → E-E-A-T → Publication | - | Générer article | Optimiser E-E-A-T | Détecter dans calendrier | Monitorer après publication |
| Sprint complet (120j simulé) | 5 topics | 5 articles | 5 optimisations | Stats du sprint | Rapport hebdo |

---

## 3. Plan de Monitoring en Production

### 3.1 Alertes d'erreur n8n

Configurer un **Error Workflow** dans n8n :

```json
{
  "name": "Error Handler - Alertes",
  "nodes": [
    {
      "parameters": {
        "errorTriggerType": "allWorkflows"
      },
      "name": "Error Trigger",
      "type": "n8n-nodes-base.errorTrigger",
      "position": [240, 300]
    },
    {
      "parameters": {
        "sendTo": "admin@votredomaine.com",
        "subject": "={{ '[n8n ERROR] ' + $json.workflow.name + ' - ' + $json.execution.error.message }}",
        "message": "={{ 'Workflow: ' + $json.workflow.name + '\\nExecution ID: ' + $json.execution.id + '\\nErreur: ' + $json.execution.error.message + '\\nNœud: ' + $json.execution.error.node + '\\nTimestamp: ' + $json.execution.error.timestamp }}"
      },
      "name": "Gmail - Alerte Erreur",
      "type": "n8n-nodes-base.gmail",
      "position": [460, 300]
    }
  ],
  "connections": {
    "Error Trigger": {
      "main": [[{"node": "Gmail - Alerte Erreur", "type": "main", "index": 0}]]
    }
  }
}
```

### 3.2 Métriques à surveiller

| Métrique | Seuil d'alerte | Fréquence de vérification |
|----------|---------------|--------------------------|
| Taux d'erreur workflows | >5% des exécutions | Quotidien |
| Temps d'exécution WF1 | >20 minutes | Par exécution |
| Temps d'exécution WF2 | >10 minutes | Par exécution |
| Coûts API Gemini | >$50/mois | Hebdomadaire |
| Coûts API OpenAI | >$20/mois | Hebdomadaire |
| Articles non-indexés >14j | >3 articles | Hebdomadaire |
| Mémoire n8n (self-hosted) | >80% RAM | Quotidien |

### 3.3 Dashboard de suivi

Créer un onglet "Dashboard" dans le Google Sheet avec :

| KPI | Formule | Source |
|-----|---------|--------|
| Articles publiés ce sprint | `=COUNTIF(Calendrier!B:B, "Publié")` | Onglet Calendrier |
| Vélocité (%) | `=publiés/total*100` | Onglet Calendrier |
| Clics totaux (dernier rapport) | `=INDEX(Performance!B:B, MATCH(MAX(Performance!A:A), Performance!A:A, 0))` | Onglet Performance |
| Entités totales extraites | `=COUNTA(Entités!A:A)-1` | Onglet Entités |
| Articles en retard | `=COUNTIFS(Calendrier!E:E, "<"&TODAY(), Calendrier!B:B, "<>Publié")` | Onglet Calendrier |

### 3.4 Procédure de reprise après erreur

| Situation | Action |
|-----------|--------|
| WF1 échoue au milieu | Réexécuter avec les mêmes paramètres (idempotent) |
| WF2 échoue après brouillon | Le brouillon est sauvé dans Drive, reprendre manuellement |
| WF3 échoue sur FAQ | Réexécuter le WF3 (le contenu Drive n'est pas écrasé) |
| WF4 ne s'exécute pas | Vérifier le cron, exécuter manuellement |
| WF5 échoue sur GSC | Vérifier les permissions OAuth, réexécuter |
| Rate limit API | Attendre 1h, réexécuter, augmenter les délais dans SplitInBatches |

---

## 4. Checklist de Validation Finale

### Avant le premier sprint

- [ ] Tous les 5 workflows importés et configurés
- [ ] Error Workflow activé
- [ ] Google Sheet avec 3 onglets créé
- [ ] Dossier Google Drive prêt
- [ ] Test WF1 réussi avec topic réel
- [ ] Test WF2 réussi avec article BOFU
- [ ] Test WF3 réussi avec E-E-A-T complet
- [ ] Test WF4 réussi avec données de calendrier
- [ ] Test WF5 réussi (même sans données GSC)
- [ ] Emails de notification reçus pour tous les workflows
- [ ] Budgets API vérifiés et alertes configurées

### Chaque semaine en production

- [ ] Vérifier le dashboard dans Google Sheets
- [ ] Relire le rapport hebdomadaire WF5
- [ ] Vérifier les exécutions dans n8n (erreurs?)
- [ ] Contrôler les coûts API
- [ ] Relire au moins 1 article généré (qualité)
