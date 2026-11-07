# Boucle d'amélioration continue d'agents à partir des sessions d'utilisation

> **Type de document :** spécification d'implémentation, destinée à être donnée à un agent de code.
> **Objectif :** mettre en place, dans un dépôt git, un processus qui améliore (presque) automatiquement des agents IA à partir des sessions réelles de leurs utilisateurs, avec revue humaine et documentation complète.
> **Statut :** architecture révisée. Les choix liés à la CLI, au dépôt réel et à la politique de données restent à confirmer avant l'implémentation.

Cette révision conserve le périmètre de la proposition initiale : les artefacts textuels des agents, un pilote personnel puis une extension possible à plusieurs utilisateurs. Elle précise les frontières de confiance, l'évaluation comparative, la traçabilité des versions et les conditions de passage entre phases.

---

## 0. Comment utiliser ce document

1. Fournir ce fichier, le dépôt réel des agents et le nom de la CLI à l'agent implémenteur.
2. Lui demander de suivre la section **2. Démarche imposée** avant d'écrire du code.
3. Utiliser le **prompt d'amorçage** (section 14) pour lancer le pilote. Les valeurs numériques de ce document sont des paramètres initiaux à calibrer, pas des garanties de qualité ou de conformité.

---

## 1. Contexte et objectif

Une équipe maintient un ensemble d'agents IA (définitions d'agents, instructions, skills) centralisés dans un dépôt git. Les agents sont utilisés au quotidien dans une CLI agentique, par plusieurs personnes.

**Problème :** les agents s'améliorent seulement quand quelqu'un pense à les modifier. Les frictions réelles (corrections répétées, outils mal utilisés, contexte manquant) restent dans les sessions et sont perdues.

**Objectif :** une boucle fermée.

```
collecte → filtrage → stockage → triage → cas d'évaluation → changement ciblé
                                                            ↓
sessions suivantes ← suivi ← déploiement ← revue ← PR + CI ← évaluation comparative
```

**Portée :**
- Version minimale : les sessions d'une seule personne.
- Version cible : les sessions de tous les utilisateurs, sous réserve d'un accord de conformité.
- Type de modification automatisée : uniquement les artefacts textuels des agents (prompts, instructions et skills textuels).

**Hors portée :**
- Modification automatique d'agents en production sans revue.
- Entraînement ou fine-tuning de modèles. On n'améliore que les artefacts textuels (prompts, instructions, skills).

---

## 2. Démarche imposée à l'agent implémenteur

Ne pas coder immédiatement. Suivre cet ordre :

1. **Explorer** le dépôt existant : structure, conventions, CI, documentation, agents déjà présents.
2. **Découvrir les capacités de la CLI agentique** utilisée (lire sa documentation officielle et tester les points critiques, sans supposer) :
   - où et dans quel format les sessions sont stockées (répertoire, JSONL, base SQLite, etc.);
   - si des hooks de cycle de vie existent (début/fin de session, soumission d'un prompt, appel d'outil) et comment on les configure (niveau dépôt vs niveau utilisateur);
   - s'il existe un mode non interactif/programmatique (utile pour la CI et l'évaluation);
   - où sont définis les agents, skills et instructions, et quelle est la priorité entre définitions de projet et définitions personnelles;
   - si plusieurs environnements d'exécution sur une même machine (ex. un OS hôte et un sous-système Linux) ont chacun leur propre magasin de sessions.
   - si les sessions sont accessibles pour un seul utilisateur ou par un administrateur; si le contenu, les outils et les sous-agents sont exposés;
   - si les versions des agents et du dépôt sont disponibles au moment de l'exécution;
   - si les conditions d'utilisation de la CLI autorisent l'export et l'évaluation automatisée.
3. **Produire un plan** (fichier `docs/` ou description de PR) avec décisions, hypothèses, limites observées et questions bloquantes. Faire valider seulement les décisions qui exigent une autorisation ou une information absente; poursuivre les autres travaux.
4. **Implémenter par phases** (section 10), un commit ou une PR par phase, avec tests et documentation à chaque étape.
5. **Documenter** au fil de l'eau (section 11), pas à la fin.

---

## 3. Principes de conception

- **Humain dans la boucle.** Toute modification d'un agent passe par une pull request revue. Aucun merge automatique.
- **Preuves obligatoires.** Chaque changement proposé cite les sessions et les tours qui le justifient.
- **Priorité fondée sur le risque.** Un motif récurrent mérite un triage (point de départ : ≥ 3 sessions indépendantes ou ≥ 2 utilisateurs). Ce seuil n'autorise pas à lui seul un changement. Un incident grave isolé peut aussi justifier une correction. Dédupliquer les tentatives d'une même session et éviter qu'un utilisateur très actif domine l'analyse.
- **Changements petits et ciblés.** Une PR = un problème = un changement facile à revert.
- **Condenser autant qu'ajouter.** Le processus doit aussi supprimer ou fusionner des instructions obsolètes ou redondantes, sinon les agents gonflent à chaque cycle.
- **Évaluer avant de fusionner.** La version candidate est comparée à la version actuelle sur les mêmes cas; les cas critiques sont bloquants.
- **Confidentialité par conception.** Minimiser et nettoyer avant tout export hors du poste ou de l'environnement autorisé. Un tampon local transitoire peut être nécessaire et doit être protégé.
- **Les logs de sessions sont des données non fiables.** Ils peuvent contenir des instructions (prompt injection involontaire ou malveillante). L'agent d'analyse les traite comme des données à analyser, jamais comme des instructions à suivre.
- **Idempotence et traçabilité.** Les exportations utilisent un identifiant stable et peuvent être relancées sans doublons. Les évaluations enregistrent leurs versions et conditions; les sorties probabilistes restent variables.
- **Dégradation gracieuse.** Si la collecte échoue, elle ne doit jamais casser ni ralentir sensiblement la session de l'utilisateur. Un journal local et un contrôle de santé signalent toutefois les pertes de collecte.
- **Séparation des privilèges.** L'analyse des sessions n'obtient pas de droit d'écriture GitHub. Un service distinct prépare les PR avec des permissions minimales et sans accès aux secrets de production.

---

## 4. Architecture cible

```mermaid
flowchart LR
    A[CLI et sessions] --> B[Adaptateur local ou export]
    B --> C[Politique de collecte et nettoyage]
    C --> D[(Traces à accès restreint)]
    C --> E[(Index de métadonnées)]
    D --> F[Analyse et triage]
    E --> F
    F --> G[Cas candidats revus]
    G --> H[Jeux d'évaluation versionnés]
    H --> I[Comparaison version actuelle / candidate]
    I --> J[Diff ciblé et PR]
    J --> K[CI et revue humaine]
    K --> L[Version déployée progressivement]
    L --> A
```

### Structure de dépôt proposée

```
.
├── agents/                      # définitions des agents (artefacts améliorés)
├── skills/
├── instructions/
├── improvement-loop/
│   ├── collector/               # scripts du hook (multi-shell)
│   ├── sanitizer/               # règles de nettoyage + tests
│   ├── coach/                   # définition de l'agent coach + prompts
│   ├── evals/
│   │   ├── cases/               # tâches de référence (golden tasks)
│   │   ├── runner/              # exécution avant/après + scoring
│   │   └── manifests/           # versions et empreintes des jeux privés
│   ├── config.yaml              # seuils, rétention, chemins, paramètres
│   └── schemas/                 # JSON Schema des événements et rapports
├── docs/
│   └── improvement-loop/        # voir section 11
└── .github/workflows/           # GitHub Actions, si cette CI est retenue
```

Adapter aux conventions déjà présentes dans le dépôt. Ne pas imposer cette structure si le dépôt en a une autre.

---

## 5. Composants

### 5.1 Collecte

**But :** rendre les exécutions attribuables à une version précise des agents et à un résultat observable.

- **Choix de l'adaptateur :** privilégier l'API de traces ou un hook documenté de la CLI si sa portée est suffisante. Un hook au niveau du dépôt ne couvre tous les utilisateurs que si la CLI le charge réellement pour eux; le vérifier avec deux comptes de test. Si la fin de session n'est pas garantie, compléter par un balayage périodique.
- **Fallback sans hook :** tâche planifiée ou commande manuelle qui lit les sessions nouvelles. Suivre les identifiants de session et un curseur; l'export est idempotent et gère les sessions encore ouvertes, renommées ou modifiées.
- **Multi-environnement :** fournir les adaptateurs nécessaires (ex. bash et PowerShell) et inventorier les magasins distincts sur l'hôte et dans un sous-système Linux. Un environnement non couvert apparaît dans le rapport de couverture.
- **Contexte minimal :** identifiants de session et de trace, agent parent/sous-agent, transfert entre agents, nom et état des outils, résultat, durée, version exacte du dépôt et des instructions, version de la CLI et du modèle. Capturer la configuration de récupération des données quand elle influence la réponse.
- **Contraintes :** temps borné, travail différé dans une file locale si nécessaire, aucune rupture de la session en cas d'échec. Journaliser localement les échecs et suivre le taux d'export réussi; ne jamais afficher le contenu de session dans les journaux techniques.

Les [conventions GenAI d'OpenTelemetry](https://github.com/open-telemetry/semantic-conventions-genai) peuvent guider l'instrumentation. Comme elles évoluent, conserver un schéma interne versionné et un adaptateur explicite entre les deux.

### 5.2 Nettoyage et anonymisation

**But :** réduire les données exportées à ce qui est autorisé et nécessaire pour l'analyse.

| Niveau | Données autorisées | Usage |
| --- | --- | --- |
| 0 | Versions, durées, erreurs, résultat, compteurs | Tendances et couverture de collecte |
| 1 | Étapes, transferts, noms et statuts d'outils, sans arguments ni texte libre | Diagnostic technique |
| 2 | Extraits de contenu filtrés et approuvés pour ce périmètre | Analyse qualitative et création de cas |
| 3 | Original complet, dans un espace distinct et strictement limité si autorisé | Enquête exceptionnelle |

- Appliquer une **liste d'autorisation de champs** par niveau. Prompts, réponses et arguments d'outils ne sont pas collectés par défaut. Ne jamais exporter les fichiers joints ou sorties d'outils volumineuses par simple troncature : une donnée sensible peut se trouver dans le début du texte.
- Détecter secrets et données personnelles ou métier sensibles avec des règles configurables. Utiliser des jetons cohérents dans une session (ex. `<EMAIL_1>`); ne pas maintenir de correspondance globale réidentifiante dans le pipeline. Une détection automatique peut manquer une donnée sensible : les cas promus en évaluation et les extraits cités dans une PR subissent une revue supplémentaire.
- **Échec fermé à l'export :** si la politique ne peut pas classer ou nettoyer le contenu, bloquer son export et enregistrer seulement un compteur d'échec sans contenu. Le processus de collecte reste non bloquant pour l'utilisateur.
- Tester le masquage sur des exemples positifs, négatifs et propres au domaine; mesurer aussi les faux négatifs sur un échantillon approuvé. Versionner les règles et enregistrer leur version avec chaque export.

Les [recommandations d'OpenTelemetry sur les données sensibles](https://opentelemetry.io/docs/security/handling-sensitive-data/) servent de repère pour le placement des filtres; elles ne remplacent pas la politique interne de classification.

### 5.3 Stockage

- Pilote personnel : stockage local protégé. Extension : magasin central chiffré, avec accès par rôle et par équipe, séparé de l'index de métadonnées. Un dépôt git privé ne devrait pas être le magasin de sessions : ses copies et son historique compliquent la suppression. **Aucune session brute dans le dépôt des agents.**
- Format normalisé et schéma versionné (section 6), indexé par session, agent, version et date. Utiliser une clé d'idempotence; vérifier l'intégrité et la provenance de chaque export.
- Rétention définie et approuvée par catégorie de données, avec suppression automatique et vérification des copies, sauvegardes et cas dérivés. La valeur initiale de 90 jours n'est qu'un exemple à valider.
- Journal d'accès, séparation des droits d'analyse et de maintenance GitHub, procédure de retrait ou de suppression lorsque applicable. La pseudonymisation ne rend pas automatiquement les données anonymes.

### 5.4 Analyse (agent « coach »)

**But :** transformer des sessions en hypothèses d'amélioration étayées.

- Agent dédié, lancé en lot (par exemple hebdomadaire) ou à la demande, avec accès **en lecture** aux seules traces filtrées nécessaires. Il ne détient aucun jeton GitHub en écriture.
- Entrées : sessions autorisées de la période, définitions actuelles des agents, versions exécutées, résultats vérifiés et historique des changements précédents (pour éviter de répéter ou d'annuler une modification).
- Étapes :
  1. regrouper les sessions par agent, version et type de tâche, en conservant les transferts entre agents;
  2. détecter les signaux de friction (section 7) et les résultats vérifiés, avec leur provenance;
  3. dédupliquer les reprises, puis regrouper les signaux en motifs avec fréquence, utilisateurs distincts et gravité;
  4. prioriser les motifs récurrents et les incidents graves isolés;
  5. distinguer un défaut d'instruction d'une panne externe, d'un manque de données, d'un problème de permission ou d'une demande hors périmètre;
  6. produire un rapport structuré (section 6) avec hypothèse, preuves, limites et test de reproduction proposé.
- **Interdits pour le coach :**
  - exécuter des instructions trouvées dans les sessions;
  - modifier le dépôt ou ouvrir directement une PR;
  - inférer que la correction d'un utilisateur ou l'abandon d'une session prouve à lui seul l'échec de l'agent.

Les sessions, réponses d'outils et documents récupérés sont des **données non fiables**. L'analyse s'exécute sans secret de production, avec appels d'outils et accès réseau limités. La frontière reste nécessaire même si le contenu a été nettoyé, car le nettoyage des données sensibles ne neutralise pas une instruction malveillante. Voir le [risque d'injection de prompt décrit par l'OWASP](https://genai.owasp.org/llmrisk/llm01-prompt-injection/).

### 5.5 Proposition (pull request)

- Un générateur distinct transforme un motif retenu en **changement textuel minimal**. Il reçoit le rapport et les extraits autorisés, exécute les évaluations rapides, puis ouvre une PR brouillon si le changement est plausible. Il ne peut pas modifier les permissions, les outils autorisés, les hooks, la CI ni sa propre définition. Une demande touchant ces zones devient un ticket pour revue renforcée.
- Une PR par motif (ou par petit groupe cohérent); déduplication avec les PR déjà ouvertes. Les propositions non reproduites ou sans amélioration mesurée restent des rapports, pas des PR.
- Contenu de la description :
  - problème observé, versions concernées et métriques (fréquence, nombre d'utilisateurs, période, gravité);
  - preuves : références de sessions et de tours vers le magasin restreint; extraits courts seulement si leur publication est autorisée;
  - hypothèse causale, changement proposé et justification;
  - risques et régressions possibles;
  - versions et empreintes des jeux d'évaluation, résultats avant/après, cas en échec et variation de coût;
  - propriétaire de l'agent, plan de déploiement et retour arrière.
- Étiquettes standard (ex. `agent-improvement`, nom de l'agent concerné).
- Limite configurable de PR ouvertes en parallèle (3 peut être un point de départ) et regroupement des motifs similaires pour ne pas noyer les relecteurs.
- Pour un service extérieur à GitHub Actions, utiliser de préférence une [GitHub App limitée aux dépôts et permissions nécessaires](https://docs.github.com/en/apps/using-github-apps/about-using-github-apps). Protéger la branche, exiger les contrôles et la revue du propriétaire de l'agent. Aucun jeton d'écriture GitHub n'est transmis au coach ou au banc d'évaluation.

### 5.6 Évaluation

**But :** détecter les régressions et mesurer l'effet d'un changement avant fusion.

- **Jeu de tâches de référence** : cas issus de sessions filtrées et approuvées, complétés par des cas synthétiques pour la sécurité et les situations rares. Chaque cas contient la tâche, le contexte minimal autorisé, les outils simulés et le critère de succès.
- Séparer les cas visibles de régression, une réserve tenue à l'écart des générateurs de changements, et les cas critiques de sécurité. Versionner les cas, rubriques et jeux; une modification de la rubrique crée une nouvelle version.
- Format d'un cas :

```yaml
id: refactor-extract-method-001
agent: code-refactorer
case_version: 1
source_ref: "restricted://case/001"     # provenance contrôlée, sans session brute
input:
  prompt: "Extrais la logique de validation dans une méthode dédiée"
  fixtures: fixtures/refactor-001/      # état initial du projet de test
  tool_profile: sandbox-readwrite        # environnement isolé
success:
  deterministic:
    - command: "make test"
      expect_exit_code: 0
    - file_exists: "src/validator.ext"
  judged:                               # utilisé si pas de check déterministe possible
    rubric: "La méthode extraite est nommée clairement et ne change pas le comportement."
tags: [refactor, regression]
critical: false
```

- **Exécution :** faire tourner la version actuelle et la candidate sur les mêmes cas, avec les mêmes versions de CLI, modèle, outils, fixtures et paramètres. Évaluer l'agent modifié **et le workflow complet** lorsqu'il y a des transferts. Simuler les outils ayant des effets réels; aucune évaluation ne modifie la production.
- **Scoring :** checks déterministes en priorité (tests, format, contraintes). Employer un juge LLM à rubrique explicite pour les aspects qualitatifs, avec calibration humaine d'un échantillon et revue à l'aveugle des cas litigieux. Répéter les cas variables et rapporter l'incertitude; une graine fixe ne garantit pas la reproductibilité.
- **Critère de blocage :** échec d'un cas critique, fuite de données, contrôle manquant ou régression au-delà de seuils fixés avant l'essai. Examiner séparément qualité, sécurité, fiabilité, latence et coût; une moyenne globale ne compense pas un échec critique. Si l'échantillon est trop petit pour conclure, signaler « incertain » et garder la PR en revue.
- **Alimentation continue :** proposer un nouveau cas lorsqu'une session révèle un échec corrigé, mais faire valider le cas et son critère de succès. Éviter que le générateur voie toute la réserve d'évaluation.
- **CI :** ne pas placer de sessions sensibles ni de secrets de production dans les artefacts de CI. Exécuter les cas privés dans un évaluateur isolé et publier uniquement des résultats agrégés et des références contrôlées. Les workflows qui traitent des PR non fiables ne doivent pas combiner code non fiable et jetons privilégiés, conformément aux [recommandations GitHub Actions](https://docs.github.com/en/actions/reference/security/secure-use).

### 5.7 Déploiement

- La fusion crée une version immuable identifiée par commit. Le mécanisme de distribution existant (clone, synchronisation, plugin, etc.) indique quand chaque utilisateur adopte cette version; la fusion seule ne prouve pas le déploiement.
- **Versionnement des agents** (semver, date ou commit) avec un `CHANGELOG` par agent. Enregistrer la version réellement chargée par la CLI dans chaque session.
- **Déploiement progressif :** pilote sur un groupe autorisé lorsque la distribution le permet, avant généralisation. Sinon, distribution par versions épinglées avec fenêtre de validation.
- **Retour arrière :** restaurer la version distribuée précédente; un revert Git peut ensuite corriger la branche. Tester cette procédure avec le mécanisme de distribution réel.
- **Mesure après déploiement :** comparer les métriques par type de tâche, population et version (section 8), en tenant compte des changements de volume et de modèle. Un gain hors ligne ne garantit pas un gain en usage réel.
- **Portée du changement :** une préférence d'un utilisateur peut rester dans sa configuration personnelle. Son passage aux instructions partagées exige des preuves couvrant les usages collectifs.

### 5.8 Orchestration et reprise

Chaque motif suit des états explicites : `observé → trié → cas validé → changement proposé → évalué → PR en revue → fusionné → distribué → mesuré`, avec des sorties `rejeté` ou `à approfondir`. L'état, la version des données et les raisons de décision sont enregistrés. Les tâches planifiées peuvent reprendre après un échec sans exporter deux fois la même session ni ouvrir deux PR pour le même motif.

Plafonner le volume de sessions analysées, le coût des évaluations et le nombre de PR par période. Garder un échantillon représentatif par agent et type de tâche; un plafond de coût ne doit pas éliminer systématiquement les cas rares mais graves. Les tâches qui échouent sont visibles dans un tableau d'exploitation avec propriétaire et action de reprise.

---

## 6. Schémas de données

### 6.1 Export de session normalisé (exemple après filtrage)

```json
{
  "schema_version": "1.0",
  "export_id": "exp-8cfe",
  "session_id": "sess-pseudo-123",
  "source_revision": "rev-2",
  "user_pseudonym": "user-pseudo-42",
  "collection_scope": "owner_opt_in",
  "workflow_id": "workflow-exemple",
  "agent": {
    "id": "code-refactorer",
    "definition_commit": "git-sha",
    "instructions_hash": "sha256"
  },
  "environment": {
    "os": "windows",
    "cli_version": "version-exacte",
    "model_id": "modele-et-version"
  },
  "started_at": "2026-11-06T15:00:00Z",
  "ended_at": "2026-11-06T15:04:05Z",
  "status": "completed",
  "outcome": { "label": "unknown", "source": "none" },
  "turns": [
    {
      "index": 0,
      "parent_index": null,
      "actor": "user",
      "event": "message",
      "content_ref": null,
      "tool": null
    }
  ],
  "usage": { "input_tokens": 400, "output_tokens": 180 },
  "redaction": { "level": 1, "status": "passed", "rules_version": "1.0" }
}
```

L'exemple est une instance, pas le JSON Schema complet. L'implémentation doit fournir ce schéma et rejeter les champs inconnus ou interdits. Au niveau 2, `content_ref` pointe vers un extrait filtré dans le magasin restreint; il n'inclut jamais directement le contenu dans l'index. La clé d'idempotence associe la source, la session, sa révision et la version des règles de filtrage : une session encore ouverte peut ainsi être réexportée sans doublon ni perte de mise à jour. Un résultat « success » exige une source vérifiable, distincte d'une simple fin de session.

### 6.2 Rapport d'analyse

```json
{
  "schema_version": "1.0",
  "period": { "from": "2026-11-01", "to": "2026-11-06" },
  "agent_id": "code-refactorer",
  "agent_commits": ["git-sha"],
  "sessions_analyzed": 24,
  "sessions_excluded": 3,
  "patterns": [
    {
      "id": "pattern-001",
      "type": "repeated_correction",
      "description": "Format de sortie ambigu",
      "independent_sessions": 4,
      "distinct_users": 2,
      "severity": "medium",
      "evidence_refs": [
        { "session_id": "sess-pseudo-123", "turns": [0, 1] }
      ],
      "root_cause_hypothesis": "Instruction de format imprécise",
      "alternative_causes": ["Changement de modèle"],
      "recommended_action": "Créer un cas d'évaluation et clarifier l'instruction",
      "risk": "low",
      "confidence": "medium"
    }
  ]
}
```

Le rapport sépare **observation**, **hypothèse** et **action recommandée**. Le diff appartient au générateur de changement, après le triage et la constitution d'un cas d'évaluation.

---

## 7. Signaux de friction à détecter

| Signal | Indice dans les sessions | Piste d'amélioration typique |
|---|---|---|
| Correction répétée | « non », « pas ça », « refais », reformulation de la même demande | Vérifier la correction; clarifier ou condenser une instruction si elle est fondée |
| Contexte redonné | Même information redonnée dans plusieurs sessions | Étudier une source de contexte autorisée ou une préférence personnelle, sans copier une donnée sensible dans un prompt global |
| Erreurs d'outils | Appels en erreur, reprises successives | Distinguer panne externe, schéma incorrect et mauvais choix d'outil |
| Refus de permissions | Outils refusés par l'utilisateur | Respecter le refus; proposer une voie de rechange ou demander plus tôt |
| Abandon | Session terminée sans résultat ou interrompue | Investigation manuelle; l'abandon seul ne prouve pas l'échec |
| Sessions anormalement longues | Tours ou durée élevés pour un même type de tâche | Chercher une boucle ou un transfert inefficace entre agents |
| Dérive de format | Réponse ne respectant pas un contrat vérifiable | Ajouter un cas d'évaluation et clarifier le contrat |
| Instruction ignorée | Règle présente mais non suivie | Tester sa priorité, sa longueur et ses contradictions |
| Échec de transfert | Sous-agent mal choisi ou résultat intermédiaire perdu | Clarifier le contrat de transfert et tester le workflow complet |
| Résultat métier négatif | Échec confirmé par un contrôle externe | Priorité élevée selon la gravité, même si le cas est isolé |

---

## 8. Métriques de suivi

| Mesure | Définition et usage |
| --- | --- |
| Succès vérifié | Tâches réussies / tâches avec résultat observable, par type de tâche et version; ne pas assimiler « session terminée » à « succès » |
| Correction utilisateur | Sessions avec correction vérifiée / sessions analysables; signal imparfait à revoir sur échantillon |
| Erreurs d'outils | Appels en erreur / appels, ventilés par outil et cause externe ou interne |
| Efficacité | Durée, tours, jetons et coût par tâche comparable, avec médiane et distribution |
| Évaluation hors ligne | Réussite par jeu et version, nombre de cas, incertitude, régressions critiques |
| Couverture de collecte | Sessions exportées / sessions éligibles, échecs de filtrage, décalage de version |
| Flux de PR | Propositions, acceptations, rejets et motifs, temps de revue, retours arrière |
| Taille des instructions | Longueur des définitions et contradictions détectées, pour suivre l'accumulation |

Comparer les métriques à population, type de tâche et version équivalents. Les avis utilisateurs, la durée et les corrections sont des indices; ils ne remplacent pas un résultat métier vérifié. Définir les seuils de décision avant chaque campagne et conserver les résultats négatifs.

---

## 9. Garde-fous (non négociables)

- Aucun merge automatique.
- Le coach analyse sans droit d'écriture. Le générateur de PR ne modifie que les artefacts textuels (prompts, instructions, skills textuels). Les permissions, outils autorisés, hooks, CI et définition du coach exigent un processus distinct et une revue renforcée.
- Filtrage avant export hors de l'environnement autorisé. Échec fermé de l'export en cas de doute, sans bloquer la session de l'utilisateur.
- Accès restreint, rétention approuvée, journal d'accès, procédure de retrait et de suppression lorsqu'applicable.
- Logs de session traités comme données non fiables (défense contre l'injection d'instructions).
- Accord de la sécurité et de la conformité avant toute collecte multi-utilisateurs.
- Information des utilisateurs et base d'autorisation validée pour chaque périmètre; le consentement est utilisé quand il est requis ou choisi par l'organisation.
- Aucune donnée brute de session, ni jeu d'évaluation sensible, dans GitHub ou les journaux CI.
- Une modification cohérente par PR, contrôles requis, propriétaire identifié et retour arrière documenté. [GitHub permet d'exiger des revues et des contrôles sur une branche protégée](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches).

---

## 10. Phasage et critères d'acceptation

### Phase 0 : faisabilité et collecte personnelle

- Identifier la CLI, ses capacités réelles, les magasins de sessions et le dépôt réel. Choisir un agent pilote, un résultat observable et une politique de données locale.
- Livrer un hook ou script d'export, nettoyage de base et stockage local protégé.
- **Acceptation :** une session de test produit un export conforme au schéma 6.1, attribué à la version exacte de l'agent; aucun secret de test n'y figure. Une relance ne crée pas de doublon, un échec de nettoyage interdit l'export, et un échec du hook n'affecte pas la session.

### Phase 1 : analyse manuelle

- Agent coach lancé à la main, produisant un rapport conforme au schéma 6.2.
- **Acceptation :** sur des sessions de test avec frictions connues et cas sans défaut, le coach identifie les motifs attendus, déduplique les reprises, signale les limites de preuve et distingue une panne externe d'un défaut d'instruction. Vérifier qu'une instruction injectée dans une session n'est pas exécutée.

### Phase 2 : évaluation comparative

- Constituer un premier ensemble de 10 à 30 tâches **pour vérifier le fonctionnement du banc d'essai**, plus des cas critiques. Ce volume ne suffit pas à lui seul à établir un gain général.
- Exécuter version actuelle et candidate dans un environnement isolé, sur les mêmes cas, avec contrôle des sorties d'outils et conservation des versions.
- **Acceptation :** une modification volontairement dégradante est détectée et bloquée; une modification neutre est correctement signalée; les résultats incertains sont distingués des résultats positifs. Une action d'outil simulée ne touche pas la production.

### Phase 3 : propositions par PR et CI

- Génération de diffs ciblés, préévaluation, ouverture de PR brouillon, évaluation complète en CI et revue du propriétaire de l'agent.
- **Acceptation :** une PR de test contient références de preuves, versions des évaluations, résultats avant/après, risques et retour arrière; la limite de PR ouvertes est respectée. La CI bloque une régression critique, les données privées restent hors des artefacts publics, et la branche protégée exige une revue.

### Phase 4 : multi-utilisateurs

- Politique approuvée, collecte par utilisateurs ou équipes autorisés, magasin central, contrôle d'accès, rétention, procédure de retrait et déploiement progressif.
- **Acceptation :** validation sécurité/conformité documentée; test de séparation des accès entre équipes; test de suppression et des copies dérivées; mesure de la couverture de collecte; retour arrière réussi sur le canal de distribution réel.

Chaque phase est livrée avec tests et documentation à jour avant de passer à la suivante.

---

## 11. Documentation à produire

Dans `docs/improvement-loop/` :

- `README.md` : vue d'ensemble, schéma de la boucle, état d'avancement par phase.
- `architecture.md` : composants, flux de données, décisions et alternatives écartées (ADR courts).
- `operations.md` : comment lancer l'analyse, interpréter un rapport, relire une PR, faire un rollback, dépanner la collecte.
- `privacy-and-compliance.md` : données collectées, règles de nettoyage, rétention, accès, opt-out, base légale/organisationnelle.
- `evals.md` : comment écrire un cas de test, ajouter un cas issu d'une session, interpréter les résultats.
- `contributing.md` : comment modifier le coach, les règles de nettoyage ou les seuils.
- `CHANGELOG.md` de la boucle elle-même.

Chaque script a un en-tête d'usage et un message d'aide. Les décisions non évidentes sont expliquées dans le code ou dans un ADR.

---

## 12. Questions ouvertes à clarifier avant d'implémenter

**Bloquantes pour la phase 0 :**

1. Quel est le dépôt réel, quelle CLI exécute les agents et quels agents pilotes peut-on utiliser?
2. Quelles classes de données des sessions personnelles l'organisation autorise-t-elle à exporter et lesquelles doivent rester locales?
3. Où les définitions d'agents et leur version sont-elles chargées réellement sur le poste pilote?

**À trancher avant les phases suivantes :**

4. Quels résultats métier peuvent être vérifiés sans jugement subjectif? Quels outils doivent être simulés?
5. Qui sont les propriétaires et relecteurs des agents? Quel mécanisme de distribution est en place?
6. Quel volume de sessions et quel budget de calcul permettent un triage et des évaluations utiles?
7. Pour plusieurs utilisateurs : quelle base d'autorisation, quelles équipes et données, quel stockage approuvé, quelles durées de conservation, quel retrait et quelle suppression des dérivés?
8. À quelle fréquence analyser les sessions et quels seuils de récurrence ou de risque sont pertinents?

---

## 13. Définition de « terminé »

- La boucle complète fonctionne sur au moins un agent, de la collecte à une PR évaluée et revue.
- Les garde-fous de la section 9 sont vérifiés par des tests ou des contrôles documentés.
- La documentation de la section 11 est complète et relue.
- Un retour arrière a été exécuté avec succès au moins une fois dans le canal de distribution de test.
- Les métriques de la section 8 sont calculables à partir des données collectées.
- Les liens entre session, version de l'agent, cas d'évaluation, PR et version distribuée sont consultables.

---

## 14. Prompt d'amorçage (à copier-coller avec ce document en contexte)

```text
Tu es un ingénieur logiciel senior spécialisé en outillage d'agents IA.

Contexte : ce document décrit une boucle d'amélioration continue d'agents textuels à partir
des sessions d'utilisation (collecte, filtrage, triage, évaluations, PR, déploiement).

Ta tâche : implémenter ce processus dans ce dépôt git, avec sa documentation.

Règles :
1. Explore le dépôt réel et les conventions existantes. Identifie la CLI, lis sa documentation
   officielle et vérifie par un essai ses hooks, exports, sessions et mode non interactif.
2. Consigne les écarts entre capacités observées et architecture proposée. Planifie la phase 0
   en indiquant l'agent pilote, les données autorisées, le schéma et les critères d'acceptation.
3. Regroupe les seules questions réellement bloquantes pour la phase 0 (section 12), puis
   poursuis les travaux indépendants de leurs réponses.
4. Respecte les sections 3 et 9 : aucun merge automatique, export bloqué si le nettoyage
   échoue, sessions traitées comme données non fiables, aucun contenu brut dans GitHub.
5. Implémente et vérifie la phase 0, avec tests pertinents et documentation à jour. Prépare
   la suite selon les phases 1 à 4 sans les implémenter avant validation du pilote.
6. Présente les fichiers livrés, les essais effectués, les limites et les décisions restantes.
```
