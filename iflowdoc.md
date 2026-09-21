# Mode opératoire — Démo Integration Suite : IDoc ↔ Kafka

**Deux iFlows Cloud Integration testables avec Postman et un broker Kafka de test**

| Version | Date | Statut |
|---|---|---|
| 1.0 | 21/09/2026 | BROUILLON — questions ouvertes au §9 |

> **Conventions**
> - Suite du mode opératoire `MO_BTP-trial_IS-S4-CAP-WorkZone.md` : les paramètres `P9` (mandant), `P10` (user technique), `P11` (hôte/port virtuels), `P12` (Location ID), `P15` (service key) et l'identifiant `S4H_PCE_BASIC` en viennent.
> - Les valeurs `<…>` sont à renseigner depuis la fiche de paramètres (§0.6). Les libellés ⚠ peuvent varier selon la version du tenant.
> - **Exemple métier** : IDoc `MATMAS05` (article) et message JSON `MaterialEvent`, en attendant la réponse à Q2.

---

## 0. Cadrage

### 0.1 Les deux flux

| iFlow | Déclencheur | Traitement | Cible |
|---|---|---|---|
| `IF_IDOC_MATMAS_TO_KAFKA` (flux 1) | Postman envoie un IDoc MATMAS05 (simule le S/4) | XSLT IDoc → `MaterialEvent`, conversion JSON, clé Kafka = n° d'article | Topic `sap.material.out` |
| `IF_KAFKA_TO_IDOC_MATMAS` (flux 2) | Record JSON sur le topic `sap.material.in` | Conversion XML, XSLT → IDoc MATMAS05 avec record de contrôle paramétré | S/4 via Cloud Connector (`/sap/bc/srt/idoc`) |

### 0.2 Architecture

```
Flux 1
[Postman] --SOAP + OAuth2--> [IDoc sender /cxf/idoc/matmas] -> XSLT -> XML→JSON -> [Kafka receiver] --> topic sap.material.out

Flux 2
topic sap.material.in --> [Kafka sender] -> JSON→XML -> XSLT -> [IDoc receiver, On-Premise] --> Cloud Connector --> S/4 /sap/bc/srt/idoc

Chaînage optionnel (§4.6 C) : flux 2 lit sap.material.out → un seul appel Postman déclenche les deux flux.
```

### 0.3 Contenu du package

| Fichier | Rôle |
|---|---|
| `iflow-resources/MATMAS05_to_MaterialEvent.xsl` | Mapping du flux 1 |
| `iflow-resources/MaterialEvent_to_MATMAS05.xsl` | Mapping du flux 2 (paramètres `SNDPOR`, `SNDPRN`, `RCVPOR`, `RCVPRN`) |
| `iflow-resources/LogCustomHeaders.groovy` | Identifiants métier dans le monitoring (article, n° IDoc, topic/offset) |
| `samples/MATMAS05_soap.xml` | IDoc de test (enveloppe SOAP) pour le flux 1 |
| `samples/MaterialEvent.json` | Record de test pour le flux 2 |
| `postman/IS_IDoc_Kafka_Demo.postman_collection.json` | Collection Postman (flux 1 + production Kafka optionnelle via Confluent Cloud) |

Les deux XSLT ont été testés avec Saxon-HE : flux 1, flux 2 et chaînage flux 1 → flux 2. Les iFlows sont à modéliser dans l'éditeur (§3 et §4) : un export `.zip` construit hors tenant n'offrirait aucune garantie d'import.

### 0.4 Contraintes Kafka (vérifiées dans la documentation SAP)

- **Pas de Cloud Connector** : l'adaptateur Kafka standard ne sait pas passer par le Cloud Connector. **Le broker de test doit être joignable depuis Internet.**
- **Authentification obligatoire** :
  - SASL (`PLAIN`, `SCRAM-SHA-256` ou `SCRAM-SHA-512`), avec ou sans TLS ;
  - ou certificat client (protocole `SSL`) ;
  - aucune connexion anonyme possible.
- **TLS** : le certificat racine du broker doit être présent dans le keystore du tenant.
- **Consumer group** : c'est l'**ID de l'iFlow** qui sert de consumer group ; un nouveau groupe démarre à l'offset **LATEST**. → Déployer le flux 2 **avant** de produire des messages, et ne pas changer son ID ensuite.
- **Clé du record** : le header `kafka.KEY` définit la clé du record Kafka.
- **Noms de topics** : caractères `[a-zA-Z0-9._-]`, avec des points **ou** des underscores, pas les deux.

### 0.5 Hypothèses de démo (à valider, §9)

- IDoc `MATMAS05`, segments `E1MARAM` et `E1MAKTM` uniquement.
- Deux topics distincts : `sap.material.out` (flux 1) et `sap.material.in` (flux 2).
- Authentification S/4 : basic auth avec le user technique `P10`.
- Flux 1 alimenté par Postman ; le branchement d'un vrai S/4 émetteur est décrit en option au §6.

### 0.6 Fiche de paramètres

| # | Paramètre | Où le trouver | Valeur |
|---|---|---|---|
| K1 | Bootstrap server `host:port` (FQDN public) | Console du broker | |
| K2 | Mécanisme SASL | Console du broker | `PLAIN` / `SCRAM-SHA-256` / `SCRAM-SHA-512` |
| K3 | Utilisateur SASL / API key | Console du broker | |
| K4 | Mot de passe SASL / API secret | Console du broker | **secret** |
| K5 | TLS (`SASL_SSL`) ou non (`SASL_PLAINTEXT`) | Console du broker | |
| K6 | Certificat racine du broker (si TLS) | Test TLS CPI (§2.2) ou fournisseur | |
| I1 | Système logique de CPI (`SNDPRN` flux 2) | Créé en `BD54` (§5.3) | ex. `CPIDEMO` |
| I2 | Port émetteur (`SNDPOR` flux 2) | Convention à fixer (§5.4) | ex. `CPIDEMO` |
| I3 | Système logique du S/4 (`RCVPRN` flux 2) | `SCC4` → mandant P9 → *Logical System* | |
| I4 | Port destinataire (`RCVPOR` flux 2) | Record de contrôle d'un IDoc entrant existant (`WE02`) ou Basis | |

---

## 1. Préparer le broker Kafka de test

1. Vérifier les prérequis du §0.4 : FQDN public, SASL ou certificat client, TLS recommandé.
2. Vérifier que les **advertised listeners** du broker annoncent le FQDN public (et non un nom interne), sinon le client se connecte au bootstrap puis échoue sur les brokers.
3. Créer deux topics : `sap.material.out` et `sap.material.in` (1 partition suffit pour la démo).
4. Créer un identifiant SASL (K3/K4) avec, si des ACL sont actives :
   - écriture sur `sap.material.out` ;
   - lecture sur `sap.material.in` ;
   - lecture sur le **consumer group** `IF_KAFKA_TO_IDOC_MATMAS` (= ID du flux 2).
5. Installer un client de test : **kcat** (ex-kafkacat), en natif ou via Docker.

Variables kcat réutilisées plus bas (adapter `security.protocol` : `SASL_SSL` si TLS, `SASL_PLAINTEXT` sinon ; ajouter `-X ssl.ca.location=<ca.pem>` si l'autorité du broker n'est pas publique) :

```sh
KCAT_OPTS="-b <K1> -X security.protocol=SASL_SSL -X sasl.mechanisms=<K2> -X sasl.username=<K3> -X sasl.password=<K4>"
kcat $KCAT_OPTS -L          # liste brokers et topics : valide connexion + authentification
```

---

## 2. Sécurité dans le tenant Cloud Integration

### 2.1 Identifiants Kafka

**Monitor** → **Integrations and APIs** → **Security Material** → **Create** → **User Credentials** :

| Champ | Valeur |
|---|---|
| Name | `KAFKA_SASL` |
| Type | User Credentials |
| User | K3 |
| Password | K4 |

→ **Deploy**.

### 2.2 Certificat du broker (si TLS)

1. **Monitor** → **Integrations and APIs** → **Connectivity Tests** → onglet **TLS** → *Host* = hôte de K1, *Port* = port de K1 → **Send**.
2. Si la chaîne n'est pas reconnue : récupérer le certificat racine (depuis le résultat du test ou auprès du fournisseur) → **Monitor** → **Keystore** → **Add** → **Certificate** → *Alias* `kafka-root-ca` → charger le fichier → **Add**.
3. Toute modification du keystore impose de **redéployer** (ou redémarrer) les iFlows Kafka.

### 2.3 Identifiants S/4

`S4H_PCE_BASIC` (user technique P10) existe déjà (mode opératoire précédent, §9.1). Les autorisations IDoc entrantes sont à ajouter au §5.6.

---

## 3. Flux 1 — `IF_IDOC_MATMAS_TO_KAFKA` (IDoc → Kafka)

### 3.1 Création

1. **Design** → **Integrations and APIs** → package `Demo S4 PCE` (ou un nouveau package `Demo IDoc Kafka`).
2. **Artifacts** → **Add** → **Integration Flow** → *Name* `IF_IDOC_MATMAS_TO_KAFKA` → **OK** → ouvrir → **Edit**.

### 3.2 Modélisation (dans l'ordre)

**a. Sender → Start : adaptateur IDoc**

| Onglet / champ | Valeur |
|---|---|
| Connection > Address | `/idoc/matmas` |
| Connection > Authorization | User Role |
| Connection > User Role | `ESBMessaging.send` |

**b. Content Modifier `Extract IDoc keys`**

Onglet **Exchange Property** :

| Action | Name | Source Type | Source Value | Data Type |
|---|---|---|---|---|
| Create | `IDocNumber` | XPath | `/MATMAS05/IDOC/EDI_DC40/DOCNUM` | `java.lang.String` |
| Create | `Material` | XPath | `/MATMAS05/IDOC/E1MARAM/MATNR` | `java.lang.String` |

Onglet **Message Header** :

| Action | Name | Source Type | Source Value |
|---|---|---|---|
| Create | `SAP_ApplicationID` | Expression | `${property.Material}` |
| Create | `kafka.KEY` | Expression | `${property.Material}` |

`SAP_ApplicationID` rend l'article cherchable dans le monitoring ; `kafka.KEY` devient la clé du record (même article → même partition).

**c. XSLT Mapping `Map to MaterialEvent`**

*Processing* → *Resource* : **Select** → **Upload from File System** → `MATMAS05_to_MaterialEvent.xsl`.

**d. XML to JSON Converter**

| Champ | Valeur |
|---|---|
| Use Namespace Mapping | Décoché |
| JSON Output Encoding | UTF-8 |
| Suppress JSON Root Element | **Décoché** (le JSON garde la racine `MaterialEvent`, attendue par le flux 2) |
| Streaming | Décoché |

**e. Groovy Script `Log custom headers`**

Assigner `LogCustomHeaders.groovy` (upload depuis le poste).

**f. End → Receiver `KAFKA` : adaptateur Kafka**

| Onglet / champ | Valeur |
|---|---|
| Connection > Host | K1 |
| Connection > Authentication | SASL |
| Connection > Connect with TLS | Selon K5 |
| Connection > SASL Mechanism | K2 |
| Connection > Credential Name | `KAFKA_SASL` |
| Processing > Topic | `{{KAFKA_TOPIC_OUT}}` (paramètre externalisé) |
| Autres paramètres | Valeurs par défaut |

### 3.3 Configuration et déploiement

1. **Save** → **Configure** → onglet **Receiver** → `KAFKA_TOPIC_OUT` = `sap.material.out` → **Save**.
2. **Deploy** → **Monitor** → **Manage Integration Content** → statut **Started** → onglet **Endpoints** : URL `<oauth.url>/cxf/idoc/matmas`.

### 3.4 Test

1. Postman → **Import** → `IS_IDoc_Kafka_Demo.postman_collection.json`.
2. Collection → onglet **Variables** : renseigner `cpi_runtime_url`, `token_url`, `client_id`, `client_secret` (service key P15). Pour les secrets, préférer un environnement Postman avec des variables de type *secret*.
3. Dossier **CPI** → onglet **Authorization** → **Get New Access Token** → **Use Token**.
4. Requête **Flux 1 - Envoyer IDoc MATMAS05** → adapter au besoin `MATNR`, `MTART`, `MBRSH`, `MEINS` → **Send** → attendu : `200` et une réponse SOAP.
5. Contrôler Kafka :

```sh
kcat $KCAT_OPTS -C -t sap.material.out -o beginning -e -f 'key=%k partition=%p offset=%o\n%s\n\n'
```

Attendu : `key=DEMO-MAT-001` et le JSON `{"MaterialEvent": {...}}`.

6. Contrôler CPI : **Monitor Message Processing** → message *Completed*, Custom Headers `Material` et `IDocNumber`.

---

## 4. Flux 2 — `IF_KAFKA_TO_IDOC_MATMAS` (Kafka → IDoc)

Prérequis : configuration S/4 du §5 réalisée.

### 4.1 Création

**Artifacts** → **Add** → **Integration Flow** → *Name* `IF_KAFKA_TO_IDOC_MATMAS` → **OK** → **Edit**.

⚠ L'ID de l'iFlow sert de consumer group : le changer revient à créer un nouveau groupe, qui repart de l'offset LATEST.

### 4.2 Runtime Configuration

Cliquer sur le fond du canevas → onglet **Runtime Configuration** → *Allowed Header(s)* :

```
kafka.TOPIC|kafka.PARTITION|kafka.OFFSET|kafka.KEY
```

Ces headers, posés par l'adaptateur Kafka sender, sont ensuite journalisés par le script.

### 4.3 Modélisation (dans l'ordre)

**a. Sender `KAFKA` → Start : adaptateur Kafka**

| Onglet / champ | Valeur |
|---|---|
| Connection > Host / Authentication / Connect with TLS / SASL Mechanism / Credential Name | Identiques au flux 1 |
| Processing > Topic | `{{KAFKA_TOPIC_IN}}` |
| Processing > Parallel Consumers | `1` |
| Processing > Error Handling | **Skip Failed Message** pour la démo (sinon un message en erreur est rejoué en boucle et bloque la partition) ; **Retry Failed Message** pour un comportement « au moins une fois » |
| Advanced | Valeurs par défaut |

**b. JSON to XML Converter**

*Use Namespace Mapping* décoché. Si l'option d'ajout d'un élément racine existe dans ta version, la laisser décochée : le JSON a déjà une racine unique `MaterialEvent`.

**c. Content Modifier `Set IDoc control data`**

Onglet **Exchange Property** :

| Action | Name | Source Type | Source Value | Data Type |
|---|---|---|---|---|
| Create | `SNDPOR` | Constant | `{{IDOC_SNDPOR}}` | |
| Create | `SNDPRN` | Constant | `{{IDOC_SNDPRN}}` | |
| Create | `RCVPOR` | Constant | `{{IDOC_RCVPOR}}` | |
| Create | `RCVPRN` | Constant | `{{IDOC_RCVPRN}}` | |
| Create | `Material` | XPath | `/MaterialEvent/material` | `java.lang.String` |

Onglet **Message Header** : `SAP_ApplicationID` = Expression `${property.Material}`.

Le XSLT déclare `<xsl:param>` avec les mêmes noms : Cloud Integration lui transmet ces exchange properties. Si ton éditeur n'accepte pas `{{…}}` dans un Content Modifier, saisir les valeurs en dur.

**d. XSLT Mapping `Map to MATMAS05`**

Resource : `MaterialEvent_to_MATMAS05.xsl`.

**e. Request Reply → Receiver `S4HANA` : adaptateur IDoc**

| Onglet / champ | Valeur |
|---|---|
| Connection > Address | `http://s4h-virtual:44300/sap/bc/srt/idoc?sap-client=<P9>` (hôte/port virtuels P11) |
| Connection > Proxy Type | On-Premise |
| Connection > Location ID | P12 |
| Connection > IDoc Content Type | **Text/XML** |
| Connection > Authentication | Basic |
| Connection > Credential Name | `S4H_PCE_BASIC` |
| Connection > Timeout | `60000` (défaut) |
| Processing > SAP Message ID Determination | Reuse (défaut) |

Avec *On-Premise*, l'adresse doit être en `http://` et pointer vers l'hôte virtuel du Cloud Connector. Le contenu *Application/x-sap.idoc* n'accepte qu'un IDoc par requête mais apporte séquencement et Exactly-Once : il n'est pas nécessaire pour la démo.

**f. Groovy Script `Log custom headers`** → `LogCustomHeaders.groovy`. L'adaptateur IDoc renvoie notamment `SapIDocDbId` (numéro d'IDoc dans le S/4), journalisé en `S4_IDocNumber`.

**g. End.**

### 4.4 Configuration et déploiement

1. **Configure** → renseigner les paramètres externalisés :

| Paramètre | Valeur |
|---|---|
| `KAFKA_TOPIC_IN` | `sap.material.in` |
| `IDOC_SNDPOR` | I2 |
| `IDOC_SNDPRN` | I1 |
| `IDOC_RCVPOR` | I4 |
| `IDOC_RCVPRN` | I3 |

2. **Deploy**, **avant** toute production de message (offset LATEST).
3. **Manage Integration Content** : statut **Started**. En cas d'erreur de polling (connexion, authentification), le statut passe à *Error* et le détail apparaît dans le panneau *Error Details*.

### 4.5 Test

Choisir une des trois options.

**A. Postman (uniquement si le broker est un cluster Confluent Cloud)** : renseigner les variables `confluent_*` de la collection → requête **Flux 2 - Produire MaterialEvent** → **Send** (API REST v3 de Confluent).

**B. kcat (tout broker)** :

```sh
kcat $KCAT_OPTS -P -t sap.material.in -k DEMO-MAT-002 samples/MaterialEvent.json
```

**C. Chaînage des deux démos** : paramètre `KAFKA_TOPIC_IN` = `sap.material.out` → redéployer → la requête Postman du flux 1 déclenche aussi le flux 2.

### 4.6 Contrôles

1. **Monitor Message Processing** : message *Completed*, Custom Headers `Material`, `S4_IDocNumber`, `KafkaTopic`, `KafkaOffset`.
2. S/4 : `WE02` → IDoc n° `S4_IDocNumber` :
   - statut **53** : article créé ;
   - statut **51** : erreur applicative, lire le message et ajuster les données (type d'article, secteur, unité, autorisations).
3. `MM03` → article `DEMO-MAT-002`.

---

## 5. S/4 — Configuration pour recevoir les IDocs (flux 2)

| # | Transaction | Action |
|---|---|---|
| 5.1 | `SICF` | Activer `/default_host/sap/bc/srt/idoc` (clic droit → *Activate Service*) |
| 5.2 | `SRTIDOC` | *Register Service* → valeurs par défaut → **Execute** |
| 5.3 | `BD54` | *New Entries* → *Log. System* `CPIDEMO` (I1), *Name* `SAP Cloud Integration trial` → sauvegarder (ordre cross-client) |
| 5.4 | `SCC4` / `WE02` | Relever le système logique du mandant P9 (I3). Relever le port destinataire (I4) dans le record de contrôle d'un IDoc entrant existant, ou auprès du Basis. Fixer une convention pour I2 (ex. nom du port XML-HTTP vers CPI s'il existe, §6) |
| 5.5 | `WE20` | *Partner Type* **LS** → **Create** → *Partner No.* `CPIDEMO` ; onglet *Post-processing: permitted agent* : *Ty.* `US`, *Agent* ton user → sauvegarder. *Inbound parameters* **+** : *Message Type* `MATMAS`, *Process Code* `MATM`, *Trigger Immediately* → sauvegarder |
| 5.6 | `PFCG` | Rôle du user P10 : ajouter `B_ALE_RECV` (type de message `MATMAS`) et les autorisations article nécessaires à la création ; compléter avec `SU53` après le premier essai |
| 5.7 | Cloud Connector | Mapping S/4 → **Resources** → **+** : *URL Path* `/sap/bc/srt/idoc`, *Access Policy* **Path Only (Sub-Paths Are Excluded)** → **Save** |
| 5.8 | `WE60` | Documentation / schéma XML de `MATMAS05` : vérifier l'ordre et la longueur des champs utilisés par le XSLT |

Outils de suivi : `WE02` / `WE05` (liste des IDocs), `BD87` (retraitement), `WE19` (outil de test), `SRT_UTIL` (journal d'erreurs du runtime SOAP), `SU53`, `SM21`.

---

## 6. Option — Remplacer Postman par un vrai S/4 émetteur (flux 1)

| # | Transaction | Action |
|---|---|---|
| 6.1 | `STRUST` | Importer le certificat racine du runtime CPI dans le PSE *SSL client Standard* |
| 6.2 | `SM59` | Destination **type G** : hôte = hôte de `oauth.url`, port `443`, *Path Prefix* `/cxf/idoc/matmas`, SSL actif, logon basic `clientid` / `clientsecret` (P15) → *Connection Test* (un HTTP 500 sans payload est normal) |
| 6.3 | `WE21` | Port **XML HTTP** (ex. `CPIDEMO`) → destination 6.2, *Content Type* Text/XML, **SOAP Protocol** coché |
| 6.4 | `WE20` | Partenaire LS `CPIDEMO` → *Outbound parameters* : `MATMAS`, port 6.3, type de base `MATMAS05`, *Transfer IDoc Immediately* |
| 6.5 | `BD10` ou `WE19` | Envoyer un article → statut **03** dans `WE02`, message *Completed* dans CPI |

---

## 7. Recette

| # | Contrôle | Attendu |
|---|---|---|
| R1 | `kcat -L` | Brokers et topics listés |
| R2 | Test TLS CPI vers K1 | Chaîne reconnue |
| R3 | Flux 1 et flux 2 dans *Manage Integration Content* | Started |
| R4 | Postman → flux 1 | `200` |
| R5 | Topic `sap.material.out` | JSON `MaterialEvent`, clé = article |
| R6 | Record sur `sap.material.in` | Message *Completed* dans CPI |
| R7 | `WE02` | IDoc MATMAS05 statut 53 (ou 51 expliqué) |

---

## 8. Dépannage

| Symptôme | Cause probable | Action |
|---|---|---|
| Postman `401` | Token, `clientsecret`, URL de token | Service key P15 |
| Postman `403` | Rôle ≠ `ESBMessaging.send` | §3.2 a |
| Postman `404` | Address ou URL (oubli de `/cxf`) | Onglet Endpoints |
| Postman `500` ou fault SOAP | Corps envoyé sans enveloppe SOAP, ou XML invalide | Envoyer l'enveloppe complète (`samples/MATMAS05_soap.xml`), lire le MPL |
| Kafka : `PKIX` / `SSLHandshakeException` | Racine du broker absente du keystore | §2.2 puis redéploiement |
| Kafka : échec d'authentification SASL | Mauvais mécanisme ou identifiants | K2 à K4, redéployer après modification |
| Kafka : timeout | Broker non joignable depuis Internet, ou advertised listeners internes | §1.2 ; pas de Cloud Connector possible |
| Kafka : `TopicAuthorizationException` / `GroupAuthorizationException` | ACL manquantes | §1.4 (groupe = ID du flux 2) |
| Flux 2 en *Error* | Polling en échec | Panneau *Error Details* |
| Flux 2 ignore un message ancien | Nouveau groupe positionné en LATEST | Produire après déploiement |
| Flux 2 rejoue sans fin | *Retry Failed Message* avec erreur persistante | Corriger ou passer en *Skip* |
| JSON to XML en erreur | Record non JSON ou sans racine unique | Format de `MaterialEvent.json` |
| IDoc receiver : connexion refusée / tunnel | Cloud Connector, Location ID | Test *Cloud Connector* CPI |
| `403` renvoyé par le Cloud Connector | `/sap/bc/srt/idoc` non exposé | §5.7 |
| `401` renvoyé par le S/4 | User P10 verrouillé ou mauvais mot de passe | `SU01` |
| Fault SOAP / `500` renvoyé par le S/4 | Service SRT non enregistré, erreur SOAP | §5.1–5.2, `SRT_UTIL` |
| IDoc statut 56 | Profil partenaire introuvable (SNDPRN / SNDPRT / MESTYP) | `WE20`, valeurs I1 à I4 |
| IDoc statut 51 | Données ou autorisations article | Message du statut, `SU53` |

---

## 9. Questions ouvertes

- **Q1** — Quel broker Kafka de test (service managé ou auto-hébergé) ? Est-il joignable depuis Internet ? Quel mode : SASL (quel mécanisme), TLS, certificat client ?
- **Q2** — `MATMAS05` convient-il, ou faut-il un autre type (`DEBMAS`, `ORDERS05`, extension Z…) ?
- **Q3** — Flux 2 : cible S/4 réelle (qui réalise le §5 ?) ou simple démonstration de l'IDoc produit ?
- **Q4** — Flux 1 : Postman uniquement, ou aussi un vrai S/4 émetteur (§6) ?
- **Q5** — Deux démos séparées (deux topics) ou chaînées (option C) ?
- **Q6** — Conventions de nommage pour I1 à I4 (système logique CPI, ports).
