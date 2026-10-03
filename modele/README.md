# Modèle de données Moscata

Modèle **conceptuel** du domaine : entités métier, relations, cardinalités.
Il ne décrit ni tables ni types SQL : le schéma physique (PostgreSQL +
TimescaleDB, Prisma) vit dans `moscata-cloud-api` (`prisma/`) et en est
dérivé. Il concerne aussi le firmware (topic `config`) et le web.

Diagrammes en [Mermaid](https://mermaid.js.org/) (`classDiagram`), rendus
nativement par GitHub et VS Code. Un fichier par domaine :

| Fichier | Domaine | État |
|---|---|---|
| [`comptes.md`](comptes.md) | Utilisateurs, organisations, rôles | première version |
| [`controleurs.md`](controleurs.md) | Contrôleurs, enrollment, identifiants MQTT, firmware | première version |
| [`facturation.md`](facturation.md) | Vente des contrôleurs, abonnement cloud | ébauche |
| `installation.md` | Zones, devices (vannes, sondes, pompes), scénarios | à faire |
| `mesures.md` | Télémétrie, événements, commandes | à faire |

## Conventions

- Même esprit que le reste de `moscata-architecture` : décisions **datées**,
  distinction nette entre décision **actée** et **proposition**.
- Noms d'entités en anglais (ils deviendront les noms du code), texte en
  français.
- On ne modélise que ce qui a un sens métier ; les détails techniques
  (index, clés techniques, colonnes d'audit `createdAt`/`updatedAt`) sont
  laissés au schéma physique.

## Glossaire

| Terme | Entité | Définition |
|---|---|---|
| **Contrôleur** | `Controller` | Un Moscata Modulo (ESP32-S3) installé sur site. C'est lui qui se connecte au broker MQTT. |
| **Numéro de série** | `Controller.serialNumber` | Identifiant unique inscrit physiquement sur le contrôleur. **Fait foi.** Sert aussi d'identifiant MQTT (`controllerId`). |
| **controllerId** | = `serialNumber` | Identifiant du contrôleur dans les topics (`moscata/{controllerId}/...`) et nom d'utilisateur MQTT. Sans `/`, `+` ni `#`. Appelé `deviceId` jusqu'au 3 octobre 2026. |
| **Device** | `Device` (à venir) | Un équipement piloté ou lu par un contrôleur : vanne, sonde, pompe. Même sens que dans le firmware et dans le champ `devices` du topic `config`. |
| **Organisation** | `Organization` | Le compte client dans le cloud : exploitation, entreprise ou particulier. Peut avoir plusieurs contrôleurs. |
| **Utilisateur** | `User` | Une personne qui se connecte à l'application web. |
| **Adhésion** | `Membership` | Le lien entre un utilisateur et une organisation, avec un rôle. |
| **Code de revendication** | `Controller.claimCodeHash` | Code secret aléatoire imprimé sous le couvercle (texte + QR code), requis pour rattacher le contrôleur à une organisation. |
| **Enrollment** | `ControllerEnrollment` | Une période pendant laquelle un contrôleur est rattaché à une organisation dans le cloud. Un contrôleur en a au plus une active, et un historique. |
| **Revendication** | `ClaimRequest` | Une tentative de rattachement d'un contrôleur à une organisation, acceptée seulement dans la fenêtre qui suit un reset usine. |
| **Vente** | `Sale` | La vente neuve d'un contrôleur par Moscata, avec sa facture. Une seule par contrôleur. |
| **Abonnement** | `Subscription` | L'abonnement mensuel au cloud, optionnel et indépendant de l'achat. |
| **Identifiant de connexion** | `DeviceCredential` | Ce qui permet à un contrôleur de s'authentifier auprès du broker (mot de passe aujourd'hui, certificat client peut-être demain). |
| **Version de firmware** | `FirmwareRelease` | Une version publiée du firmware, distribuable par mise à jour à distance (OTA). |
| **Mise à jour** | `FirmwareUpdate` | Une tentative de mise à jour d'un contrôleur donné vers une version donnée. |

## Décisions

**29 septembre 2026 : actées.**
- Chaque contrôleur est rattaché à une **organisation**, jamais directement
  à un utilisateur. Une organisation peut avoir **plusieurs** contrôleurs.
- **Vente directe uniquement** à ce jour : pas d'installateur ni de
  revendeur professionnel.
- Chaque contrôleur est vendu avec **une facture unique**.
- Un contrôleur payé **fonctionne à vie**, sans cloud ni abonnement : aucune
  vérification de licence ne doit bloquer son fonctionnement local.
- Un contrôleur peut être **revendu d'occasion**. Tous les contrôleurs en
  circulation sont connus, et le **changement de propriétaire** doit être
  géré.
- L'**identifiant unique inscrit physiquement** sur le contrôleur fait foi.
- Le **cloud est optionnel** et se facture par **abonnement mensuel**,
  souscrit **séparément** de l'achat : on peut acheter sans s'abonner et
  s'abonner plus tard. Un acheteur d'occasion peut s'inscrire et s'abonner.
- **Mises à jour firmware OTA gratuites**, avec ou sans abonnement, dès que
  le contrôleur est joignable. Chaque contrôleur a une **date de mise à
  jour** de son firmware.
- **Enrollment** (voir `controleurs.md`) : **reset usine** du contrôleur
  (appui long sur le bouton, déjà prévu), **puis rattachement** à
  l'organisation avec l'étiquette sous le couvercle (numéro de série, code
  de revendication, QR code), dans une fenêtre limitée après le reset. Le
  mail sert à la vérification du compte et aux notifications (dont
  l'ancien propriétaire en cas de transfert).
- Le **reset usine est irréversible** : fait physiquement sur le
  contrôleur, il le remet à l'état neuf, sans retour en arrière possible
  depuis le Moscata. Il **clôt l'enrollment** : même sans revente, le
  propriétaire réenrôle le contrôleur comme s'il venait de l'acheter. Seuls
  les abonnés pourront restaurer une **sauvegarde de configuration**
  conservée dans le cloud.
- Un reset fait **hors ligne** n'est connu du cloud qu'à la reconnexion
  suivante (peut-être jamais) : décalage **accepté**, le Moscata restant
  opérationnel sans cloud.

**3 octobre 2026 : actée.**
- Nommage : `Controller` pour le Moscata Modulo, `Device` pour vannes et
  sondes (comme le firmware). `deviceId` est renommé **`controllerId`** dans
  `../MQTT-Topics.md`, `moscata-mqtt` (documentation, scripts) et
  `moscata-cloud-api` (documentation). Les ACL utilisent le nom
  d'utilisateur MQTT : ni la configuration du broker ni les topics ne
  changent.

**29 septembre 2026 : propositions (non actées).**
- **Numéro de série = `controllerId`** : un seul identifiant partout.
- **Données cloisonnées par enrollment** : un nouveau propriétaire ne voit
  pas l'historique de l'ancien (voir `controleurs.md`).
- **Utilisateurs via un fournisseur d'identité OpenID Connect** (inscription,
  vérification du mail, MFA, mot de passe oublié, connexion Google/Apple) :
  l'API ne stocke aucun mot de passe utilisateur.
- Rôles d'adhésion, cycle de vie du contrôleur, séparation entre
  contrôleur et identifiants de connexion, entités firmware : voir
  `comptes.md` et `controleurs.md`.

## Questions ouvertes

- **Fenêtre de revendication** après un reset usine : durée, et
  configuration du Wi-Fi après reset (relève du firmware).
- **Fournisseur OpenID Connect** : piste Zitadel (Cloud UE au départ),
  options et méthodes de connexion détaillées dans `comptes.md`.
- **Abonnement** : par organisation ou par contrôleur ? Tarifs, période
  d'essai, que se passe-t-il à l'échéance (données conservées combien de
  temps ?).
- **Factures** : gérées dans ce système ou dans un outil de facturation
  séparé, dont on ne garderait que la référence ?
- **Contrôleur volé** : permettre au propriétaire de le déclarer volé pour
  bloquer toute revendication ?
- **Mot de passe ou mTLS** pour les contrôleurs : à trancher avant la
  production en série (décision de fabrication autant que de sécurité, qui
  relève de `moscata-mqtt`). Le modèle supporte les deux.
- **Authentification dynamique au broker** : avec des milliers de
  contrôleurs, les comptes MQTT ne peuvent plus être créés par script ;
  Mosquitto devra vérifier les identifiants depuis la base ou l'API. Choix à
  faire dans `moscata-mqtt`. Les contrôleurs **non abonnés** doivent aussi
  pouvoir se connecter (mises à jour OTA gratuites).
- **Cyber Resilience Act** (UE) : obligations de mises à jour de sécurité
  pour les produits connectés vendus dans l'UE (principales obligations à
  partir de fin 2027). Période de support à définir ; **à faire vérifier
  par un juriste**.
- **Fréquence de télémétrie** : relève du firmware, à revoir avec lui.
  Impacte `mesures.md`.

## Changements à prévoir dans le contrat MQTT

À reporter dans `../MQTT-Topics.md` une fois validés avec le firmware :
- événement `factory.reset` (publié à la première connexion après un reset
  usine, ouvre la fenêtre de revendication) ;
- `config` vide retained publié par l'API après un reset usine (efface la
  configuration précédente, que le broker redélivrerait sinon) ;
- première connexion après un reset en **session propre** (clean start),
  pour ne pas recevoir les messages en file de la session précédente ;
- commande `firmware.update` (mise à jour OTA).

## Évolutions prévues (non modélisées)

À ajouter seulement quand le besoin sera réel ; le modèle actuel les rend
possibles sans restructuration :
- **Installateur / support tiers** : `Organization.type = partner` et une
  entité `AccessGrant` (organisation cliente → organisation partenaire,
  éventuellement limitée à un contrôleur, datée, révocable par le client).
- **Revendeur professionnel** : stock de contrôleurs chez le revendeur et
  rattachement de la facturation.
- **Journal d'audit** des actions humaines (qui a lancé quel arrosage, qui a
  modifié quel scénario) : utile dès maintenant, indispensable avec des
  tiers. À traiter avec `mesures.md` (commandes).
