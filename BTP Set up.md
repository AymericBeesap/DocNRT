# Mode opératoire — Démo SAP BTP Trial

**Integration Suite + Postman ↔ S/4HANA 2021 FPS02 (via Cloud Connector) et application CAP publiée dans SAP Build Work Zone, standard edition**

| Version | Date | Statut |
|---|---|---|
| 1.0 | 18/09/2026 | BROUILLON — questions ouvertes au §16 |

> **Conventions**
> - Les libellés d'écran sont donnés en anglais (langue par défaut du cockpit BTP). SAP les fait évoluer régulièrement : les points marqués ⚠ sont connus pour changer.
> - Toute valeur entre chevrons `<…>` est à renseigner depuis la **fiche de paramètres (§0.3)**. Aucune valeur système n'est supposée.
> - Les noms proposés (`S4H_PCE`, `s4h-virtual`, `ZBTP_DEMO`…) sont des propositions de nommage, à aligner sur tes conventions.
> - L'API `API_BUSINESS_PARTNER` (lecture) est utilisée **à titre d'exemple** en attendant la réponse à Q3.

---

## 0. Cadrage

### 0.1 Objectifs de la démo

1. **Intégration** : Postman appelle un iFlow Cloud Integration (OAuth 2.0 client credentials). L'iFlow lit une API OData du S/4HANA 2021 FPS02 à travers le Cloud Connector et renvoie le résultat.
2. **Extension** : une application CAP (service + UI Fiori elements) est déployée sur Cloud Foundry et exposée sous forme de tuile dans un site SAP Build Work Zone, standard edition.

### 0.2 Architecture cible

```
[Postman] --(OAuth2 client credentials)--> [Cloud Integration : iFlow]
                                                 | HTTP receiver, Proxy Type = On-Premise, Location ID
                                                 v
                              [Connectivity service BTP] <== tunnel TLS ==> [Cloud Connector]
                                                                                 | HTTPS
                                                                                 v
                                                             [S/4HANA 2021 FPS02 : SAP Gateway / OData]

[Navigateur] --(login IAS / OIDC)--> [Site Work Zone] --(managed approuter)--> [HTML5 App Repository : app Fiori]
                                                                                     |
                                                                                     v
                                                                  [CAP srv (Cloud Foundry)] --> [HANA Cloud / HDI]
                                                                                     |  (variante §12.6)
                                                                                     +--> destination S4H_PCE --> Cloud Connector --> S/4
```

### 0.3 Fiche de paramètres (à remplir avant de commencer)

| # | Paramètre | Où le trouver | Valeur |
|---|---|---|---|
| P1 | Région du compte trial | Choisie au §1 | |
| P2 | Subaccount ID (GUID) | Cockpit > subaccount > **Overview** > *Subaccount ID* | |
| P3 | Subdomain | Cockpit > subaccount > **Overview** > *Subdomain* | |
| P4 | CF API Endpoint | **Overview** > section *Cloud Foundry Environment* > *API Endpoint* | |
| P5 | CF Org Name / Org ID | **Overview** > section *Cloud Foundry Environment* | |
| P6 | CF Space | **Cloud Foundry** > **Spaces** | `dev` |
| P7 | FQDN S/4 joignable **depuis la machine Cloud Connector** | `SMICM` / `RZ11` (`icm/host_name_full`) + confirmation Basis/ECS (Q1, Q2) | |
| P8 | Port HTTPS S/4 | `SMICM` > *Goto* > *Services* | |
| P9 | Mandant S/4 | `SCC4` | |
| P10 | Utilisateur technique S/4 | Créé au §6.4 | |
| P11 | Hôte / port virtuels | Définis au §7.5 | ex. `s4h-virtual` / `44300` |
| P12 | Location ID | Défini au §7.4 | ex. `S4PCE` |
| P13 | Nom de la destination | §8 | ex. `S4H_PCE` |
| P14 | Tenant SAP Cloud Identity Services | E-mail d'activation (§3.2) | |
| P15 | `url`, `tokenurl`, `clientid`, `clientsecret` du runtime CPI | Service key (§5.4) | **Ne jamais versionner dans Git** |

### 0.4 Prérequis, limites et points de vigilance

- **Durée** : le trial BTP est non commercial, limité à 90 jours ; à expiration, subaccounts, applications, services et données sont supprimés.
- **Integration Suite trial** : disponible uniquement en **US East (VA) – AWS** et **Singapore – Azure** ; **un seul tenant Integration Suite par compte trial**.
- **SAP Build Work Zone** : les nouvelles souscriptions exigent un trust **OpenID Connect** avec un tenant **SAP Cloud Identity Services** (KBA SAP 3600432). Le trust SAML / Default Identity Provider ne suffit plus. → §3 obligatoire **avant** §4.2.
- **HANA Cloud trial** : l'instance est arrêtée chaque nuit, à redémarrer chaque jour d'utilisation.
- **Gouvernance** ⚠ : raccorder un S/4 client à un compte trial personnel doit être validé par la sécurité du client (Q6). L'API Business Partner expose des **données personnelles** : utiliser un système / mandant de test avec des données fictives.
- **Secrets** : service keys, mots de passe du user technique et du Cloud Connector restent hors du dépôt DocNRT (utiliser un coffre ou des variables Postman de type *secret*).
- **Poste** : navigateur récent, Postman (desktop recommandé), droits administrateur sur la machine qui hébergera le Cloud Connector.

---

## 1. Création du compte BTP trial

1. Ouvrir `https://www.sap.com/products/technology-platform/trial.html` et lancer l'essai gratuit.
2. Se connecter avec son **SAP Universal ID** ou créer un compte : e-mail, prénom, nom, pays, mot de passe → cliquer sur le lien d'activation reçu par e-mail → vérifier le numéro de téléphone si demandé → accepter les conditions.
3. Dans le cockpit trial, cliquer **Go To Your Trial Account**.
4. Choisir la région (Q6) : **US East (VA) – AWS** ou **Singapore – Azure** (seules régions avec Integration Suite trial) → **Create Account**.
5. Attendre la fin du provisioning → **Continue**.
6. Résultat attendu : un global account `<xxxx>trial`, un subaccount `trial`, une org Cloud Foundry et un space `dev`.
7. Ouvrir le subaccount → **Overview** → renseigner **P1 à P6**.

---

## 2. Subaccount

### 2.A Option recommandée : utiliser le subaccount `trial` créé automatiquement

Cloud Foundry y est déjà activé, le space `dev` existe et la plupart des entitlements sont pré-affectés. Passer directement au §2.3 pour contrôle.

### 2.B Option : créer un subaccount dédié

1. Global account → **Account Explorer** → **Create** → **Subaccount**.
2. Renseigner :

| Champ | Valeur |
|---|---|
| Display Name | `demo-s4-is-cap` |
| Subdomain | `demo-s4-is-cap-<suffixe>` (unique, minuscules, sans espace) |
| Region | La même région que le compte trial (P1) |
| Beta features | Non coché |

3. **Create**, attendre le statut *Created*.
4. Subaccount → **Overview** → section *Cloud Foundry Environment* → **Enable Cloud Foundry** : laisser *Plan* et *Landscape* proposés, saisir *Instance Name* et *Org Name* → **Create**.
5. **Cloud Foundry** → **Spaces** → **Create Space** : *Space Name* `dev`, cocher **Space Manager** et **Space Developer** → **Create**.
6. ⚠ Integration Suite n'accepte qu'un seul tenant par compte trial : ne le souscrire que dans **un** subaccount.

### 2.3 Entitlements

Subaccount → **Entitlements** → **Edit** → **Add Service Plans** → rechercher chaque service, cocher le plan → **Add N Service Plans** → **Save**.

| Service (libellé cockpit) | Plan | Usage |
|---|---|---|
| Integration Suite | `trial` (Application) | Tenant Integration Suite |
| SAP Process Integration Runtime | `integration-flow` | Client OAuth utilisé par Postman |
| SAP Business Application Studio | `trial` (Application) | IDE |
| SAP Build Work Zone, standard edition | Plan de type **Application / Subscription** (pas le plan de type Instance) | Site Work Zone |
| Cloud Identity Services | `default` (Application) | Tenant IAS trial (et **pas** le plan `application`) |
| SAP HANA Cloud | `tools` (Application) **et** `hana` | Outil HANA Cloud Central + instance BDD |
| SAP HANA Schemas & HDI Containers | `hdi-shared` | Container HDI de l'app CAP |
| Authorization and Trust Management Service | `application` | XSUAA de l'app CAP |
| Destination Service | `lite` | Destinations |
| Connectivity Service | `lite` | Variante CAP → S/4 (§12.6) |
| HTML5 Application Repository Service | `app-host` (et `app-runtime`) | Hébergement de l'UI |
| Cloud Foundry Runtime | `MEMORY` | Quota mémoire |

Si un service n'apparaît pas : vérifier la région (P1) et l'âge du compte trial (Integration Suite peut ne plus apparaître sur un compte de plus de 90 jours).

---

## 3. Identité : SAP Cloud Identity Services et trust OIDC (prérequis Work Zone)

1. Subaccount → **Services** → **Instances and Subscriptions** → **Create** :
   - *Service* : **Cloud Identity Services**
   - *Plan* : **default**
   - → **Create**
2. Ouvrir l'e-mail reçu → **Activate Your Account** → définir le mot de passe administrateur du tenant. Noter l'URL du tenant (**P14**).
3. Subaccount → **Security** → **Trust Configuration** → **Establish Trust** :
   - Sélectionner le tenant P14 → **Next**
   - Domaine : conserver la proposition → **Next**
   - *Name* : `IAS-trial` ; *Description* libre ; **Available for User Logon** : coché ; **Create Shadow Users During Logon** : coché → **Next** → **Finish**
4. Vérifier qu'une nouvelle ligne de type **OpenID Connect** apparaît, statut **Active**.
5. Attendre une quinzaine de minutes avant de souscrire Work Zone (§4.2).
6. **Point clé — role collections par fournisseur d'identité** : une role collection est affectée à un couple *utilisateur + identity provider*. Après le trust, deux connexions coexistent : *Default Identity Provider* (SAP ID) et le tenant IAS. Work Zone et l'app CAP ouverte depuis Work Zone utilisent **IAS**.
   - **Security** → **Users** → **Create** : *User Name* = ton e-mail ; *Identity Provider* = tenant IAS (P14) ; *E-Mail* = ton e-mail → **Create**.
   - Toutes les role collections « Work Zone » et « app CAP » des sections suivantes doivent être affectées à **cet** utilisateur (origine IAS).

---

## 4. Souscriptions et role collections

Chemin commun : subaccount → **Services** → **Instances and Subscriptions** → **Create**. Affectation des rôles : **Security** → **Users** → utilisateur → **Assign Role Collection**. Après chaque affectation : **se déconnecter / reconnecter** (ou vider le cache).

### 4.1 SAP Business Application Studio

1. *Service* : **SAP Business Application Studio** ; *Plan* : **trial** → **Create** (souvent déjà souscrit dans le subaccount `trial`).
2. Role collections sur l'utilisateur **Default Identity Provider** : `Business_Application_Studio_Developer`, `Business_Application_Studio_Administrator`.
3. À la connexion à BAS, si une page de choix d'IdP s'affiche, choisir celui sur lequel ces rôles sont affectés.

### 4.2 SAP Build Work Zone, standard edition

1. *Service* : **SAP Build Work Zone, standard edition** ; *Plan* : plan de type **Application / Subscription** → **Create**.
2. Si l'erreur « *You haven't configured your subaccount… to trust a tenant of SAP Cloud Identity Services using OpenID Connect* » apparaît → reprendre le §3 dans l'ordre (souscription IAS `default`, activation, trust, attente), puis resouscrire.
3. Role collection `Launchpad_Admin` → utilisateur **origine IAS** (et, par confort, utilisateur Default).

### 4.3 Integration Suite

1. *Service* : **Integration Suite** ; *Plan* : **trial** → **Create**.
2. Role collection `Integration_Provisioner` → ton utilisateur.
3. Se reconnecter.

### 4.4 SAP HANA Cloud (si persistance CAP, Q4)

1. *Service* : **SAP HANA Cloud** ; *Plan* : **tools** → **Create**.
2. Role collection `SAP HANA Cloud Administrator`.

---

## 5. Integration Suite : activation et client OAuth pour Postman

### 5.1 Activer Cloud Integration

1. **Instances and Subscriptions** → **Integration Suite** → **Go to Application**.
2. Page d'accueil → **Add Capabilities** (⚠ libellé susceptible de changer, ex. *Manage Capabilities*).
3. Cocher **Build Integration Scenarios** (Cloud Integration) → **Next** → options par défaut → **Activate**.
4. Attendre le statut *Active* (plusieurs minutes).
5. Optionnel (Q5) : **Manage APIs** si la démo doit inclure API Management.

> Un booster Integration Suite existe (global account → **Boosters**) et automatise 5.2–5.3 ; le tutoriel SAP indique qu'il n'est fiable qu'en région Singapore. Le pas-à-pas manuel ci-dessous fonctionne dans les deux régions.

### 5.2 Role collections Cloud Integration

Après activation, affecter à ton utilisateur : `PI_Administrator`, `PI_Business_Expert`, `PI_Integration_Developer` → se reconnecter.

### 5.3 Instance « SAP Process Integration Runtime » (client OAuth de l'expéditeur)

**Instances and Subscriptions** → **Create** :

| Champ | Valeur |
|---|---|
| Service | SAP Process Integration Runtime |
| Plan | `integration-flow` |
| Runtime Environment | Cloud Foundry (space `dev`) |
| Instance Name | `pi-iflow-postman` |
| Roles (écran Parameters) | `ESBMessaging.send` |
| Grant-types | `Client Credentials` |

→ **Create**.

### 5.4 Service key

1. Ligne de l'instance → **…** → **Create Service Key** : *Name* `pi-iflow-postman-key` ; *Key Type* **ClientId/Secret** → **Create**.
2. **View** → relever dans le bloc `oauth` : `url` (URL runtime), `tokenurl`, `clientid`, `clientsecret` → **P15**.

> Sur un tenant trial, l'authentification entrante par certificat client n'est pas supportée : rester en client credentials.

---

## 6. Côté S/4HANA 2021 FPS02

### 6.1 Relever les informations de connexion

| Transaction | Action | Résultat |
|---|---|---|
| `SMICM` | *Goto* → *Services* | Port **HTTPS** actif → P8 |
| `RZ11` | Paramètre `icm/host_name_full` → *Display* | FQDN du serveur → base de P7 |
| `SCC4` | Liste des mandants | Mandant de démo → P9 |
| `SICF` | Service `/default_host/sap/opu/odata` | Doit être actif (clic droit → *Activate Service* sinon) |

⚠ Selon l'hébergement (Q1), le FQDN joignable depuis le Cloud Connector peut différer de `icm/host_name_full` (Web Dispatcher, load balancer, nom réseau interne). **Faire confirmer P7/P8 par l'équipe Basis / SAP ECS.**

### 6.2 Activer le service OData (exemple `API_BUSINESS_PARTNER`)

1. `/IWFND/MAINT_SERVICE` → vérifier si le service est déjà présent dans la liste.
2. Sinon **Add Service** :
   - *System Alias* : `LOCAL` → **Get Services**
   - *Technical Service Name* : `API_BUSINESS_PARTNER` → sélectionner → **Add Selected Services**
   - *Package Assignment* : `$TMP` (démo) ou package transportable → **OK**
3. Dans la liste, sélectionner le service → vérifier que l'**ICF Node** est vert (sinon *ICF Node* → *Activate*).
4. **Noter** le *Technical Service Name* et la *Version* affichés (utilisés au §6.5).

### 6.3 Tester et récupérer les métadonnées

1. `/IWFND/GW_CLIENT` → *HTTP Method* `GET` → *Request URI* :
   `/sap/opu/odata/sap/API_BUSINESS_PARTNER/A_BusinessPartner?$top=5&$format=json` → **Execute** → attendu : `200`.
2. *Request URI* `/sap/opu/odata/sap/API_BUSINESS_PARTNER/$metadata` → **Execute** → **Response in Browser** / sauvegarder le fichier `API_BUSINESS_PARTNER.edmx` (utilisé au §12.6).

### 6.4 Utilisateur technique (`SU01`)

| Onglet / champ | Valeur |
|---|---|
| User | `ZBTP_DEMO` (à aligner sur tes conventions) |
| Address | Nom, e-mail d'équipe |
| Logon Data > User Type | **System** (B) |
| Logon Data > Password | Mot de passe fort → P10 |
| Roles | Rôle du §6.5 |

### 6.5 Rôle d'autorisations (`PFCG`)

1. `PFCG` → *Role* `ZBTP_DEMO_API_BP` → **Single Role** → description → sauvegarder.
2. Onglet **Menu** → *Insert Node* → **Authorization Default** :
   - *Authorization Default* : `TADIR Service` ; *Program ID* `R3TR` ; *Object Type* `IWSG` → F4 → sélectionner l'entrée correspondant au Technical Service Name / version notés au §6.2.
   - Recommencer avec *Object Type* `IWSV` → F4 → `API_BUSINESS_PARTNER` / version notée.
3. Onglet **Authorizations** → *Change Authorization Data* → compléter les champs ouverts proposés (objets `B_BUPA_*` en affichage, activité `03`) → **Generate**.
4. Onglet **User** → `ZBTP_DEMO` → **User Comparison**.

### 6.6 Traces et diagnostics côté S/4

| Transaction | Usage |
|---|---|
| `SU53` (*Display for other user*) | Dernier refus d'autorisation du user technique |
| `STAUTHTRACE` | Trace d'autorisations |
| `/IWFND/ERROR_LOG` | Erreurs Gateway (hub) |
| `/IWBEP/ERROR_LOG` | Erreurs Gateway (backend) |
| `SMICM` → *Goto* → *Trace File* | Trace ICM (HTTP) |
| `SM21` | Journal système |
| `SU01` | Statut du user (verrouillé, mot de passe) |

---

## 7. Cloud Connector

### 7.1 Prérequis (dépend de Q1)

- Machine Windows ou Linux 64 bits dans un réseau qui voit le S/4 (P7:P8).
- **JDK complet** (pas un JRE) : SAP JVM 8 ou une version de SapMachine supportée → vérifier la page *Prerequisites* de la doc SAP BTP Connectivity avant l'installation.
- Flux : Cloud Connector → S/4 en HTTPS (P7:P8) ; Cloud Connector → Internet en HTTPS 443 sortant vers la région BTP (via proxy si besoin) ; poste admin → Cloud Connector port 8443.

### 7.2 Installation

1. Télécharger depuis `https://tools.hana.ondemand.com/#cloud` → section **Cloud Connector** → installeur Windows (MSI) ou Linux (RPM). La version *portable* est réservée aux tests.
2. **Windows** : lancer le MSI → répertoire d'installation → port `8443` → chemin du JDK → cocher le démarrage après installation.
3. **Linux** : `sudo rpm -i com.sap.scc-ui-<version>.x86_64.rpm` puis vérifier le service (`systemctl status scc_daemon`).

### 7.3 Première connexion

1. Navigateur → `https://<hote_cloud_connector>:8443`.
2. Identifiants initiaux : `Administrator` / `manage` → changer immédiatement le mot de passe.
3. *Installation type* : **Master (Primary Installation)** → **Save**.
4. Si proxy d'entreprise : renseigner *HTTPS Proxy* (hôte, port, user).

### 7.4 Rattacher le subaccount

**Méthode A (si proposée par ton cockpit)** : cockpit → subaccount → **Connectivity** → **Cloud Connectors** → **Download Authentication Data** ; puis dans le Cloud Connector → **Add Subaccount** → *Configure using authentication data from file* → charger le fichier → saisir *Location ID* et *Description* → **Save**. Méthode à privilégier avec un SAP Universal ID.

**Méthode B (manuelle)** : **Add Subaccount** :

| Champ | Valeur |
|---|---|
| Region | Région du subaccount (P1) |
| Subaccount | **GUID** du subaccount (P2), pas son nom |
| Display Name | `BTP trial demo` |
| Subaccount User | Ton login BTP (e-mail) |
| Password | Ton mot de passe (si échec avec Universal ID → méthode A) |
| Location ID | `S4PCE` (P12) — facultatif mais recommandé ; à reporter à l'identique dans la destination et l'iFlow |
| Description | Libre |

**Contrôles** : statut **Connected** (vert) dans le Cloud Connector ; dans le cockpit, **Connectivity** → **Cloud Connectors** affiche le connecteur avec son Location ID.

### 7.5 Mapping système (Cloud To On-Premise)

Menu **Cloud To On-Premise** → onglet **Access Control** → **+** (*Add System Mapping*) :

| Champ | Valeur |
|---|---|
| Back-end Type | **ABAP System** |
| Protocol | **HTTPS** |
| Internal Host | P7 |
| Internal Port | P8 |
| Virtual Host | `s4h-virtual` (P11) |
| Virtual Port | `44300` (P11) |
| Principal Type | **None** (basic auth via user technique) |
| Host In Request Header | **Use Virtual Host** |
| Description | `S/4HANA 2021 FPS02 demo` |
| Check Internal Host | Coché |

→ **Finish**. Attendu : statut **Reachable**.

### 7.6 Ressources exposées

Sélectionner le mapping → tableau **Resources** → **+** :

| Champ | Valeur |
|---|---|
| URL Path | `/sap/opu/odata/sap/API_BUSINESS_PARTNER` (principe du moindre privilège) |
| Enabled | Coché |
| Access Policy | **Path And All Sub-Paths** |

→ **Save**.

### 7.7 Certificat du back-end et supervision

- Par défaut, tant que la liste blanche du *Trust Store* back-end est vide (**Configuration** → **On Premise**), le Cloud Connector accepte le certificat serveur du S/4.
- Supervision : menus **Monitor**, **Audit**, **Log And Trace Files** (niveau de trace à augmenter temporairement en cas de problème).

---

## 8. Destination BTP vers le S/4 (pour BAS et l'app CAP)

Subaccount → **Connectivity** → **Destinations** → **Create** (⚠ selon la version : *New Destination* / *From Scratch*) :

| Champ | Valeur |
|---|---|
| Name | `S4H_PCE` (P13) |
| Type | HTTP |
| Description | `S/4HANA 2021 FPS02 via Cloud Connector` |
| URL | `http://s4h-virtual:44300` |
| Proxy Type | **OnPremise** |
| Authentication | **BasicAuthentication** |
| Location ID | P12 (si défini au §7.4) |
| User / Password | P10 |

**Additional Properties** (**New Property**) :

| Propriété | Valeur |
|---|---|
| `sap-client` | P9 |
| `WebIDEEnabled` | `true` |
| `WebIDEUsage` | `odata_abap` |
| `HTML5.DynamicDestination` | `true` |

→ **Save** → **Check Connection**.

- URL en `http://` : le segment BTP → Cloud Connector est chiffré par le tunnel ; le protocole vers le S/4 est celui du mapping (HTTPS, §7.5).
- Un retour `200` (ou `401`/`403`) prouve que la requête atteint le S/4 ; un échec de connexion pointe vers le Cloud Connector ou le Location ID.
- Cloud Integration **n'utilise pas** cette destination : l'iFlow porte ses propres paramètres (§9).

---

## 9. iFlow Cloud Integration

### 9.1 Identifiants S/4 dans le tenant

Integration Suite → **Monitor** → **Integrations and APIs** → **Security Material** → **Create** → **User Credentials** :

| Champ | Valeur |
|---|---|
| Name | `S4H_PCE_BASIC` |
| Type | User Credentials |
| User / Password | P10 |

→ **Deploy**.

### 9.2 Test de connectivité

**Monitor** → **Integrations and APIs** → **Connectivity Tests** → onglet **Cloud Connector** → *Location ID* = P12 → **Send** → attendu : succès.

### 9.3 Package et iFlow

1. **Design** → **Integrations and APIs** → **Create** : *Name* `Demo S4 PCE`, *Technical Name* (auto), *Short Description* → **Save**.
2. Onglet **Artifacts** → **Add** → **Integration Flow** : *Name* `IF_S4_BP_Read` → **OK** → ouvrir → **Edit**.

### 9.4 Modélisation

1. **Sender** : relier le participant *Sender* à *Start* → adapter **HTTPS** → onglet **Connection** :

| Champ | Valeur |
|---|---|
| Address | `/s4/bp` |
| Authorization | User Role |
| User Role | `ESBMessaging.send` |
| CSRF Protected | Décoché (démo en GET) |

2. Palette → **Call** → **External Call** → **Request Reply** : le placer entre *Start* et *End*.
3. Ajouter un participant **Receiver** (renommer `S4HANA`), le relier au *Request Reply* → adapter **HTTP** → onglet **Connection** :

| Champ | Valeur |
|---|---|
| Address | `http://s4h-virtual:44300/sap/opu/odata/sap/API_BUSINESS_PARTNER/A_BusinessPartner` |
| Query | `$top=10&$format=json&sap-client=<P9>` |
| Proxy Type | **On-Premise** |
| Location ID | P12 |
| Method | GET |
| Authentication | Basic |
| Credential Name | `S4H_PCE_BASIC` |

Variante : adapter **OData** (V2) avec les mêmes paramètres de connexion ; onglet **Processing** → *Operation* `Query (GET)`, *Resource Path* `A_BusinessPartner`.

### 9.5 Déploiement

1. **Save** → **Deploy**.
2. **Monitor** → **Integrations and APIs** → **Manage Integration Content** → statut **Started** → onglet **Endpoints** → copier l'URL (forme `<url runtime>/http/s4/bp`).
3. Pour la démo : **Log Configuration** → niveau **Trace** (actif temporairement) pour afficher les payloads dans le monitoring.

---

## 10. Test Postman

1. **Environments** → créer `BTP-trial-IS` avec les variables `cpi_endpoint` (§9.5), `token_url`, `client_id`, `client_secret` (type *secret*) (§5.4).
2. **New** → **HTTP Request** → `GET {{cpi_endpoint}}`.
3. Onglet **Authorization** → *Type* **OAuth 2.0** → *Add authorization data to* **Request Headers** → **Configure New Token** :

| Champ | Valeur |
|---|---|
| Token Name | `cpi-trial` |
| Grant Type | Client Credentials |
| Access Token URL | `{{token_url}}` |
| Client ID | `{{client_id}}` |
| Client Secret | `{{client_secret}}` |
| Scope | Vide |
| Client Authentication | Send as Basic Auth header |

4. **Get New Access Token** → **Proceed** → **Use Token** → **Send**.
5. Attendu : `200 OK` + JSON (`d.results`) contenant les business partners.
6. Variante plus simple : *Type* **Basic Auth**, *Username* = `clientid`, *Password* = `clientsecret`.
7. Contrôle : Integration Suite → **Monitor** → **Monitor Message Processing** → message *Completed* (payload visible si niveau Trace).

---

## 11. HANA Cloud (si persistance CAP, Q4)

1. **Instances and Subscriptions** → **SAP HANA Cloud** (tools) → ouvrir **SAP HANA Cloud Central** → **Create Instance** :
   - *Type* : **SAP HANA Cloud, SAP HANA Database**
   - *Instance Name* : `hana-demo`
   - *Administrator Password* (user `DBADMIN`)
   - *Allowed connections* : **Allow all IP addresses** (démo uniquement)
   - → **Create** → attendre **Running**.
2. Si l'instance est créée au niveau subaccount et non dans le space CF : **Manage Configuration** → **Instance Mapping** → **Add** : *Environment Type* Cloud Foundry ; *Environment Instance ID* = Org ID (P5) ; *Environment Group* = GUID du space (`cf space dev --guid`) → **Save**.
3. Chaque jour d'utilisation : **Start** dans HANA Cloud Central.

---

## 12. Application CAP dans BAS

### 12.1 Dev space

1. **Instances and Subscriptions** → **SAP Business Application Studio** → **Go to Application**.
2. **Create Dev Space** : *Name* `demo_cap` ; type **Full-Stack Cloud Application** → **Create Dev Space** → attendre **Running** → ouvrir.

### 12.2 Projet (Node.js, exemple bookshop — Q4)

Terminal BAS :

```sh
cds init demo-cap --nodejs --add sample
cd demo-cap
npm install
cds watch
```

Ouvrir l'aperçu (port 4004) : vérifier les services OData et l'aperçu Fiori.

### 12.3 UI Fiori (si app propre plutôt que les apps du sample)

1. Palette de commandes → **Fiori: Open Application Generator**.
2. *Template* : **List Report Page** → *Data Source* : **Use a Local CAP Project** → projet + service → entité principale.
3. *Project Attributes* : module name, titre, namespace ; **Add FLP configuration : Yes** (*Semantic Object*, *Action*, *Title*) ; configuration de déploiement : la laisser à `cds add workzone` (§12.4).
4. Générer les apps **avant** l'étape 12.4.

### 12.4 Préparation production et Work Zone

```sh
cds add hana,xsuaa,mta,workzone
```

Contrôler ensuite :

| Fichier | Contenu attendu |
|---|---|
| `mta.yaml` | Module `srv`, module de déploiement DB, module de déploiement du contenu HTML5, ressources HDI (`hdi-shared`), XSUAA, HTML5 repo (`app-host`), destination |
| `app/<app>/webapp/manifest.json` | `sap.app.crossNavigation.inbounds` (semantic object / action) et `sap.cloud.service` |
| `app/<app>/xs-app.json`, `ui5-deploy.yaml` | Présents |
| `xs-security.json` | Role templates et role collections (ex. `admin`) |

- Si la ressource HTML5 `app-host` manque : `cds add html5-repo`.
- Si `crossNavigation` manque : palette → **Fiori: Add Fiori Launchpad Configuration**.

### 12.5 Rôles de l'application

CAP crée par défaut une role collection `admin-<org>-<space>` pour le rôle `admin` du sample. Après déploiement : **Security** → **Users** → utilisateur **origine IAS** → **Assign Role Collection** → cette role collection.

### 12.6 Variante : consommer le S/4 depuis CAP (Q4)

```sh
cds add connectivity,destination
npm add @sap-cloud-sdk/http-client @sap-cloud-sdk/connectivity @sap-cloud-sdk/resilience
cds import API_BUSINESS_PARTNER.edmx --as cds
```

Compléter `package.json` (l'entrée est créée par `cds import`) :

```json
"cds": {
  "requires": {
    "API_BUSINESS_PARTNER": {
      "kind": "odata-v2",
      "model": "srv/external/API_BUSINESS_PARTNER",
      "[production]": {
        "credentials": {
          "destination": "S4H_PCE",
          "path": "/sap/opu/odata/sap/API_BUSINESS_PARTNER"
        }
      }
    }
  }
}
```

`srv/s4-service.cds` :

```cds
using { API_BUSINESS_PARTNER as bp } from './external/API_BUSINESS_PARTNER';

service S4Service @(requires: 'authenticated-user') {
  @readonly
  entity BusinessPartners as projection on bp.A_BusinessPartner {
    key BusinessPartner, BusinessPartnerFullName, BusinessPartnerCategory
  };
}
```

`srv/s4-service.js` :

```js
const cds = require('@sap/cds');

module.exports = class S4Service extends cds.ApplicationService {
  async init() {
    const bp = await cds.connect.to('API_BUSINESS_PARTNER');
    this.on('READ', 'BusinessPartners', req => bp.run(req.query));
    return super.init();
  }
};
```

En local, `cds watch` simule (mock) le service distant ; l'appel réel au S/4 se teste après déploiement.

### 12.7 Build et déploiement

1. Connexion CF : palette → **CF: Login to Cloud Foundry** (endpoint P4, org P5, space `dev`) ou terminal `cf login -a <P4> --sso`.
2. Figer les dépendances : `npm install --package-lock-only`.
3. Déployer : `cds up`.
   - Si `cds up` n'est pas reconnu : `npm i -g @sap/cds-dk`, ou équivalent manuel `mbt build` puis `cf deploy mta_archives/<archive>.mtar`.
4. Contrôles :
   - `cf apps` : application `srv` *started* ; le déployeur DB s'arrête après exécution (normal).
   - Cockpit → **HTML5 Applications** (⚠ ou **HTML5** → **Application Repository**) : les apps sont listées.

---

## 13. Publication dans SAP Build Work Zone

1. **Instances and Subscriptions** → **SAP Build Work Zone, standard edition** → **Go to Application** (connexion IAS).
2. **Channel Manager** → fournisseur **HTML5 Apps** → icône de rafraîchissement / **Update Content** → attendre le statut à jour.
3. **Content Manager** → **Content Explorer** → **HTML5 Apps** → cocher les apps → **Add to My Content**.
4. **My Content** → **Create** → **Group** : *Title* `Démo CAP S/4` → affecter les apps → **Save**.
5. **My Content** → rôle **Everyone** → **Edit** → affecter les apps → **Save**.
6. **Site Directory** → **Create Site** → *Site Name* `Demo BTP S4` → **Create** → vérifier dans les paramètres du site que le rôle **Everyone** est affecté → **Go to site**.
7. Cliquer sur la tuile → l'app s'ouvre avec ses données.

---

## 14. Recette

| # | Contrôle | Attendu |
|---|---|---|
| R1 | Cloud Connector : subaccount | Connected |
| R2 | Cloud Connector : mapping | Reachable |
| R3 | Destination `S4H_PCE` → Check Connection | Réponse du S/4 |
| R4 | Connectivity Test CPI (Location ID) | Succès |
| R5 | Postman → iFlow | `200` + JSON |
| R6 | Monitor Message Processing | Completed |
| R7 | `cf apps` | `srv` started |
| R8 | Site Work Zone | Tuile visible, app fonctionnelle |
| R9 | (Variante 12.6) entité `BusinessPartners` dans l'app CAP | Données S/4 |

---

## 15. Dépannage

| Symptôme | Cause probable | Action |
|---|---|---|
| Souscription Work Zone en échec « OIDC trust » | Trust IAS absent ou pas encore propagé | §3 dans l'ordre (plan `default`, activation, trust), attendre ~15 min, resouscrire |
| « Access Denied » Work Zone / Integration Suite | Role collection absente ou affectée sur le mauvais IdP | §3.6, se reconnecter |
| Postman `401` | Token, `clientsecret` ou URL de token | Service key §5.4 |
| Postman `403` | Rôle ≠ `ESBMessaging.send`, ou CSRF coché | §5.3 / §9.4 |
| Postman `404` | Address de l'adapter HTTPS ou iFlow non démarré | §9.5 |
| Erreur receiver « connection refused » / tunnel | Cloud Connector déconnecté, Location ID différent | R1, R4, P12 identique partout |
| `403` venant du Cloud Connector | Chemin non exposé dans *Resources* | §7.6, audit Cloud Connector |
| `401` venant du S/4 | User technique verrouillé / mauvais mot de passe / mandant | `SU01`, `sap-client` |
| `403` venant du S/4 | Autorisations `S_SERVICE` ou métier | `SU53` (autre user), `STAUTHTRACE`, §6.5 |
| `500` venant du S/4 | Erreur Gateway | `/IWFND/ERROR_LOG`, `/IWBEP/ERROR_LOG` |
| App absente du Content Explorer | Canal HTML5 non rafraîchi, `sap.cloud.service` ou `crossNavigation` manquant | §13.2, §12.4 |
| Tuile absente du site | Apps non affectées au rôle Everyone | §13.5 |
| App CAP en `403` depuis Work Zone | Role collection sur l'utilisateur Default au lieu d'IAS | §12.5 |
| Déploiement HDI en échec | Instance HANA arrêtée ou non mappée au space | §11 |

---

## 16. Questions ouvertes (à trancher avant exécution)

- **Q1** — Le S/4 2021 FPS02 est-il un S/4HANA Cloud, private edition (RISE, opéré par SAP ECS) ou un S/4 on-premise / hébergé ? Où sera installé le Cloud Connector et quels flux sont ouverts (vers P7:P8, et 443 sortant vers BTP) ?
- **Q2** — P7, P8, P9 ; qui a les droits `SU01` / `PFCG` / `/IWFND/MAINT_SERVICE` ; basic auth avec user technique acceptée, ou propagation de principal requise ?
- **Q3** — Quelle(s) API pour la démo, en lecture seule ou aussi en écriture (POST avec jeton CSRF) ?
- **Q4** — App CAP : Node.js ou Java ; persistance HANA Cloud, consommation S/4 via destination, ou les deux ; reprise de l'app bookstore de démo existante ?
- **Q5** — Integration Suite : Cloud Integration seul, ou aussi API Management devant l'iFlow ?
- **Q6** — Région trial ; validation par le client du raccordement de son S/4 à un compte trial personnel.

---

## Annexe A — Récapitulatif des role collections

| Role collection | Pour | Utilisateur (IdP) |
|---|---|---|
| Subaccount Administrator | Admin subaccount, Cloud Connector, destinations | Default |
| `Business_Application_Studio_Developer` / `_Administrator` | BAS | Default |
| `Integration_Provisioner` | Activer les capacités Integration Suite | Default |
| `PI_Administrator`, `PI_Business_Expert`, `PI_Integration_Developer` | Cloud Integration | Default |
| `SAP HANA Cloud Administrator` | HANA Cloud Central | Default |
| `Launchpad_Admin` | Site Manager Work Zone | **IAS** (+ Default) |
| `admin-<org>-<space>` (ou celles de `xs-security.json`) | App CAP | **IAS** |

## Annexe B — Transactions S/4 utilisées

`SMICM`, `RZ11`, `SCC4`, `SICF`, `/IWFND/MAINT_SERVICE`, `/IWFND/GW_CLIENT`, `SU01`, `PFCG`, `SU53`, `STAUTHTRACE`, `/IWFND/ERROR_LOG`, `/IWBEP/ERROR_LOG`, `SM21`.

## Annexe C — Automatisation Terraform

Les étapes 2.B, 2.3, 3.1, 3.3, 4.x (souscriptions et affectations de role collections) et 5.3 relèvent d'objets gérés par le provider Terraform SAP BTP (subaccount, entitlements, subscriptions, service instances, trust, role collection assignments). Pour la destination du §8, vérifier la couverture dans la version du provider utilisée. Restent manuels : activation des capacités Integration Suite, configuration du Cloud Connector, iFlow, contenu du site Work Zone et paramétrage S/4.
