# Étude comparative — SAP CTS/TMS, SAP gCTS et abapGit

**Sujet :** intérêt réel de Git pour le versionnement, la revue du code et le transport des objets ABAP dans un paysage **DEV → QUAL → PROD**  
**Date :** 29 octobre 2026  
**Nature :** note d'analyse technique et d'aide à la décision  
**Périmètre supposé :** un système ABAP de développement partagé, un système de qualification et un système de production ; transports Workbench et Customizing ; pas de système de développement parallèle disponible.

> **Précision terminologique importante.** Le sigle « SAP CTM » peut prêter à confusion. Pour les transports ABAP classiques, l'appellation SAP est **CTS** (*Change and Transport System*), administré notamment avec **TMS/STMS**. Ce document compare principalement **CTS/TMS**, **gCTS** et **abapGit**. Il traite également **SAP Cloud Transport Management (cTMS)**, au cas où « CTM » désignerait ce service cloud. Le produit SAP « CTM » (*Capable-to-Match*), lié à la planification de la chaîne logistique, est sans rapport avec les transports de code.

## 1. Synthèse exécutive

**Conclusion : dans un paysage SAP ABAP DEV → QUAL → PROD à développement partagé et sans véritable possibilité de paralléliser les versions actives, mettre les objets dans Git n'apporte pas automatiquement un meilleur mécanisme de transport.**

- **CTS/TMS** demeure la solution de référence pour livrer un périmètre SAP complet et gérer les imports dans l'ordre attendu, notamment lorsqu'il inclut des objets particuliers, du Customizing, des corrections SAP et des ajustements de modifications.
- **gCTS** constitue une **alternative de distribution** pour des objets éligibles : les demandes de transport restent présentes dans DEV, mais les versions sont conservées sur Git et les systèmes cibles peuvent importer l'état représenté par un commit. Il apporte de l'historique, des diff, des possibilités de revue et d'intégration CI/CD. **Il ne supprime ni les verrous d'objets ABAP ni le besoin de contrôler les imports.**
- **abapGit** est particulièrement intéressant comme **outil complémentaire** : exporter du code, le relire via des pull requests, partager des bibliothèques, comparer des versions ou faire migrer des objets. Sa couverture fonctionnelle n'en fait pas, à lui seul, un remplaçant universel de CTS/TMS.
- **Le rollback est le point critique.** gCTS permet de sélectionner un commit antérieur et de calculer les différences à importer, mais SAP recommande explicitement de réserver le rollback d'import aux environnements de test et de qualité, **pas à la production**. Revenir sur un commit Git ne restaure pas nécessairement les données métiers, les schémas de base, les effets de migrations ou les traitements exécutés depuis le déploiement.
- **La promesse « Git = branches = développements parallèles » ne tient pas dans un DEV partagé** : un objet ABAP n'y possède qu'une version active et ses verrous restent gérés par SAP. Les branches existent techniquement, mais alterner entre elles peut imposer de libérer les travaux en cours, d'interrompre des développements ou d'avoir des systèmes DEV/COR distincts.

**Avis proposé :** conserver **CTS/TMS comme mécanisme principal de promotion**, tester **abapGit pour la revue du code et la traçabilité**, et réserver **gCTS à un pilote sur un périmètre autonome et explicitement supporté**. Un remplacement généralisé des transports par Git ne paraît justifié que si la couverture, les scénarios de correctifs, l'exploitation et la reprise sur incident ont été démontrés par un POC. [S1][S2][S3][S4][S5]

## 2. Que font réellement ces technologies ?

### 2.1 CTS/TMS classique

Le système **CTS** enregistre les modifications effectuées dans les outils SAP dans des tâches et demandes de transport ; **TMS/STMS** organise la circulation et l'import entre systèmes. Le processus usuel est :

```text
DEV : modifier et activer des objets
      ↓
      créer / alimenter / libérer une demande de transport
      ↓
QUAL : importer le transport, exécuter les tests, accepter / refuser
      ↓
PROD : importer le transport approuvé
```

**Atouts :** mécanisme standard de la plateforme ABAP ; large couverture des objets de transport ; import et activation via les outils SAP ; exploitation, journaux et gestion des dépendances familiers aux équipes Basis. **Faiblesses :** pas de pull request Git native ; comparaison transversale de code moins ergonomique ; retour arrière souvent réalisé par un nouveau transport correctif ; orchestration des dépendances et du séquencement à maintenir.

### 2.2 gCTS : Git-enabled Change and Transport System

**gCTS ne remplace pas l'environnement de développement ABAP.** Le code est toujours modifié et activé dans DEV, les objets continuent à être affectés à des tâches/demandes et à être verrouillés. Dans le flux standard, la libération d'une demande entraîne la création d'un **commit Git** dans le dépôt concerné ; le commit est poussé sur le serveur Git si la configuration le prévoit. Un système QUAL ou PROD récupère ensuite le commit sélectionné : gCTS reconstruit une demande d'import correspondant aux différences et fait intervenir les outils SAP de transport et d'activation. Une intégration BAdI peut permettre une granularité à la **tâche** plutôt qu'à la demande. [S2][S3][S6]

```text
                       ┌─────────────── Git central ───────────────┐
DEV ABAP               │ historique, commits, diff, revue, CI/CD  │
objets + verrous       │                                            │
TR libérée ── commit ─►│                                            │
                       └────────────┬────────────────┬──────────────┘
                                    │                │
                              pull commit       pull commit
                                    ▼                ▼
                               QUAL ABAP         PROD ABAP
                              import / tests    import contrôlé
```

**Point subtil :** un commit correspond à une version de l'ensemble géré par le dépôt, **pas à une archive de transport classique interchangeable** avec le fichier CTS. L'import d'un commit calcule les écarts par rapport à la version déjà présente sur le système cible. L'import peut agréger plusieurs commits intermédiaires, d'où la nécessité de savoir précisément **quel ensemble fonctionnel** a été validé en QUAL. [S2][S6]

### 2.3 abapGit

**abapGit** sérialise des objets ABAP compatibles dans des fichiers texte et peut les exporter vers un dépôt Git ou les importer depuis celui-ci. Il facilite les comparaisons, le partage de code et les *pull requests*. La version communautaire est open source ; la variante intégrée dans certains environnements ABAP cloud a ses propres capacités et restrictions. [S7][S8]

Deux modes d'usage doivent être distingués :

1. **Complément de CTS (à privilégier pour le cas étudié)** : les objets continuent à être livrés par STMS. Git est un reflet du code servant à la revue, à l'audit, à la comparaison et éventuellement au contrôle qualité. Une branche Git liée à un transport peut servir de support à une pull request, mais **ne constitue pas une deuxième version exécutable indépendante dans DEV**.
2. **Import/export via abapGit** : code extrait de Git puis importé dans un système ABAP. Ce flux peut convenir à des composants autonomes, bibliothèques, extensions réutilisables ou migrations ; il ne couvre pas automatiquement tous les objets, le séquencement et les opérations d'une chaîne de transports ERP.

**Attention :** gCTS et abapGit n'utilisent pas nécessairement un format de dépôt compatible ; **partager le même dépôt comme s'il était interchangeable entre les deux outils n'est pas pris en charge**. SAP documente une procédure de migration par import d'objets puis nouvel enregistrement avec gCTS, pas une exploitation conjointe du même dépôt. [S9]

### 2.4 Et SAP Cloud Transport Management (cTMS) ?

**cTMS** est un service SAP BTP qui gère des **routes/nœuds, files d'attente et promotions d'artefacts cloud**. Pour **SAP BTP ABAP environment**, un transport cTMS peut référencer un composant logiciel et un commit Git ; le système cible récupère ensuite la version correspondante. **Ce service n'est pas la même chose que CTS/STMS classique ni que gCTS** : il orchestre notamment des livraisons cloud, tandis que gCTS assure le stockage et la distribution d'objets ABAP via Git. L'intégration et les types de contenu dépendent du produit cible. La documentation SAP actuelle liste explicitement **BTP ABAP** parmi les types pris en charge ; cela n'autorise pas à déduire qu'il remplace les transports ABAP d'un S/4HANA on-premise. [S10][S11]

## 3. Comparatif opérationnel

| Dimension | CTS/TMS classique | gCTS | abapGit, en complément de CTS |
|---|---|---|---|
| **Transport DEV → QUAL → PROD** | Natif, principal | Possible sur objets admissibles ; promotion par commit | Réalisé par CTS si abapGit est complémentaire |
| **Enregistrement des modifications** | Tâches et demandes SAP | Tâches et demandes SAP **conservées** | Export Git volontaire ou automatisé ; CTS reste distinct |
| **Version active d'un objet dans DEV** | Une | **Une** | **Une** |
| **Verrous ABAP** | Oui | **Oui, inchangés** | Restent en place dans SAP |
| **Historique/diff Git** | Non natif | Oui, sur le contenu sérialisé | Oui, pour les objets exportables |
| **Revue via pull request** | À intégrer autrement | Possible via Git/pipeline, mais la libération SAP et la validation Git doivent être articulées | Bon cas d'usage ; peut être organisée avant libération du transport |
| **Branches parallèles dans un seul DEV** | Non | Branches Git possibles, **mais pas deux versions actives simultanées** | Branches de revue possibles ; pas de développement ABAP simultané indépendant |
| **Couverture des objets** | Référence pour objets CTS compatibles | Très large, **avec restrictions selon version/objets/scénarios** | Limitée aux sérialiseurs et types pris en charge |
| **Customizing** | Natif selon objets, clients et transports | Oui sur versions récentes, sous réserves précises | Partiel / dépend du type et de la stratégie choisie |
| **Notes SAP / ajustements de modifications** | Oui, selon scénario | **Ne remplace pas le CTS classique** pour ces demandes | Ne remplace pas CTS |
| **Retour à une version antérieure** | Correctif, réimport et/ou restauration selon incident | Pull d'un commit antérieur techniquement possible ; **déconseillé en PROD par SAP** | Revenir sur Git ne restaure pas automatiquement l'exécution SAP |
| **Reprise des données métiers** | Non automatique | **Non automatique** | **Non automatique** |
| **Intégration CI/CD** | Possible avec outils complémentaires | Facilitée par commits, API et outils de pipeline | Facilitée pour les contrôles de code exporté |
| **Complexité d'exploitation** | Connue des équipes Basis | Git + droits + dépôts + imports + règles de release | Gestion d'un reflet Git + synchronisation avec transports |
| **Statut de support** | Standard SAP | Standard SAP, selon produit/version | Projet communautaire pour la version classique ; support distinct pour intégrations SAP |

**Lecture :** Git apporte surtout une **couche de visibilité et de contrôle de qualité**. L'économie opérationnelle d'un basculement total vers gCTS dépend d'éléments non résolus par Git : couverture, dépendances, séquencement, autorisations, arrêt des traitements, Customizing et retour arrière. [S1][S3][S7][S8][S12]

## 4. Limites décisives : tous les objets sont-ils exportables ?

### 4.1 abapGit : l'exportation est conditionnée par le type d'objet

La documentation abapGit publie une **liste de types supportés**, variable selon la version d'abapGit et la version ABAP. Des objets courants (classes `CLAS`, programmes `PROG`, tables `TABL`, CDS `DDLS`, éléments DDIC, etc.) sont pris en charge ; d'autres ont des restrictions, demandes d'évolution ou traitements particuliers. Le transport de **données de tables / Customizing** nécessite une approche adaptée (`TABU`, BC Sets ou fonction d'export de données selon contexte) et ne se substitue pas de manière générale à l'ensemble des transports de configuration. [S7][S8]

**Conséquence :** si un développement « métier » comprend du code exportable **et** du contenu non exportable, Git ne fournit qu'un **état partiel du livrable**. Une pull request « verte » ne prouve donc pas que tout ce qui sera importé en production a été revu ni qu'il pourra être restauré à partir de Git.

### 4.2 gCTS : couverture plus large, mais pas universelle

SAP présente gCTS comme prenant en charge **presque tous** les objets de développement et de Customizing, mais renvoie explicitement à la **SAP Note 2888887** pour les restrictions de types ; la **Note 2821718** rassemble également des contraintes gCTS. Le périmètre varie avec la release. [S3][S12]

Exemple historique concret : SAP a ajouté dans ABAP Platform 2020 le support de `VDAT`, `TDAT` et de certaines suppressions `TABU`, tout en indiquant à cette date que `CDAT` n'était pas pris en charge. **Ce point est un exemple de variabilité temporelle, pas une assertion d'incompatibilité sur toutes les versions actuelles** : une liste cloud actuelle peut présenter d'autres capacités. [S13][S14]

Autres limites structurantes :

- **Notes SAP et ajustements après Support Package / modifications SAP** : les transports correspondants nécessitent un paysage CTS classique ; gCTS ne s'y substitue pas. [S12]
- **Customizing dépendant du mandant** : il faut vérifier le mandant cible et l'import ; SAP documente que l'import simultané de Customizing dans plusieurs mandants n'est pas fourni par défaut. [S15]
- **Première capture d'un objet partiel** : l'initialisation de certains objets requiert leur version complète dans Git avant des exports partiels ultérieurs. [S16]
- **Format et lisibilité** : être présent dans Git ne signifie pas que chaque donnée de Customizing sera facile à examiner sous forme de diff métier. Une discussion SAP Community relève précisément cette difficulté. [S17]
- **Dépendances et objets générés** : elles doivent être vérifiées dans les imports et contrôles fonctionnels, même si la sérialisation réussit.

### 4.3 Règle d'architecture recommandée

Ne pas alterner **au fil des changements d'un même objet** entre les mécanismes CTS et gCTS. SAP indique que les deux peuvent coexister dans un paysage, mais que le mode de traitement doit être décidé de façon stable pour un objet ou un ensemble cohérent d'objets, notamment via les couches de transport. [S6]

**Critère de blocage pour un remplacement complet :** au moins un objet ou une catégorie indispensable à une livraison métier typique ne peut pas passer de manière fiable par gCTS **et** l'entreprise ne dispose pas d'une procédure de livraison mixte, ordonnée, testée et auditée.

## 5. Branches Git et verrous ABAP : pourquoi le gain est limité dans un DEV unique

La distinction principale est entre **branche Git** et **version de travail active dans SAP** :

- Une branche Git est un historique de commits. Plusieurs branches peuvent donc coexister **dans le dépôt**.
- Dans un **système ABAP donné**, il n'y a qu'une **version active** d'un objet. Le verrouillage par les tâches/demandes subsiste avec gCTS. Deux développeurs ne peuvent pas modifier simultanément deux branches différentes **du même objet actif dans le même DEV**, comme avec deux espaces de travail locaux Git sur des postes indépendants. [S2][S6]
- Basculer la branche active dans un système peut **modifier le contenu ABAP déployé dans ce système** ; ce n'est pas seulement changer de référence Git. [S6]

### Exemple : fonctionnalité en cours + urgence de production

**Situation :** une équipe développe la fonctionnalité **A** dans DEV ; un incident de PROD exige une correction **B** sur une version antérieure du même objet.

**Avec un seul DEV** :

1. Le code A est potentiellement déjà actif dans le système de développement ; un transport ouvert peut verrouiller l'objet.
2. Une branche Git « correctif B » ne crée pas automatiquement une deuxième copie exécutable de cet objet dans DEV.
3. Pour développer B sur une base stable, il faut **isoler ou interrompre A**, basculer de contexte avec précautions, libérer ou mettre en sécurité les travaux en cours, puis **réappliquer B** sur la ligne de développement principal.
4. Cette manipulation comporte un coût, des risques de conflit et peut perturber les autres développeurs.

SAP décrit officiellement, pour un paysage cloud à **une ligne de code et trois systèmes**, la nécessité de **mettre en pause les développements**, de libérer les transports ouverts, de basculer sur une branche de release pour le correctif urgent, puis de réintégrer la correction dans `main`. Pour le cas on-premise, SAP précise aussi que la limitation d'une version active par objet reste inchangée et que des paysages DEV/COR séparés demeurent souvent nécessaires. [S18][S19]

**Conclusion :** dire « gCTS ne permet aucune branche » serait inexact ; dire « gCTS règle le développement parallèle dans notre DEV partagé » serait également inexact. **Dans le contexte posé, la valeur d'une stratégie de branches est réduite**, sauf à créer des systèmes de développement réellement isolés, à limiter les branches à la validation de releases, ou à accepter des interruptions contrôlées.

## 6. Retour arrière : distinguer trois opérations

| Opération | CTS/TMS | gCTS | Rétablit les données métier ? |
|---|---|---|---|
| Annuler une modification **non encore livrée** | Modifier/revenir à la version antérieure puis transporter | Changer/revenir au code dans DEV ; gérer les commits | Non pertinent si aucun effet métier |
| Remettre **un ancien état du code** en QUAL | Nouveau transport ou procédure contrôlée | Import possible d'un commit plus ancien | **Non**, sauf traitement spécifique |
| Annuler **un déploiement en PROD** avec tous ses effets | Correctif en avant, plan de restauration / reprise | **Pas de rollback gCTS en PROD recommandé par SAP** ; correctif en avant ou reprise d'exploitation | **Non**, ni avec Git ni avec gCTS |

SAP précise que la fonction de rollback d'import gCTS est destinée aux systèmes **test/qualification**, **pas à PROD**. Le rollback n'est donc ni un mécanisme de sécurité automatique universel ni une justification suffisante d'adoption. [S1]

Exemples où « revenir au commit N-1 » est insuffisant :

- un transport a modifié une structure de table et des données ont été écrites sous le nouveau modèle ;
- un programme a déjà créé des écritures métiers, des fichiers, des messages ou des appels externes ;
- du Customizing dans plusieurs mandants et un objet ABAP ont changé de concert ;
- un objet absent de Git a aussi été transporté par CTS ;
- plusieurs livraisons se sont succédé et un retour global supprimerait des correctifs valides.

La question d'exploitation doit être : **« Comment restaurer un état applicatif cohérent, y compris les données et les changements intermédiaires ? »** et non seulement « Comment faire un `git revert` ? ».

**Procédure de production recommandée :** geler le déploiement, identifier le périmètre et l'état exact, établir un plan de correction **forward fix** (nouvelle version transportée), réserver les actions de restauration système/base de données à des procédures Basis/DBA validées, puis documenter les vérifications fonctionnelles. Une éventuelle automatisation de pipeline doit être testée et bornée ; elle n'efface pas les risques applicatifs.

## 7. Git apporte-t-il une valeur mesurable sans remplacer les transports ?

### Valeur souvent réelle

1. **Revue de code avant mise en production :** diff lisible, commentaires, approbation par pair, audit de type « quatre yeux ». Si l'entreprise exige une validation avant **libération du transport**, un export anticipé ou une intégration dédiée est nécessaire : **un commit gCTS créé à la libération n'est pas, à lui seul, une barrière de validation préalable**.
2. **Traçabilité étendue :** historique du code, rapprochement entre demande métier, transport, commit, revue et contrôle ATC.
3. **Contrôles automatiques :** déclenchement de pipelines et de checks sur les commits, à condition de disposer de l'infrastructure et d'un SAP où exécuter les validations qui l'exigent.
4. **Partage de bibliothèques et migrations :** avantage notable d'abapGit pour exporter/importer un composant entre environnements.
5. **Visibilité transverse aux équipes :** comparaison de sources sans navigation objet par objet dans les outils classiques.

### Valeur faible, voire négative, si…

- Git est **une copie manuelle non synchronisée** du contenu réellement transporté ;
- la majorité des modifications relèvent de Customizing ou d'objets difficilement exportables ;
- aucune équipe ne réalise de revue, de diff ou de contrôle CI malgré le dépôt Git ;
- les approbations et le suivi existants dans CTS/ChaRM/Cloud ALM satisfont déjà les exigences ;
- il faut maintenir **deux chaînes de promotion concurrentes** sans cartographie des objets et sans règles de dépendance ;
- le coût d'administration (Git, authentification, droits, pipelines, incidents de sérialisation/import, formation) dépasse les bénéfices constatés.

**Approche pragmatique :** si l'objectif majeur est **la revue et l'audit**, il n'est pas nécessaire de bouleverser la chaîne de transport. Un **miroir Git du code** synchronisé avec CTS peut répondre au besoin avec un risque de migration plus faible.

## 8. Retours d'expérience et discussions SAP Community : adoption réelle ou discours théorique ?

L'analyse doit distinguer les **témoignages d'utilisation**, les **démonstrations techniques** et la **documentation produit**. Une publication de tutoriel ne constitue pas la preuve d'une industrialisation sur un ERP complet.

### 8.1 abapGit : usages d'entreprises documentés, souvent complémentaires

**Liste d'organisations utilisatrices.** Le projet abapGit publie une page « Who Uses abapGit? » citant notamment **AB-InBev, bonprix, Endress+Hauser, Hansgrohe, Haufe, Linde Engineering, Queensland Health et RheinEnergie**. Cette page est un **indice public d'adoption**, pas un inventaire vérifié de leur utilisation en production en 2026, ni de leur architecture de transport. [S20]

**Cas détaillé : revue de code au-dessus de CTS.** Un article SAP Community, « *How we do ABAP code review using abapGit* », décrit un flux concret : sérialisation du code, création d'une branche Git liée au transport, pull request de revue, **blocage de la libération du transport en l'absence d'approbation**, puis fusion Git lors du transport/import. L'article décrit donc une **gouvernance du code qui conserve le transport SAP**. C'est précisément le modèle le plus pertinent pour un DEV partagé. Il s'agit néanmoins d'un témoignage technique et non d'une mesure universelle de ROI. [S21]

**Autre cas nommé : SAP IBSO Engineering Latin America.** L'article « *ABAP Code Reviews with abapGit* » explique une exportation et un commit volontairement manuels pour éviter de présenter les branches Git comme des branches de développement ABAP : cas de revue de code, **pas preuve d'un remplacement de TMS**. [S22]

### 8.2 gCTS : produit réel et scénarios opérationnels, preuves publiques d'adoption plus limitées

- SAP documente et maintient gCTS comme composant officiel ; la documentation et la SAP Community expliquent ses évolutions et son usage sur S/4HANA et ABAP cloud. Le billet « *gCTS is still here* » répond à des questions d'exploitation réelles, dont GitHub Actions, la synchronisation des cibles et le Customizing. **Il atteste d'essais et de problèmes rencontrés**, mais ne fournit pas à lui seul une étude chiffrée de généralisation en entreprise. [S17]
- Un tutoriel de membre de la communauté montre une chaîne **gCTS + GitHub + Jenkins**. C'est une preuve de **faisabilité technique**, pas nécessairement d'usage industriel sur des centaines de transports SAP. [S23]
- SAP publie aussi un scénario **ChaRM + gCTS** où une demande de changement est liée à Git, puis promue vers PROD. Il confirme qu'une **gouvernance complète est concevable**, sous réserve de sa configuration, des versions et du périmètre effectivement déployés. [S24]
- Dans les réponses d'une session SAP Community, SAP indique que **gCTS ne rend pas le développement ABAP parallèle**, que CTS et gCTS peuvent coexister, et qu'il n'y a **pas de projet de remplacement de CTS/STMS**. Ce sont des limites produit plus décisives que les promesses générales de DevOps. [S6]

**Appréciation de la preuve publique :**

| Sujet | Niveau de preuve trouvé | Ce qu'il est raisonnable d'en déduire |
|---|---|---|
| abapGit réellement utilisé dans des entreprises | **Oui**, liste de clients et récits techniques | Outil crédible pour développement, partage et revue |
| abapGit utilisé pour **remplacer CTS de bout en bout** dans un ERP complet | **Non établi par les sources consultées** | Ne pas l'affirmer |
| gCTS utilisable techniquement en DEV/QUAL/PROD | **Oui**, documentation officielle et scénarios | Faisabilité de principe établie |
| gCTS massivement utilisé en production par des entreprises nommées, avec chiffres publics de ROI | **Non démontré par l'échantillon consulté** | Adoption réelle possible, mais non quantifiée ici |
| Git supprime les verrous / automatise tout rollback | **Non**, contredit par SAP | Bénéfices à ne pas inclure dans le business case |

> **Limite méthodologique :** cette revue s'appuie sur des ressources SAP Help, SAP Community et abapGit publiques consultées au 29/10/2026. Elle n'est pas une enquête terrain exhaustive ; l'absence de cas client détaillé publié n'est pas une preuve d'absence d'utilisation. Les publications datées et les discussions peuvent se rapporter à différentes releases SAP.

## 9. Adéquation à notre chaîne DEV → QUAL → PROD

### Option A — CTS/TMS seul

**Pertinent si :** développement majoritairement SAP classique, processus stable, nombreux objets de configuration, obligation de couvrir les cas spéciaux et peu de bénéfices attendus d'une revue Git.

**Risque :** qualité et audit du code à renforcer par d'autres moyens (ATC, processus de revue, contrôles de release).

### Option B — CTS/TMS + abapGit (recommandée pour commencer)

```text
                  ┌─ export abapGit → Git → PR / diff / ATC / revue
DEV + CTS ────────┤
                  └─ transport CTS approuvé → QUAL → PROD
```

**Pertinent si :** le besoin prioritaire est de **voir et valider les différences**, sans changer les mécanismes de transport déjà maîtrisés.

**Conditions de réussite :**

- un export cohérent des objets **avant approbation**, et non une archive Git aléatoire ;
- identification systématique du transport, de la révision Git et du ticket ;
- signalement explicite des objets absents ou non sérialisables dans la revue ;
- règle claire en cas de divergence entre **révision Git examinée** et **objet effectivement importé** ;
- interdiction d'import direct depuis Git en QUAL/PROD sans procédure validée, si CTS reste le canal officiel.

### Option C — gCTS sur un périmètre autonome

**Pertinent si :** application ou ensemble de packages à forte proportion de code compatible, équipe ayant besoin de commits Git et de CI/CD, capacité d'exploiter une chaîne Git complète, peu de dépendances aux modifications SAP et au Customizing spécial.

**Conditions de réussite :** piloter la stratégie de dépôt, les couches de transport, les accès Git, les approbations, l'initialisation des packages, la promotion de commits identiques, les échecs d'import, les correctifs urgents, les versions SAP et les objets exclus. Ne pas mélanger les canaux pour un même objet sans gouvernance.

### Option D — Remplacement total de CTS par Git

**Non recommandé dans l'hypothèse actuelle**, car le périmètre métier peut excéder la couverture de la sérialisation, certains transports restent classiques, les verrous ABAP persistent, le rollback PROD n'est pas garanti et l'exploitation des chaînes devient plus complexe.

## 10. POC : comment décider sur des faits plutôt que sur une démonstration Git ?

**Périmètre conseillé :** une application Z représentative, un package avec code moderne et ancien, des objets DDIC et du Customizing, un véritable paysage DEV/QUAL (ou des systèmes techniques de test), une équipe ABAP et Basis. **Ne pas utiliser PROD pour les tests destructifs.**

| Test | Démonstration attendue | Critère d'échec / point de vigilance |
|---|---|---|
| **T1. Inventaire** | Extraire des transports réels les types d'objets (`E070/E071`, analyse des objets et dépendances) | Types essentiels non couverts ou incertains |
| **T2. Couverture** | Export/import d'un échantillon `CLAS`, `PROG`, `FUGR`, `TABL`, `DDLS`, enhancements et écrans/formulaires présents | Objets manquants, génération ou activation incomplète |
| **T3. Customizing** | Valeurs `TABU`/`VDAT`/`TDAT`, suppressions, mandants et éventuels `CDAT` selon release | Résultat partiel, illisible ou incohérent entre mandants |
| **T4. Dépendances** | Déployer en QUAL un ensemble cohérent avec séquence et imports contrôlés | Objet activé incorrectement, dépendance absente, résultat non reproductible |
| **T5. Revue avant livraison** | Un pair identifie le diff, les objets exclus et approuve le **contenu exact** livré | Git contient une autre version que le transport / commit déployé |
| **T6. Incident et rollback QUAL** | Revenir à un commit antérieur sur QUAL, examiner logs, activations, données inchangées | Rollback non reproductible ou effet résiduel masqué |
| **T7. Correctif urgent** | Simuler A en cours et correctif B sur **le même objet** avec un seul DEV | Nécessité d'arrêter l'équipe ou risque d'embarquer A non validé |
| **T8. Livraison mixte** | Intégrer objet gCTS + objet CTS avec approbation et ordre vérifiables | Deux sources de vérité sans ordre garanti |
| **T9. Résilience** | Panne Git, authentification expirée, commit non poussé, échec de pull/import | Incapacité à livrer ou à diagnostiquer l'état cible |
| **T10. Effort** | Mesurer temps de préparation, revue, correction, import, support et maintenance | Gains de revue inférieurs à l'effort ajouté |

**Livrables obligatoires du POC :**

1. matrice des **types réellement présents** dans les transports de l'entreprise, avec statut de couverture par release ;
2. procédures de **release**, **hotfix**, **échec d'import**, **restauration** et **convergence du code** ;
3. chaîne d'audit **ticket ↔ transport ↔ commit Git ↔ approbation ↔ version importée en QUAL/PROD** ;
4. mesures avant/après : temps de revue, défauts détectés, temps de préparation, incidents de livraison, temps de diagnostic ;
5. décision **GO / NO GO**, validée conjointement par développement, architecture, sécurité et Basis.

### Critères de décision proposés

**GO pour abapGit + CTS** si la revue Git fonctionne, si les exclusions sont signalées et si la chaîne CTS n'est pas fragilisée.

**GO pour gCTS sur périmètre délimité** si 100 % des objets **critiques du périmètre ciblé** sont transportés et réactivés correctement, si un hotfix a été simulé, si la production conserve un scénario de reprise maîtrisé et si les coûts additionnels sont acceptables.

**NO GO pour le remplacement généralisé** si le système ne peut pas livrer un changement métier complet dans une seule gouvernance, ou si la procédure de retour à l'état cohérent dépend d'un hypothétique « git revert automatique ».

## 11. Recommandation finale

Pour un **DEV unique partagé**, avec **verrouillage des objets ABAP**, des **transports mixtes code/Customizing** et une attente de **rollback fiable** :

> **Ne pas remplacer CTS/TMS uniquement pour « mettre les transports sur Git ».**
>
> **Conserver la promotion SAP existante** et déployer Git d'abord là où sa valeur est démontrable : **revue de code, diff, audit, partage et contrôles de qualité**.
>
> **Évaluer gCTS séparément**, sur une application autonome, en jugeant les capacités d'import, les restrictions de version, les contraintes de branches et la reprise sur incident — pas sur la seule création d'un commit.

La réponse à la question de départ est donc **nuancée** : **oui**, exporter le code sur Git peut améliorer la qualité et la traçabilité ; **non**, cela n'améliore pas automatiquement le transport SAP, ne rend pas tous les objets versionnables, ne crée pas de branches de travail indépendantes dans un seul DEV et ne rend pas le rollback de production automatique ou sûr.

---

## Sources et liens consultés

**Priorité de lecture : documentation SAP officielle, puis publications d'experts SAP, puis retours des membres de SAP Community.** Les restrictions techniques s'apprécient selon la version exacte d'ABAP Platform, S/4HANA ou BTP ABAP et les SAP Notes à jour.

- **[S1] SAP Help — Git-Based Software Distribution**, processus et avertissement explicite sur le rollback uniquement en test/qualité : <https://help.sap.com/docs/LATEST/4a368c163b08418890a406d413933ba7/85ca2b82afe942efb2fa244940f0453c.html>
- **[S2] SAP Help — CTS Entities Used in gCTS**, demandes, commits, verrouillage, import des différences : <https://help.sap.com/docs/latest/4a368c163b08418890a406d413933ba7/ffd38f5ef3a74d93ba6123a5899e5658.html>
- **[S3] SAP Help — Features of gCTS**, couverture « nearly all », restriction Note 2888887 : <https://help.sap.com/docs/ABAP_PLATFORM_NEW/4a368c163b08418890a406d413933ba7/d22a2e9e099b4f1e8d4c6f4721d4e782.html>
- **[S4] SAP — Change and Transport System**, présentation du standard : <https://supporta.sap.com/en/tools/software-logistics-tools/change-and-transport-system.html>
- **[S5] SAP Community — SAP BTP ABAP environment FAQ**, positionnement abapGit et gCTS : <https://pages.community.sap.com/topics/business-ai-platform-abap-environment/faq>
- **[S6] SAP Community — Some answers for the gCTS Community Call**, coexistence, verrouillage, limites, granularité des commits : <https://community.sap.com/t5/technology-blog-posts-by-sap/some-answers-for-the-gcts-community-call/ba-p/13472982>
- **[S7] abapGit Documentation — Supported Object Types** : <https://docs.abapgit.org/user-guide/reference/supported.html>
- **[S8] abapGit Documentation — Troubleshooting**, couverture selon versions : <https://docs.abapgit.org/user-guide/getting-started/troubleshooting.html>
- **[S9] SAP Help — gCTS Troubleshooting**, incompatibilité des formats abapGit/gCTS dans un même dépôt : <https://help.sap.com/docs/ABAP_PLATFORM_NEW/0fadfa00c56c4936a383af16982af221/852c9602da9d4215ab85a27ca26e1567.html>
- **[S10] SAP Help — SAP Cloud Transport Management, Supported Content Types**, référence au type **BTP ABAP** : <https://help.sap.com/docs/cloud-transport-management/sap-cloud-transport-management/supported-content-types>
- **[S11] SAP Community — Setting up SAP Cloud Transport Management for SAP BTP ABAP Environment**, paysage trois systèmes : <https://community.sap.com/t5/technology-blog-posts-by-sap/setting-up-sap-cloud-transport-management-for-sap-btp-abap-environment/ba-p/13558213>
- **[S12] SAP Help — gCTS (BC-CTS-GIT)**, contraintes, notes SAP et ajustements de modifications : <https://help.sap.com/docs/ABAP_PLATFORM_NEW/9d6aa238582042678952ab3b4aa5cc71/f319b168e87e42149e25e13c08d002b9.html>
- **[S13] SAP Help — Git-Enabled Change and Transport System (ABAP Platform 2020)**, évolution du Customizing et exemple de `CDAT` : <https://help.sap.com/docs/PRODUCTS/66906ae3920c4fc684cf588290fb9267/a0148cb81b1346109437b66b66980def.html>
- **[S14] SAP Help — Supported ABAP Object Types (BTP ABAP environment)**, illustration des différences de périmètre : <https://help.sap.com/docs/sap-btp-abap-environment/abap-environment/supported-abap-object-types>
- **[S15] SAP Help — Manually Deploy Objects**, Customizing mandant-dépendant : <https://help.sap.com/docs/ABAP_PLATFORM_NEW/4a368c163b08418890a406d413933ba7/36e4d0c2d0484c08923b77b01522c0d8.html>
- **[S16] SAP Help — gCTS Troubleshooting**, chargement initial d'objets complets / objets partiels : <https://help.sap.com/docs/ABAP_PLATFORM_NEW/0fadfa00c56c4936a383af16982af221/852c9602da9d4215ab85a27ca26e1567.html>
- **[S17] SAP Community — gCTS is still here**, questions et limites remontées par les lecteurs : <https://community.sap.com/t5/technology-blog-posts-by-sap/gcts-is-still-here/bc-p/13864142>
- **[S18] SAP Help — Use Case 1: One Codeline in a 3-System Landscape**, interruptions nécessaires pour corrections urgentes : <https://help.sap.com/docs/btp/sap-business-technology-platform/use-case-1-one-codeline-in-3-system-landscape>
- **[S19] SAP Community — Co-ordinate maintenance and new development with gCTS**, une seule version active par objet, DEV/COR distincts souvent nécessaires : <https://community.sap.com/t5/technology-blog-posts-by-sap/co-ordinate-maintenance-and-new-development-with-gcts/ba-p/13901231>
- **[S20] abapGit Documentation — Who Uses abapGit?**, liste déclarative (publication historique) : <https://docs.abapgit.org/user-guide/other/where-used.html>
- **[S21] SAP Community — How we do ABAP code review using abapGit**, exemple de revue avec transport CTS : <https://community.sap.com/t5/application-development-and-automation-blog-posts/how-we-do-abap-code-review-using-abapgit/ba-p/13448724>
- **[S22] SAP Community — ABAP Code Reviews with abapGit**, SAP IBSO Engineering Latin America : <https://community.sap.com/t5/application-development-and-automation-blog-posts/abap-code-reviews-with-abapgit/bc-p/13354406>
- **[S23] SAP Community — SAP DevOps Using gCTS, GitHub and Jenkins**, démonstration d'architecture : <https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-members/sap-devops-using-gcts-github-and-jenkins/ba-p/13500525>
- **[S24] SAP Community — How Change Request Management (ChaRM) Leverages gCTS**, workflow de promotion et intégration de changements : <https://community.sap.com/t5/technology-blog-posts-by-sap/how-change-request-management-charm-leverages-git-enabled-cts-gcts/ba-p/13508779>

### Points à vérifier avant toute décision d'architecture

- Releases exactes **SAP_BASIS / ABAP Platform / S/4HANA**, notes **2821718**, **2888887** et corrections gCTS pertinentes.
- Existence d'un **DEV unique ou de plusieurs environnements isolés**, et processus de gestion des hotfixes.
- Inventaire réel des transports Workbench/Customizing, **mandants**, modifications SAP, notes SAP et objets générés.
- Besoin d'une simple **revue Git**, d'une **traçabilité** renforcée, ou d'une véritable **nouvelle chaîne de déploiement** : ce sont trois projets de complexité différente.
