Vos deux URL diffèrent sur un point qui n'est pas anodin : la première porte un port explicite, la seconde non. Ce ne sont donc probablement pas les mêmes chemins réseau — accès direct à l'ICM d'un côté, Web Dispatcher ou reverse proxy de l'autre. Avant de fouiller les autorisations, il faut lever cette ambiguïté.

Le test qui tranche en trente secondes

Appelez l'application en direct, mais sans le port, c'est-à-dire par la même route que le Launchpad :

https://host/sap/bc/ui5_ui5/sap/zapp/index.html?sap-client=050
Résultat	Conclusion
403	Le problème est le routage / le filtrage d'URL, pas les autorisations
L'application se charge	Le problème est dans le lancement depuis le Launchpad

Si c'est la première ligne, votre Web Dispatcher filtre les chemins autorisés et /sap/bc/ui5_ui5/sap/zapp n'y figure pas. Les applications SAP standard passent parce que leurs chemins sont couverts par les règles livrées ; une BSP déployée dans un nom propre ne l'est pas. C'est à l'équipe Basis, dans les règles de filtrage du dispatcher.

Identifier la requête fautive

En parallèle, ouvrez la console (F12), onglet Réseau, filtrez sur le statut 403 et relevez l'URL exacte. Trois familles, trois diagnostics complètement différents :

URL en 403	Piste
/sap/bc/ui5_ui5/sap/zapp/...	Routage, ou autorisation de démarrage de l'application
/sap/opu/odata4/...	Autorisation sur le groupe de services OData V4
/sap/bc/ui2/...	Autorisation sur les services d'infrastructure du Launchpad

Ouvrez aussi le corps de la réponse 403 : SAP y renvoie généralement un message explicite, bien plus utile que le code seul.

Si c'est une autorisation

Lancez SU53 immédiatement après avoir reproduit l'erreur, dans la même session utilisateur. L'écran nomme l'objet et les valeurs manquantes, ce qui évite toute conjecture.

Le candidat le plus probable est S_SERVICE sur le groupe de services OData V4. C'est un piège spécifique à la V4 : quand vous ajoutez un catalogue à un rôle PFCG, la génération des autorisations par défaut sait déduire les services OData V2 référencés par les target mappings, mais pas les groupes de services V4. L'autorisation n'est donc pas générée, et il faut l'ajouter à la main dans l'onglet Autorisations, avec le type de service IWSG.

Cela expliquerait aussi pourquoi l'appel direct semble fonctionner : la page se charge, la coquille SAPUI5 s'affiche, mais l'appel OData échoue derrière. Vérifiez ce point — l'application affiche-t-elle réellement des données quand vous l'ouvrez en direct, ou seulement un écran vide ?

Complétez avec /IWFND/ERROR_LOG côté front-end, qui journalise les refus d'autorisation sur les services.

C'est une lacune de mon mode opératoire RAP : l'étape E3 mentionne « compléter le rôle avec l'autorisation d'exécution du service » sans préciser que, pour la V4, elle ne se génère pas automatiquement. Je corrigerai ce point.

Deux éléments me feraient gagner du temps : le résultat du test sans port, et l'URL exacte qui remonte en 403 dans l'onglet Réseau.
