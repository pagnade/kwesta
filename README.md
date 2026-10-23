# Kwestor (MVP)

**Votre exposition, croisée en continu avec les événements mondiaux : la probabilité qu'ils soient réels, et ce qu'ils vous coûtent s'ils le sont.**

Kwestor s'adresse aux équipes supply chain, achats, risque et trésorerie dont l'activité dépend de routes, de ports, de fournisseurs et d'infrastructures exposés aux crises. L'outil ne cherche pas à dire ce qui est important pour le monde. Il dit ce qui compte **pour vous**, en dollars et en jours :

1. **Vos dépendances** (fournisseurs, sites, routes, contrats) sont importées et reliées à un graphe d'infrastructures critiques : détroits, ports, terminaux GNL, câbles, fonderies, cloud.
2. **Les événements mondiaux** sont détectés dans les flux d'actualités, dédupliqués, regroupés et notés. La question « est-ce réel ? » (crédibilité) reste séparée de « à quel point est-ce grave ? » (sévérité).
3. **L'impact** est chiffré sur vos expositions : perte attendue, scénario médian et défavorable, réponse la moins coûteuse (absorber sur le stock ou basculer). Les rumeurs à fort enjeu partent en vérification, pas en alerte.

Ce n'est ni un agrégateur d'actualités, ni un score de risque générique : sans exposition importée, Kwestor ne mesure que l'importance mondiale d'un événement, et le dit.

*Le nom* : *kwestor* est la forme polonaise de *quaestor*, le magistrat romain chargé du trésor public. Le mot vient de *quaerere*, « enquêter ».

> Lire aussi [`docs/ANALYSE_CRITIQUE.md`](docs/ANALYSE_CRITIQUE.md). Ce document conteste le blueprint initial et dit ce que ce MVP prouve et ne prouve pas.

## Démarrage

```bash
pip install -r requirements.txt
uvicorn kwestor.api:app --port 8000          # mode démo (données synthétiques, rejouées en « live »)
KWESTOR_MODE=live uvicorn kwestor.api:app    # flux RSS réels (BBC, Guardian, Al Jazeera, France 24, gCaptain)
KWESTOR_MODE=both ...                          # les deux
python -m pytest -q                             # 35 tests
```

Ouvrir http://localhost:8000. Variables d'environnement facultatives :
- `ANTHROPIC_API_KEY` : briefings narratifs générés par Claude. Sans clé, le briefing utilise un template déterministe.
- `KWESTOR_API_KEY` : exige l'en-tête `X-API-Key` sur les écritures.
- `KWESTOR_DB` : chemin du fichier SQLite.

## Ce que fait la console

| Vue | Contenu |
|---|---|
| Carte | Événements (couleur = palier, taille = score) et nœuds d'infrastructure (couleur = détresse *attendue* = détresse-si-vrai × crédibilité) |
| Flux | Filtres par palier, type, région, statut et texte. Marqueur `VERIFY` sur les événements à fort impact mais peu crédibles |
| Événement | Matrice **crédibilité × impact-si-vrai**, facteurs du score, impact propagé sur les actifs, timeline, provenance de chaque signal (grade de source, langue, copie syndiquée), événements corrélés par infrastructure partagée, workflow (investigating / monitoring / escalated / dismissed), vérité terrain (confirmé / faux), journal d'audit |
| Alertes | Une alerte par couple (événement, watchlist). Re-notification uniquement si le palier monte. Acquittement et clôture |
| Brief IA | Briefing ancré sur les événements notés, avec citations `[S<id>]` |
| Watchlists | Pays, nœuds, secteurs et types d'incident pondérés, palier minimal, webhook (Slack, Teams, SOAR) |
| Qualité | Taux de doublons et de bruit, compression signaux→événements, latence, score de Brier, fiabilité des sources mise à jour par les verdicts |

## Exposition client : l'impact en dollars et en jours

Le score de risque mesure l'importance d'un événement **pour le monde**. L'onglet *Exposure* mesure ce qu'il coûte **au client**, à partir de ses propres dépendances importées en CSV (modèle : `GET /api/exposures/template`, exemple fictif : `data/sample_exposures.csv`).

| Colonne | Sens |
|---|---|
| `nodes` / `countries` | Dépendances de l'exposition (ids de nœuds séparés par `;`, codes ISO2). Une route casse à son maillon le plus faible (max, pas somme) |
| `annual_value_usd` | Valeur annuelle du flux qui transite par ces dépendances |
| `buffer_days` | Jours de stock ou de couverture avant que la rupture ne touche l'activité |
| `reroute_days`, `reroute_cost_pct` | Alternative : délai de bascule et surcoût en % du flux. Les deux vides = pas d'alternative |
| `loss_given_stop` | Valeur détruite par dollar de flux arrêté. Peut dépasser 1 : une puce à 1 $ qui arrête une ligne de production à 6 $ |

**Calcul.** Pour chaque événement, la détresse-si-vrai propagée dans le graphe atteint (ou non) chaque exposition. Une durée de rupture est tirée d'un a priori par type d'incident (lognormale, σ = 0,8). La stratégie la moins coûteuse est retenue : absorber sur le stock, ou basculer. Perte attendue = crédibilité × E[perte | vrai].

**Deux points de rigueur intégrés :**
1. **Convexité.** `max(0, durée − stock)` est le payoff d'un call sur la durée, avec le stock pour strike. Évaluer à la durée médiane sous-estimerait l'espérance (Jensen), donc la perte est intégrée sur la distribution des durées. Effet sur la démo : Douala passe de 118 k$ (durée médiane) à 210 k$ (espérance).
2. **Pas de double comptage.** Trois dépêches Red Sea touchent la même route : au niveau portefeuille, on retient le maximum par exposition, et la somme brute n'est affichée qu'en borne haute.

**Risque de queue signalé.** Quand plus de 50 % de la perte attendue vient des 10 % de scénarios les plus longs, la console l'affiche. Exemple de la démo : l'exercice militaire autour de Taïwan pèse 1,3 M$ de perte attendue sur les MCU mono-source, alors que **le cas médian coûte 0 $** (45 jours de stock). Le chiffre repose entièrement sur l'hypothèse de durée, et il faut la challenger avant d'agir.

**Alertes.** La watchlist « Portfolio exposure ($) » se déclenche sur le palier financier (par défaut P1 ≥ 5 M$, P2 ≥ 1 M$, P3 ≥ 100 k$ de perte attendue, réglable via `KWESTOR_LOSS_TIERS`). La grève de Douala, P4 au niveau mondial, déclenche ainsi une alerte P3 pour le client. L'import est **atomique** : une seule ligne invalide (nœud inconnu, colonnes incohérentes) rejette tout, car un portefeuille à moitié importé sous-estimerait l'exposition sans le dire.

API : `GET /api/exposures`, `POST /api/exposures/import` (`{"csv": "...", "replace": true}`), `DELETE /api/exposures/{id}`, `GET /api/portfolio`.

## Pipeline

```
RSS / JSON / POST /api/ingest
  → URL canonique + hash de contenu (doublons exacts)
  → détection de langue (script + mots-outils)
  → MinHash 3-shingles (copies syndiquées : rattachées à l'événement, NON comptées comme sources indépendantes)
  → extraction : gazetteer pays/nœuds (alias multilingues, alias ambigus soumis au contexte régional),
                 lexiques pondérés par type d'incident, modulateurs d'intensité et de prudence (« unconfirmed », « drill »)
  → filtre de pertinence (≥ 2 indices convergents + localisation, ou nœud d'infrastructure)
  → clustering (similarité texte/titre + nœuds partagés + pays, fenêtre de 72 h)
  → scoring → alertes → bus SSE → console
```

## Modèle de risque : pourquoi cette forme

Le score suit la décomposition d'une perte attendue en crédit, **PD × LGD × EAD** :

| Composante | Question | Calcul |
|---|---|---|
| **Crédibilité** (≈ PD) | « Est-ce réel ? » | Agrégation bayésienne en log-odds des sources indépendantes. Un signal isolé vaut la fiabilité de sa source (grade Admiralty A–F). Chaque corroboration ajoute une évidence décroissante (1/(1+0,5k)) et ne retranche jamais. Le langage prudent réduit la fiabilité effective |
| **Sévérité** (≈ LGD) | « À quel point est-ce grave si c'est vrai ? » | A priori du type d'incident × intensité textuelle (fermeture, victimes, exercice, « rétabli »…) |
| **Exposition** (≈ EAD) | « Quelle part de ce qui compte est sur le chemin ? » | Choc propagé dans le graphe de dépendances façon *DebtRank* (chaque nœud transmet une seule fois, donc les cycles Suez ↔ Bab el-Mandeb n'amplifient pas), pondéré par la criticité du nœud |
| Décroissance, vélocité | Fraîcheur, accélération | Demi-vie par type ; nombre de sources indépendantes sur 6 h |

`score = 100·(1 − e^(−3·EL))` avec `EL = crédibilité × sévérité × (0,35 + 0,65·exposition) × décroissance × vélocité`.

La règle de conception clé : **on n'affiche pas un score unique**. Un événement à fort impact-si-vrai et faible crédibilité (une rumeur de fermeture d'Ormuz) n'est pas un « risque moyen ». C'est une **tâche de vérification**, routée vers un analyste, et non une alerte P1 vers la direction.

La boucle d'apprentissage : chaque verdict d'analyste met à jour la fiabilité de la source (Beta-Binomiale) et alimente le score de Brier. Tant qu'il n'y a pas une centaine de verdicts, **les scores sont des a priori, pas des mesures.**

## Résultats observés (exécution du 22/10/2026)

**Démo (23 signaux synthétiques)** : 2 bruits filtrés, 2 copies syndiquées détectées, 13 événements. La rumeur d'Ormuz (crédibilité 25 %, impact-si-vrai 0,64) part en alerte `VERIFY`. Le Red Sea passe P1 après corroboration. La grève de Douala (français) et la sécheresse du canal de Panama (espagnol) sont correctement typées.

**Live (145 articles RSS réels, 5 flux)** : 124 classés bruit, 18 événements. La première version laissait passer des faux positifs grossiers : une perquisition chez une élue classée « sécurité maritime » à cause de « seized », un article sur une flotte dual-fuel, une fusillade scolaire. Le filtre de pertinence a été durci, et la fusion par simple pays partagé interdite. Le reste est encore imparfait : plusieurs dépêches sur le même sujet ne fusionnent pas quand elles sont formulées différemment. C'est la limite attendue d'un clustering lexical. Les embeddings sont la prochaine étape, à justifier par un jeu étiqueté.

## Structure

```
kwestor/
  gazetteer.py  pays, 41 nœuds d'infrastructure/commodités, arêtes de dépendance (ordres de grandeur publics, à remplacer)
  text.py       normalisation, langue, MinHash, URL canonique
  extract.py    entités, géotag, classification, intensité/prudence
  graph.py      propagation DebtRank
  risk.py       crédibilité, sévérité, exposition, score, fiabilité Beta, Brier
  exposure.py   exposition client : import CSV validé, perte attendue $ (convexe, bande P50/P95, part de queue)
  pipeline.py   orchestration, clustering, rescoring, workflow, verdicts
  alerts.py     watchlists, cycle de vie des alertes, webhooks
  briefing.py   brief Claude (ancré, cité, fallback serveur) ou template
  api.py        FastAPI + SSE + workers démo/live/décroissance
  web/index.html console (MapLibre, sans build)
data/sample_signals.json   corpus SYNTHÉTIQUE de démonstration (fictif)
data/sample_exposures.csv  portefeuille FICTIF de démonstration
tests/
```

## Limites connues (volontairement visibles)

- **Durées de rupture = a priori grossiers** (ex. 21 jours médians pour un incident maritime, alors que la crise Red Sea a duré bien plus longtemps). C'est l'hypothèse qui pèse le plus sur les montants en dollars, et elle doit être remplacée par l'historique du client.
- **Pas de corrélation entre expositions** : les pertes de deux routes touchées par le même événement sont additionnées comme si elles étaient indépendantes. Pas de capacité limitée sur l'itinéraire alternatif (tout le monde ne peut pas basculer par le Cap en même temps).
- Les seuils de palier financier sont absolus. Pour une entreprise plus grande ou plus petite, il faut les régler (`KWESTOR_LOSS_TIERS`).
- Les **contradictions** ne sont pas modélisées : un démenti officiel rattaché à un événement augmente sa crédibilité au lieu de la réduire. Il faut une détection de stance (affirme / dément).
- Langues : alias et lexiques en/fr/es partiels, pas de traduction. Le script arabe, cyrillique ou chinois est détecté mais pas analysé.
- Pas de RBAC/OIDC, pas de classification par région. Seuls une clé API optionnelle et l'audit des actions sont en place.
- SQLite et bus en mémoire : un seul processus. Le schéma est transposable tel quel sur Postgres.
- Les figures de criticité et les poids d'arêtes sont des ordres de grandeur publics (EIA, CNUCED) **non calibrés**.
