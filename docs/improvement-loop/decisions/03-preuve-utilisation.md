# Décision 03 — Prouver l'utilisation d'un skill ou d'un profil

**Statut :** décision d'architecture proposée, à valider sur Copilot CLI réel avant automatisation.

**Périmètre :** `SKILL.md` et `*.agent.md` distribués volontairement depuis « agent-repo » sous `~/.copilot`, puis utilisés dans plusieurs dépôts de code. « agent-repo » est un nom fictif; ce document ne constate aucun état de ce dépôt, des postes ou des règles internes.

## Décision

Conserver **quatre affirmations distinctes par artefact et session, puis par tour lorsque la trace le permet** : copie installée, définition résolue et disponible pour Copilot, invocation effective, influence sur la réponse. Chaque affirmation porte un **état**, un niveau de preuve `confirmé`, `déduit` ou `inconnu`, et des références vérifiables. Une preuve à une étape ne valide jamais automatiquement l'étape suivante. En particulier, `gh skill list` prouve au mieux la présence d'une copie; un nom de skill dans un prompt ne prouve pas son invocation; un événement d'invocation ne prouve pas que la réponse en a suivi les instructions.

La voie minimale est un inventaire local, des observations de la CLI et des hooks de session, complétés par une transcription **si son format est exploitable et autorisé**. Pour établir l'invocation d'un skill avec une source documentée, essayer en pilote l'événement OpenTelemetry (OTel) `github.copilot.skill.invoked`, exporté localement avec capture des messages désactivée. C'est **un événement de télémétrie**, pas un hook. Aucun événement de hook consacré aux skills n'est documenté dans la [liste officielle des hooks](https://docs.github.com/en/copilot/reference/hooks-reference). Le [référentiel CLI](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference) documente séparément l'événement OTel et son activation; sa présence, sa corrélation au tour et ses champs doivent être testés sur les versions déployées.

## Sens des quatre affirmations

| Affirmation | `confirmé` | `déduit` | `inconnu` |
| --- | --- | --- | --- |
| **Installation** | Présence ou absence de la copie vérifiée localement à un instant connu; empreinte de ses octets si présente. Provenance confirmée séparément, avec reçu d'installation ou métadonnées concordantes. | Une commande d'installation ou un manifeste existe, sans lecture de la copie actuelle. | Copie inaccessible ou état non vérifié. |
| **Chargement / résolution** | Dans cette session et ce dépôt, la CLI montre l'artefact découvert, activé et son origine effective, ou son indisponibilité; pour un skill disponible, cela ne signifie pas encore que le corps de `SKILL.md` a été injecté. | Inventaire et règles de priorité suggèrent la définition gagnante, sans observation de la CLI. | Confiance du dépôt, collision, changement en cours de session ou sortie CLI non déterminante. |
| **Invocation** | La trace du runtime identifie l'artefact résolu et l'activation sur le tour : événement OTel de skill validé, événement explicite de transcription validé, sélection effective du profil principal ou démarrage du sous-agent identifié. L'injection du corps du skill se produit ici. | Demande explicite `/nom-du-skill` ou `--agent`, appel d'un outil `skill`, ou comportement compatible, sans trace fiable de résolution et d'exécution. | Trace absente, ambiguë ou incompatible. L'absence d'événement n'établit pas l'absence d'invocation. |
| **Influence sur la réponse** | Un effet est établi **dans un essai comparatif contrôlé** avec versions, conditions et critère prédéfini; cela ne confirme pas la cause d'une réponse de production isolée. | Une réponse de production applique une consigne distinctive après invocation confirmée, avec causes concurrentes examinées. | Aucune comparaison ou consigne distinctive; réponse compatible avec plusieurs sources. |

**Précision de vocabulaire :** la documentation dit que le corps de `SKILL.md` est injecté quand Copilot *choisit d'utiliser* le skill; `/skills list` et `/skills info` décrivent donc la découverte et la disponibilité, pas cette injection. Pour un profil, l'affichage dans la liste des agents est distinct de sa sélection ou délégation. [Skills Copilot CLI](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-skills), [agents personnalisés](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/create-custom-agents-for-cli).

**Règle de publication :** `observed` dans l'exemple de schéma 6.1 de la proposition signifie **invocation confirmée par une trace du runtime**, liée à la bonne copie. `available_only` signifie seulement « disponible, invocation non observée avec les sources examinées », jamais « inutilisé ». Si la résolution ou la couverture des traces est insuffisante, écrire `unknown`. Ne jamais attribuer automatiquement à la copie personnelle le comportement d'une autre définition portant le même nom. Le niveau mesure la solidité de la preuve de l'**état déclaré** : « absence de copie confirmée » est possible; « invocation non observée » reste une observation, pas une négation d'usage.

## Sources de preuve et collecte recommandée

1. **À l'installation, puis au début de session :** depuis le dépôt de code courant, relever `gh skill list --agent github-copilot --json skillName,sourceURL,scope,version,pinned,path`, l'empreinte de `SKILL.md` et des ressources distribuées, et le reçu de pose du profil `*.agent.md` s'il existe. Le hash des fichiers réellement installés prévaut sur le tag ou le SHA annoncé par la source. `gh skill list` inventorie des installations, pas l'état interne de Copilot. Les profils exigent leur propre inventaire; `gh skill install --agent github-copilot` cible l'hôte du *skill*, pas l'installation d'un profil. [Inventaire `gh skill`](https://cli.github.com/manual/gh_skill_list), [installation `gh skill`](https://cli.github.com/manual/gh_skill_install).
2. **À la résolution :** consigner, sur un poste pilote, `/env`, `/skills info NOM`, `/agent` et `/instructions` (ou leur équivalent vérifié sur la version présente). Pour une collecte automatisée, `copilot skill list --json` donne `name`, `source`, `path`, `enabled` selon la [référence CLI](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference). Vérifier que cette sortie distingue réellement les définitions concurrentes; sinon conserver la résolution `déduit` ou `inconnu`. Ne pas exporter les chemins personnels bruts.
3. **Par hook personnel séparé des artefacts :** `sessionStart` donne `sessionId`, `cwd`, `source` et l'heure pour lier l'inventaire à la session; il peut aussi se déclencher lors d'une reprise. `agentStop` donne `transcriptPath` à la fin d'un tour; `sessionEnd` donne la raison de clôture, pas la transcription. `subagentStart`/`subagentStop` peuvent identifier les sous-agents personnalisés; `postToolUse`/`postToolUseFailure` donnent des métadonnées d'outil. `subagentStart` donne le nom de l'agent, **pas le chemin de son profil** : confirmer la copie exige un rapprochement indépendant avec la résolution effective. Ne conserver que les champs autorisés, jamais les arguments ou résultats par défaut. Les hooks `preToolUse`/`postToolUse` ne sont **pas** un contrat d'observation de toute invocation de skill. La documentation indique en outre que l'agent intégré `general-purpose` n'émet pas `subagentStart`/`subagentStop`. [Charges utiles et limites des hooks](https://docs.github.com/en/copilot/reference/hooks-reference).
4. **Pour l'invocation de skill, si le pilote le confirme :** activer l'export OTel **vers un fichier local protégé** avec `COPILOT_OTEL_FILE_EXPORTER_PATH` et laisser `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=false`. Extraire seulement le nom, la référence de définition si présente, l'identifiant de session, la référence de trace/tour et l'horodatage de `github.copilot.skill.invoked`. Apparier le chemin local à l'inventaire local avant pseudonymisation; ne transmettre ni chemin brut ni contenu du skill. Ne pas présumer qu'un événement OTel contient à lui seul le hash de la copie ou un identifiant de tour stable : vérifier la corrélation avec `gen_ai.conversation.id`, les spans et, si disponible, `github.copilot.turn_id`. [OTel Copilot CLI](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference).
5. **Transcription en recours ou complément :** lire localement le fichier indiqué par `transcriptPath` ou `~/.copilot/session-state/<session>/events.jsonl` seulement après inspection du format sur la version pilote. Si un événement explicite d'invocation, son chemin et son tour sont observés et validés, il peut confirmer cette invocation; un simple texte qui cite un nom ou lit un fichier n'y suffit pas. Isoler le parseur par version, rejeter les événements inconnus pour l'attribution, et nettoyer avant tout export. Le stockage de session est documenté, mais son format d'événements n'est pas présenté comme un contrat d'intégration stable. [Répertoire de session](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-config-dir-reference), [données de session](https://docs.github.com/en/copilot/concepts/security-governance-and-network-settings/session-data).

Le hook de collecte sort sans modifier la réponse, les permissions ou les outils. Une panne d'inventaire, d'OTel ou de parseur baisse le niveau de preuve et incrémente un compteur local de perte; elle ne bloque pas Copilot. Un export dont le filtrage échoue est refusé. Une file locale protégée peut différer l'analyse pour maintenir un temps de hook borné. Les hooks et l'OTel sont deux canaux à rapprocher par session et par tour, sans supposer qu'ils partagent toujours un identifiant de tour exploitable.

## Résolution des collisions et identité des copies

L'identité d'un artefact ne se réduit pas à son nom : conserver `kind + id + source_scope + hash_de_la_copie + intervalle_d'observation`. Sur les versions documentées, un skill de projet du même nom prime sur le skill personnel; pour les agents de même **ID de fichier**, le profil personnel prime sur celui du projet, et le champ frontmatter `name` ne décide pas de la déduplication. Des racines ajoutées, plugins et définitions distantes peuvent aussi introduire des homonymes. Enregistrer toutes les définitions candidates localement et la gagnante observée, puis n'exporter que leurs identifiants pseudonymisés et leurs hashes nécessaires. [Ordre des skills et agents](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference), [répertoire de configuration](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-config-dir-reference).

Une copie personnelle présente dans `~/.copilot` peut donc être **installée confirmée** et **non résolue pour cette session**. Le dépôt de code courant, `COPILOT_HOME`, les répertoires additionnels, la désactivation, la confiance du dossier, les reprises et `/skills reload` font partie du contexte de résolution. Si la copie change entre inventaire et invocation, ou si le même nom se résout différemment après rechargement, ouvrir un nouvel intervalle d'observation; sans empreinte au moment utile, rétrograder l'attribution à `inconnu`. Un hash de l'agent-repo source ne remplace jamais celui de la copie locale.

## Schéma de preuve à ajouter à l'export 6.1

Le schéma ci-dessous est une **recommandation d'extension**, non une affirmation sur un format Copilot existant. Une entrée par artefact, intervalle de résolution et tour lorsque celui-ci est identifiable; sinon `turn_ref: null`. Plusieurs entrées sont possibles dans une même session. L'effet comparatif, s'il est établi, vit dans un enregistrement d'évaluation distinct, lié par `comparison_ref`, et ne se propage pas aux réponses de production.

```json
{
  "schema_version": "usage-evidence/1",
  "session_ref": "sess-pseudo-123",
  "execution_repository_ref": "repo-pseudo-7",
  "turn_ref": "tour-2-ou-null",
  "parent_agent_ref": null,
  "observed_from": "2026-11-06T15:00:00Z",
  "observed_to": "2026-11-06T15:04:00Z",
  "environment": {"copilot_version": "à-relever", "gh_version": "à-relever", "os_context": "windows-ou-wsl", "collector_version": "1"},
  "artifact": {
    "kind": "skill",
    "id": "exemple",
    "source_scope": "user",
    "resolved_source": "user",
    "source_ref": "tag-ou-sha-ou-null",
    "source_provenance_level": "déduit",
    "installed_definition_sha256": "sha256:...",
    "installed_tree_sha256": "sha256:...",
    "candidates": [{"scope": "user", "definition_sha256": "sha256:...", "resolution": "winner"}],
    "collision": false
  },
  "claims": {
    "installed": {"state": "present", "level": "confirmé", "reason": "copy_hashed", "evidence_refs": ["ev-1"]},
    "loaded": {"state": "available", "level": "confirmé", "reason": "cli_enabled_resolved", "evidence_refs": ["ev-2"]},
    "invoked": {"state": "yes", "level": "confirmé", "reason": "runtime_skill_event", "evidence_refs": ["ev-3"]},
    "influenced": {"state": "unknown", "level": "inconnu", "reason": "no_controlled_comparison", "comparison_ref": null, "evidence_refs": []}
  },
  "evidence": [
    {"ref": "ev-1", "source": "local_inventory", "event": "copy_hash", "time": "2026-11-06T15:00:00Z", "digest": "sha256:..."},
    {"ref": "ev-2", "source": "copilot_cli", "event": "skill_list", "time": "2026-11-06T15:00:01Z", "digest": "sha256:..."},
    {"ref": "ev-3", "source": "otel", "event": "github.copilot.skill.invoked", "time": "2026-11-06T15:01:00Z", "digest": "sha256:..."}
  ],
  "coverage": {"hook_seen": true, "otel_seen": true, "transcript_parseable": null, "turn_link_verified": true, "content_export_level": 0}
}
```

`evidence_refs` pointe vers des événements locaux normalisés et filtrés; `digest` identifie la charge locale sans la recopier. Les champs de chemin, texte de prompt, réponse et contenu de skill restent hors de l'index. Le hash d'arbre est calculé sur les chemins relatifs triés et les octets des fichiers distribués, avec une convention versionnée; celui de la définition cible le fichier `SKILL.md` ou `*.agent.md`. `source_ref` vaut `null` si la provenance n'est pas vérifiée. `turn_ref` vaut `null` si la corrélation session/tour est incertaine. Les états attendus sont `present|absent|unknown` pour l'installation, `available|unavailable|unknown` pour le chargement, `yes|not_observed|unknown` pour l'invocation et `yes|unknown` pour l'influence; `not_observed` ne signifie pas `no`. Les raisons sont des valeurs de schéma, notamment `copy_hashed`, `cli_enabled_resolved`, `runtime_skill_event`, `profile_selected`, `explicit_request_only`, `ambiguous_collision`, `trace_missing`, `copy_changed`, `no_controlled_comparison`.

Les niveaux concernent **chaque affirmation**, pas une confiance globale de la session. L'export 6.1 peut garder `usage_state` pour compatibilité, mais le dériver uniquement de `claims.invoked` et préserver `coverage` et les raisons. Une requête analytique doit pouvoir filtrer les preuves confirmées sans assimiler `inconnu` à `false`.

## Vérification sur Copilot CLI réel, par phases

**Phase 0 — faisabilité personnelle.** Sur un poste équipé dont la version de `copilot` et de `gh` est consignée, choisir un skill et un profil de test, installés personnellement, et **deux dépôts de code distincts**, dont un avec instructions locales. Créer des tâches sans données sensibles et relever la vérité terrain par inspection manuelle : copie et hashes, `/env`, `/skills info`, `/agent`, hooks reçus, éventuel événement OTel, contenu/format de `transcriptPath`. Tester un tour sans skill, une invocation explicite `/skill`, une tâche susceptible d'activer le skill automatiquement, un profil principal via `--agent` ou `/agent`, une délégation au profil, puis une reprise de session. L'activation automatique est probabiliste : son absence dans un essai n'est pas un échec du collecteur. Ne pas utiliser un marqueur dans la réponse comme seule preuve d'invocation. [Commandes de skills](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-skills), [option `--agent`](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-programmatic-reference).

**Phase 0 — collisions et pertes.** Dans le deuxième dépôt, ajouter temporairement un skill de projet homonyme et un profil de projet avec le même ID que la copie personnelle; vérifier les priorités différentes, la provenance effectivement retenue et l'absence d'attribution au mauvais hash. Tester un skill désactivé, un changement de copie suivi de `/skills reload`, un hook absent/désactivé et un arrêt brutal. Exiger que toute lacune de preuve devienne `inconnu` et que le journal local mesure la perte, sans contenu de session. Les tests se font dans des fixtures isolées, sans modifier les artefacts de travail des développeurs.

**Phase 1 — normalisation.** Stabiliser les adaptateurs de transcription et d'OTel sur les seules versions observées, dédupliquer les événements répétés et relier session, tour, acteur principal/sous-agent et intervalle d'inventaire. Vérifier sur un échantillon relu manuellement qu'il n'existe **aucun faux `invoked=confirmé`** dans les cas sans invocation et de collision; publier aussi le taux de `inconnu`, par version et canal. Si OTel est absent ou non autorisé, ne pas relever artificiellement le niveau via un mot-clé ou un appel d'outil ambigu.

**Phase 2 — influence.** Dans des environnements Copilot isolés et des dépôts fixtures identiques, comparer de façon répétée une copie candidate à une copie témoin, avec modèle, consigne, outils et critères fixés avant l'essai. Prévoir des cas positifs où une consigne distinctive, bénigne et exclusive à l'artefact doit produire un résultat observable, ainsi que des cas neutres et de collision. Annoncer les effectifs, échecs et variabilité. Ce test établit au mieux un effet dans ces conditions; il ne justifie pas d'étiqueter `influenced=confirmé` les réponses de production passées.

**Phase 4 — extension éventuelle.** N'agréger les preuves de plusieurs postes qu'après définition organisationnelle du périmètre autorisé, des catégories exportables, du stockage, de la rétention, de l'accès et du retrait. Mesurer la couverture parmi les postes équipés et consentis/autorisés, en séparant sessions sans trace, artefacts seulement installés et invocations confirmées. La synchronisation native des sessions avec le compte GitHub personnel ne donne pas à elle seule un inventaire d'équipe. [Données de session Copilot](https://docs.github.com/en/copilot/concepts/security-governance-and-network-settings/session-data).

## Limites, faux positifs et risques

- Un nom dans le prompt, une réponse, `/skills list`, un chemin lu par un outil ou une ligne `gh skill list` peut ressembler à un usage sans injection du skill. Un profil présent dans `/agent` peut n'avoir jamais été sélectionné.
- Un événement de skill sans chemin ou avec homonyme, un `--agent` qui se résout vers un autre ID, une modification locale après inventaire, une reprise, une compaction ou une trace tronquée imposent une attribution prudente. `subagentStart` ne couvre pas l'agent intégré `general-purpose`; la couverture des autres événements doit être mesurée, pas présumée.
- Une instruction identique peut venir du skill, du profil, d'instructions personnelles ou de projet, du prompt ou du modèle. La conformité de la réponse ne permet donc pas d'identifier seule la cause. Même une comparaison contrôlée reste sensible à la variabilité du modèle et aux outils.
- OTel et les transcriptions peuvent exposer des chemins, identifiants et contenus sensibles; la capture intégrale des messages OTel reste désactivée pour ce besoin. Les hooks peuvent recevoir prompts et résultats d'outils : sélection stricte des champs, filtrage local avant export, aucun contenu brut dans GitHub ou la CI. L'export OTel en fichier ajoute un magasin local à protéger et à purger.
- Les commandes et formats observés aujourd'hui peuvent évoluer. Une mise à jour de `copilot` ou `gh` invalide la matrice de preuve jusqu'à un test de compatibilité; les sources officielles décrivent une capacité, pas son activation sur un poste donné.

## Choix restant à l'organisation

1. Quel skill et quel profil réels serviront de pilotes, par quel mécanisme et à quelles versions sont-ils installés, et quels deux dépôts de code représentent les usages visés ?
2. Les métadonnées OTel locales, les hashes et les extraits de transcription sont-ils autorisés, pour quels utilisateurs/dépôts, avec quelle rétention et quel contrôle d'accès ? Le niveau 0 de la proposition suffit-il au pilote ?
3. Quel taux d'`inconnu` est acceptable pour le triage, et l'organisation veut-elle financer/autoriser l'essai comparatif nécessaire avant d'affirmer une influence ?

Ces choix n'empêchent pas de définir le schéma et les essais de phase 0. Le présent document n'affirme pas que ces essais ont été exécutés : `copilot` n'est pas disponible dans cet environnement de rédaction, et aucun poste pilote ni vrai « agent-repo » n'a été inspecté.
