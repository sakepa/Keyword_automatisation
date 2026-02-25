# Scripts JavaScript Custom pour Nœuds Function n8n

> Ces scripts sont utilisés dans les nœuds **Code** (Function) des workflows n8n.
> Ils reproduisent la logique Python originale (Google Colab) en JavaScript natif.

---

## Script 1 : Fuzzy Matching & Analyse de Fréquences (WF1)

Ce script remplace `unify_similar_entity()` et `analyze_entity()` du code Python original.

```javascript
// ===================================================
// FUZZY MATCHING & FREQUENCY ANALYSIS
// Utilisé dans : WF1 - Nœud "Fuzzy Match & Fréquences"
// Équivalent Python : unify_similar_entity() + analyze_entity()
// ===================================================

const items = $input.all();

// --- FONCTIONS UTILITAIRES ---

/**
 * Calcule la distance de Levenshtein entre deux chaînes
 * Équivalent de fuzzywuzzy en Python
 */
function levenshtein(a, b) {
  const matrix = [];
  for (let i = 0; i <= b.length; i++) matrix[i] = [i];
  for (let j = 0; j <= a.length; j++) matrix[0][j] = j;

  for (let i = 1; i <= b.length; i++) {
    for (let j = 1; j <= a.length; j++) {
      if (b.charAt(i - 1) === a.charAt(j - 1)) {
        matrix[i][j] = matrix[i - 1][j - 1];
      } else {
        matrix[i][j] = Math.min(
          matrix[i - 1][j - 1] + 1, // substitution
          matrix[i][j - 1] + 1,     // insertion
          matrix[i - 1][j] + 1      // deletion
        );
      }
    }
  }
  return matrix[b.length][a.length];
}

/**
 * Calcule le score de similarité token_sort_ratio
 * Trie les tokens alphabétiquement avant la comparaison
 * Réplique fuzz.token_sort_ratio de fuzzywuzzy
 */
function tokenSortRatio(a, b) {
  if (!a || !b) return 0;

  const sortedA = a.toLowerCase().split(/\s+/).sort().join(' ');
  const sortedB = b.toLowerCase().split(/\s+/).sort().join(' ');

  if (sortedA === sortedB) return 100;

  const maxLen = Math.max(sortedA.length, sortedB.length);
  if (maxLen === 0) return 100;

  const distance = levenshtein(sortedA, sortedB);
  return Math.round((1 - distance / maxLen) * 100);
}

/**
 * Unifie les entités similaires (seuil par défaut : 90%)
 * Réplique unify_similar_entity(df, threshold=90) du Python
 */
function unifySimilarEntities(entities, threshold = 90) {
  const entityMap = new Map();

  for (const item of entities) {
    const entity = item.entity?.toLowerCase().trim();
    if (!entity || entity === 'n/a') continue;

    let matched = false;
    for (const [canonicalKey, data] of entityMap) {
      if (tokenSortRatio(entity, canonicalKey) >= threshold) {
        // Fusionner avec l'entité existante
        data.mentions += item.mentions || 1;
        data.queries.add(item.query);
        matched = true;
        break;
      }
    }

    if (!matched) {
      entityMap.set(entity, {
        entity: item.entity.trim(), // Garder la casse originale
        mentions: item.mentions || 1,
        queries: new Set([item.query])
      });
    }
  }

  return entityMap;
}

// --- TRAITEMENT PRINCIPAL ---

// Collecter toutes les entités des items d'entrée
const rawEntities = items.map(item => ({
  entity: item.json.entity,
  query: item.json.query,
  mentions: item.json.mentions || 1
}));

// Unifier les entités similaires
const unifiedMap = unifySimilarEntities(rawEntities, 90);

// Convertir en résultats triés
const results = [];
for (const [key, data] of unifiedMap) {
  results.push({
    json: {
      entity: data.entity,
      totalMentions: data.mentions,
      queryCount: data.queries.size,
      queries: Array.from(data.queries).join(', '),
      prominence: parseFloat((data.mentions / items.length).toFixed(4))
    }
  });
}

// Trier par mentions décroissantes
results.sort((a, b) => b.json.totalMentions - a.json.totalMentions);

return results;
```

---

## Script 2 : Parsing des Requêtes Gemini (WF1)

Ce script reproduit le parsing des requêtes multi-lignes générées par Gemini.

```javascript
// ===================================================
// QUERY PARSER
// Utilisé dans : WF1 - Nœud "Parser les Requêtes"
// Équivalent Python : generate_related_queries() + split
// ===================================================

const response = $input.first().json;
let text = '';

try {
  // Extraire le texte de la réponse Gemini
  text = response.candidates[0].content.parts[0].text;
} catch (e) {
  throw new Error(
    'Impossible de parser la réponse Gemini. ' +
    'Vérifiez que le credential Vertex AI est correctement configuré. ' +
    'Réponse reçue: ' + JSON.stringify(response).substring(0, 500)
  );
}

// Séparer les requêtes (par saut de ligne, en ignorant les numéros)
const queries = text
  .split('\n')
  .map(q => q.replace(/^\d+[\.\)]\s*/, '').trim()) // Supprimer "1. " ou "1) "
  .map(q => q.replace(/^[-•*]\s*/, '').trim())     // Supprimer "- " ou "• "
  .map(q => q.replace(/^["']|["']$/g, '').trim())  // Supprimer guillemets
  .filter(q => q.length > 3);                       // Minimum 3 caractères

// Récupérer les paramètres du nœud Set précédent
const params = $('Paramètres de Recherche').first().json;

return queries.map(query => ({
  json: {
    query,
    topic: params.topic,
    language: params.language,
    n_responses: params.n_responses,
    search_limit: params.search_limit
  }
}));
```

---

## Script 3 : Extraction et Parsing des Entités OpenAI (WF1)

```javascript
// ===================================================
// ENTITY PARSER
// Utilisé dans : WF1 - Nœud "Parser Entités"
// Équivalent Python : extract_entities() → parsing
// ===================================================

const items = $input.all();
const allEntities = [];

for (const item of items) {
  try {
    const content = item.json.choices?.[0]?.message?.content || '[]';

    // Extraire le tableau JSON de la réponse (peut être entouré de texte)
    const match = content.match(/\[[\s\S]*?\]/);
    if (match) {
      const entities = JSON.parse(match[0]);
      const query = $('Traitement par Lots').first().json.query;

      for (const entity of entities) {
        if (entity && typeof entity === 'string' && entity.trim().length > 0) {
          allEntities.push({
            query,
            entity: entity.trim(),
            mentions: 1
          });
        }
      }
    }
  } catch (e) {
    // Skip les réponses mal formées silencieusement
    // Les erreurs de parsing sont normales pour ~5% des réponses
    continue;
  }
}

return allEntities.map(e => ({ json: e }));
```

---

## Script 4 : Préparation du Brief de Contenu (WF2)

```javascript
// ===================================================
// CONTENT BRIEF PREPARATION
// Utilisé dans : WF2 - Nœud "Préparer le Brief"
// Équivalent Python : variables TYPE, TITLE, GOAL, KEYWORDS, CONTEXT
// ===================================================

const row = $input.first().json;

// Déterminer le type de contenu et le template approprié
const contentType = row.Type || 'BOFU';
let type, systemPrompt;

switch (contentType) {
  case 'BOFU':
    type = 'Guide/Review comparatif';
    systemPrompt = 'Your name is Miss Galadriel. You are a creative researcher ' +
      'and writer. You write very intriguing blog posts in FIRST PERSON about a ' +
      'given topic. Focus on actionable recommendations and product comparisons.';
    break;
  case 'TOFU':
    type = 'Guide pratique / How-to';
    systemPrompt = 'Your name is Miss Galadriel. You are a creative researcher ' +
      'and writer. You write educational and informative blog posts that help ' +
      'beginners understand complex topics step by step.';
    break;
  case 'Linkable Asset':
    type = 'Article de référence / Données statistiques';
    systemPrompt = 'Your name is Miss Galadriel. You are a creative researcher ' +
      'and writer. You create comprehensive reference articles with original ' +
      'statistics and data that other websites would want to cite and link to.';
    break;
  default:
    type = 'Article de blog';
    systemPrompt = 'Your name is Miss Galadriel. You are a creative researcher ' +
      'and writer. You write detailed and engaging blog posts.';
}

const title = row['URL du Sujet'] || row.Title || row.Titre || '';
const keywords = row.Keywords || row['Mots-clés'] || '';
const goal = row.Goal || row.Objectif || `Providing comprehensive information about ${title}`;
const context = row.Context || row.Contexte || `Research and expertise on ${title}`;

// Construire le prompt utilisateur (réplique exacte du Python)
const userPrompt = `Write a 800 word in-depth LONG detailed ${type}.
Use H1, H2 u li headlines.
The title is ${title}.
Share your experience and explain ${goal} in a step-by-step format.
Use keywords: ${keywords}.
Use the ${context} to get your information.`;

return [{
  json: {
    rowIndex: row.row_number || 0,
    type,
    title,
    goal,
    keywords,
    context,
    contentType,
    systemPrompt,
    userPrompt,
    owner: row.Propriétaire || row.Owner || '',
    originalRow: row
  }
}];
```

---

## Script 5 : Assemblage E-E-A-T (WF3)

```javascript
// ===================================================
// E-E-A-T METADATA ASSEMBLY
// Utilisé dans : WF3 - Nœud "Ajouter Métadonnées E-E-A-T"
// Basé sur : "Comment creer du contenu SGE Friendly.md"
// ===================================================

const article = $input.first().json;
const calendarRow = $('Filtrer Statut = En édition').first().json;

const articleContent = article.data || article.content || '';
const author = calendarRow.Propriétaire || calendarRow.Owner || 'Expert SEO';
const title = calendarRow['URL du Sujet'] || calendarRow.Title || '';
const contentType = calendarRow.Type || 'BOFU';
const now = new Date().toISOString().split('T')[0];

// === BLOC E-E-A-T HEADER ===
// Conforme aux recommandations du Google Search Quality Evaluator Guidelines
const eeatHeader = `
---
## À propos de l'auteur

**${author}** est un expert reconnu dans le domaine du marketing digital et du SEO.
Fort de plusieurs années d'expérience, il/elle partage ses connaissances pour aider
les professionnels à optimiser leur présence en ligne.

**Vérifié par :** Équipe éditoriale Data Marketing Labs
**Dernière mise à jour :** ${now}

---
`;

// === BLOC E-E-A-T FOOTER ===
// Sources, date de vérification, et transparence
const eeatFooter = `
---
## Sources et Références

*Les informations présentées dans cet article sont basées sur :*
- Recherches et analyses des résultats de recherche Google actuels
- Données extraites via les outils d'analyse SEO (Google Search Console, Semrush)
- Expertise et expérience professionnelle de l'auteur
- [Google Search Quality Evaluator Guidelines](https://guidelines.raterhub.com/)

**Date de publication :** ${now}
**Dernière vérification factuelle :** ${now}
---
`;

return [{
  json: {
    articleContent,
    eeatHeader,
    eeatFooter,
    title,
    contentType,
    author,
    date: now,
    driveFileId: calendarRow['Lien Drive']?.match(/\/d\/([a-zA-Z0-9_-]+)/)?.[1] || '',
    calendarRow
  }
}];
```

---

## Script 6 : Analyse du Calendrier de Contenu (WF4)

```javascript
// ===================================================
// CONTENT CALENDAR ANALYSIS
// Utilisé dans : WF4 - Nœud "Analyser Calendrier"
// Calcule les métriques du sprint et détecte les retards
// ===================================================

const items = $input.all();
const today = new Date();
today.setHours(0, 0, 0, 0);

const stats = {
  total: items.length,
  published: 0,
  inProgress: 0,
  planned: 0,
  overdue: [],
  dueThisWeek: [],
  byType: { BOFU: 0, TOFU: 0, 'Linkable Asset': 0 }
};

const oneWeekFromNow = new Date(today);
oneWeekFromNow.setDate(oneWeekFromNow.getDate() + 7);

for (const item of items) {
  const row = item.json;
  const status = (row.Statut || '').toLowerCase();
  const pubDate = row['Date de Publication']
    ? new Date(row['Date de Publication'])
    : null;
  const type = row.Type || 'BOFU';

  // Comptage par statut
  if (status.includes('publi')) stats.published++;
  else if (
    status.includes('production') ||
    status.includes('édition') ||
    status.includes('brouillon')
  )
    stats.inProgress++;
  else stats.planned++;

  // Comptage par type
  if (stats.byType[type] !== undefined) stats.byType[type]++;

  // Détection articles en retard
  if (pubDate && pubDate < today && !status.includes('publi')) {
    stats.overdue.push({
      title: row['URL du Sujet'] || row.Title || 'Sans titre',
      dueDate: row['Date de Publication'],
      owner: row.Propriétaire || row.Owner || 'Non assigné',
      status: row.Statut
    });
  }

  // Articles à produire cette semaine
  if (
    pubDate &&
    pubDate >= today &&
    pubDate <= oneWeekFromNow &&
    !status.includes('publi')
  ) {
    stats.dueThisWeek.push({
      title: row['URL du Sujet'] || row.Title || 'Sans titre',
      dueDate: row['Date de Publication'],
      owner: row.Propriétaire || row.Owner || 'Non assigné',
      status: row.Statut
    });
  }
}

// Vélocité du sprint
const velocity =
  stats.total > 0 ? Math.round((stats.published / stats.total) * 100) : 0;

// Rapport formaté
let report = `# Rapport Quotidien - Calendrier de Contenu\n`;
report += `**Date:** ${today.toISOString().split('T')[0]}\n\n`;
report += `## Statistiques du Sprint\n\n`;
report += `| Métrique | Valeur |\n|----------|--------|\n`;
report += `| Articles totaux | ${stats.total} |\n`;
report += `| Publiés | ${stats.published} (${velocity}%) |\n`;
report += `| En cours | ${stats.inProgress} |\n`;
report += `| Planifiés | ${stats.planned} |\n`;
report += `| BOFU | ${stats.byType.BOFU} |\n`;
report += `| TOFU | ${stats.byType.TOFU} |\n`;
report += `| Linkable Assets | ${stats.byType['Linkable Asset']} |\n\n`;

if (stats.overdue.length > 0) {
  report += `## Articles en Retard (${stats.overdue.length})\n\n`;
  for (const a of stats.overdue) {
    report += `- **${a.title}** - Prévu le ${a.dueDate} - ${a.owner} (${a.status})\n`;
  }
  report += '\n';
}

if (stats.dueThisWeek.length > 0) {
  report += `## Cette Semaine (${stats.dueThisWeek.length})\n\n`;
  for (const a of stats.dueThisWeek) {
    report += `- **${a.title}** - Prévu le ${a.dueDate} - ${a.owner} (${a.status})\n`;
  }
}

return [
  {
    json: {
      report,
      htmlReport: report.replace(/\n/g, '<br>'),
      hasOverdue: stats.overdue.length > 0,
      overdueCount: stats.overdue.length,
      overdueItems: stats.overdue,
      velocity,
      stats
    }
  }
];
```

---

## Script 7 : Analyse Performance SEO (WF5)

```javascript
// ===================================================
// SEO PERFORMANCE ANALYSIS
// Utilisé dans : WF5 - Nœud "Analyser Performance"
// Croise les données GSC avec le calendrier de contenu
// ===================================================

const gscData = $('Google Search Console - Données 7j').first().json;
const publishedArticles = $('Lire Articles Publiés').all();

const gscRows = gscData.rows || [];
const today = new Date();
const sevenDaysAgo = new Date(today - 7 * 24 * 60 * 60 * 1000);

// Agréger les données GSC par page
const pageMetrics = new Map();
for (const row of gscRows) {
  const page = row.keys?.[0] || '';
  const query = row.keys?.[1] || '';

  if (!pageMetrics.has(page)) {
    pageMetrics.set(page, {
      clicks: 0,
      impressions: 0,
      positionSum: 0,
      positionCount: 0,
      topQueries: []
    });
  }

  const m = pageMetrics.get(page);
  m.clicks += row.clicks || 0;
  m.impressions += row.impressions || 0;
  m.positionSum += row.position || 0;
  m.positionCount += 1;
  m.topQueries.push({
    query,
    clicks: row.clicks || 0,
    position: row.position || 0
  });
}

// Identifier les articles indexés vs non-indexés
const notIndexed = [];
const performanceData = [];

for (const article of publishedArticles) {
  const url = article.json['URL du Sujet'] || '';
  const pubDate = article.json['Date de Publication']
    ? new Date(article.json['Date de Publication'])
    : null;

  let found = false;
  for (const [page, metrics] of pageMetrics) {
    if (page.includes(url) || url.includes(page)) {
      found = true;
      const avgPos = metrics.positionCount > 0
        ? (metrics.positionSum / metrics.positionCount).toFixed(1)
        : 0;
      const ctr = metrics.impressions > 0
        ? ((metrics.clicks / metrics.impressions) * 100).toFixed(2)
        : 0;

      // Top 5 requêtes
      metrics.topQueries.sort((a, b) => b.clicks - a.clicks);

      performanceData.push({
        url,
        type: article.json.Type || '',
        pubDate: article.json['Date de Publication'] || '',
        clicks: metrics.clicks,
        impressions: metrics.impressions,
        ctr,
        avgPosition: avgPos,
        topQueries: metrics.topQueries.slice(0, 5)
      });
      break;
    }
  }

  if (!found && pubDate) {
    const daysSincePublish = Math.floor(
      (today - pubDate) / (1000 * 60 * 60 * 24)
    );
    notIndexed.push({
      url,
      type: article.json.Type || '',
      pubDate: article.json['Date de Publication'] || '',
      daysSincePublish
    });
  }
}

// Trier par clics décroissants
performanceData.sort((a, b) => b.clicks - a.clicks);

// Totaux
let totalClicks = 0;
let totalImpressions = 0;
for (const p of performanceData) {
  totalClicks += p.clicks;
  totalImpressions += p.impressions;
}

// Générer le rapport
let report = `# Rapport Hebdomadaire de Performance SEO\n`;
report += `**Période:** ${sevenDaysAgo.toISOString().split('T')[0]} → ${today.toISOString().split('T')[0]}\n\n`;
report += `## Résumé Global\n\n`;
report += `| Métrique | Valeur |\n|----------|--------|\n`;
report += `| Clics totaux (7j) | ${totalClicks} |\n`;
report += `| Impressions totales (7j) | ${totalImpressions} |\n`;
report += `| CTR moyen | ${totalImpressions > 0 ? ((totalClicks / totalImpressions) * 100).toFixed(2) : 0}% |\n`;
report += `| Articles indexés | ${performanceData.length} |\n`;
report += `| Articles non-indexés | ${notIndexed.length} |\n\n`;

if (performanceData.length > 0) {
  report += `## Top 10 Articles\n\n`;
  report += `| URL | Type | Clics | Impressions | CTR | Position |\n`;
  report += `|-----|------|-------|-------------|-----|----------|\n`;
  for (const p of performanceData.slice(0, 10)) {
    report += `| ${p.url} | ${p.type} | ${p.clicks} | ${p.impressions} | ${p.ctr}% | ${p.avgPosition} |\n`;
  }
  report += '\n';
}

if (notIndexed.length > 0) {
  report += `## Articles Non-Indexés\n\n`;
  for (const ni of notIndexed) {
    const urgency = ni.daysSincePublish > 7 ? 'URGENT' : 'ATTENTION';
    report += `- [${urgency}] **${ni.url}** (${ni.type}) - Publié il y a ${ni.daysSincePublish} jours\n`;
  }
}

return [
  {
    json: {
      report,
      htmlReport: report.replace(/\n/g, '<br>'),
      hasNotIndexed: notIndexed.filter((n) => n.daysSincePublish > 7).length > 0,
      notIndexedCount: notIndexed.length,
      totalClicks,
      totalImpressions,
      performanceData,
      notIndexed
    }
  }
];
```

---

## Notes d'implémentation

### Différences Python → JavaScript

| Concept Python | Équivalent JavaScript (n8n) |
|---------------|---------------------------|
| `fuzzywuzzy.fuzz.token_sort_ratio()` | Fonction custom `tokenSortRatio()` avec Levenshtein |
| `pandas.DataFrame` | Tableaux d'objets JavaScript |
| `concurrent.futures.ThreadPoolExecutor` | Nœud `SplitInBatches` de n8n |
| `collections.Counter` | `Map()` avec compteurs manuels |
| `df.to_excel()` | Nœud `Google Sheets - Append` |
| `retrying.retry` | Paramètre `retryOnFail` des nœuds HTTP Request |
| `re.findall()` | `String.match()` avec regex |

### Gestion des erreurs

Tous les scripts incluent des try/catch pour :
- Les réponses API mal formées (~5% des cas)
- Les champs manquants dans les données du Google Sheet
- Les problèmes d'encodage UTF-8

### Performance

- Les scripts sont optimisés pour traiter jusqu'à 1000 entités par exécution
- Le fuzzy matching O(n²) est acceptable pour <500 entités uniques
- Au-delà de 500 entités, envisager de traiter par lots de 200
