# Comptes : utilisateurs, organisations, rôles

Voir [`README.md`](README.md) pour le glossaire et les conventions.

**État** : première version, 29 septembre 2026. Le rattachement des
contrôleurs à une organisation est **acté** ; le reste est une
**proposition**.

## Diagramme

```mermaid
classDiagram
  direction LR

  class User {
    identityProviderSubject
    email
    displayName
    locale
    status : UserStatus
  }

  class Organization {
    name
    type : OrganizationType
    timezone
  }

  class Membership {
    role : Role
    joinedAt
  }

  class Invitation {
    email
    role : Role
    expiresAt
    acceptedAt
  }

  class ControllerEnrollment {
    enrolledAt
    endedAt
  }

  class Controller {
    serialNumber
  }

  class Role {
    <<enumeration>>
    owner
    admin
    operator
    viewer
  }

  class OrganizationType {
    <<enumeration>>
    customer
  }

  class UserStatus {
    <<enumeration>>
    active
    disabled
  }

  User "1" -- "0..*" Membership
  Organization "1" -- "1..*" Membership
  Organization "1" -- "0..*" Invitation
  User "1" --> "0..*" Invitation : a invité
  Organization "1" -- "0..*" ControllerEnrollment
  ControllerEnrollment "0..*" -- "1" Controller
```

`Controller` et `ControllerEnrollment` sont détaillés dans
[`controleurs.md`](controleurs.md). Une organisation peut avoir plusieurs
contrôleurs (décision du 29 septembre 2026), chacun par un enrollment.

## Entités

### `User`
Une personne qui se connecte à l'application web. Un utilisateur peut être
membre de **plusieurs** organisations (par exemple sa propre exploitation et
celle d'un proche qu'il aide), et de zéro juste après son inscription.

- `email` : adresse vérifiée par le fournisseur d'identité, recopiée pour les
  notifications (enrollment, transfert, abonnement).
- `locale` : langue de l'interface et des notifications.
- `status = disabled` : compte bloqué sans supprimer son historique (actions
  passées, audit).
- **Authentification déléguée à un fournisseur OpenID Connect**
  (proposition) : inscription, vérification du mail, MFA, mot de passe
  oublié, connexion Google/Apple. `identityProviderSubject` est
  l'identifiant (`sub`) de l'utilisateur chez ce fournisseur ; **aucun mot
  de passe utilisateur n'est stocké** dans notre base. Le choix du
  fournisseur reste ouvert.
- Un utilisateur peut exister **sans abonnement** : l'inscription au cloud
  et l'abonnement sont distincts (voir `facturation.md`).

### `Organization`
Le client, propriétaire des contrôleurs. Une organisation existe même avec
un seul membre (un particulier a son organisation personnelle) : c'est ce
qui permet d'ajouter des membres plus tard sans rien migrer.

- `type` : seulement `customer` aujourd'hui (vente directe). La valeur
  `partner` est réservée aux évolutions installateur / revendeur (voir
  `README.md`).
- `timezone` : les scénarios d'arrosage et les graphes s'expriment en heure
  locale ; un contrôleur hérite du fuseau de son organisation sauf
  indication contraire (à confirmer avec `installation.md`).

### `Membership`
Lien utilisateur ↔ organisation, porteur du rôle. Un utilisateur a au plus
une adhésion par organisation.

### `Invitation`
Pour ajouter un membre à une organisation par email, y compris quelqu'un
qui n'a pas encore de compte. Devient une `Membership` à l'acceptation ;
expire sinon.

## Rôles proposés

| Rôle | Voir mesures et état | Piloter (lancer/arrêter un arrosage) | Configurer (zones, scénarios) | Gérer membres et contrôleurs | Supprimer l'organisation, facturation |
|---|---|---|---|---|---|
| `viewer` | ✅ | | | | |
| `operator` | ✅ | ✅ | | | |
| `admin` | ✅ | ✅ | ✅ | ✅ | |
| `owner` | ✅ | ✅ | ✅ | ✅ | ✅ |

Règles :
- Une organisation a **toujours au moins un `owner`** : le dernier ne peut ni
  partir ni être rétrogradé.
- Les rôles sont **par organisation**, pas par contrôleur, dans cette
  version. Des droits par contrôleur ou par zone (un salarié qui ne voit que
  sa serre) sont possibles plus tard, en ajoutant une portée à
  `Membership`.

## Règle d'accès

> Un utilisateur a accès à un contrôleur si et seulement s'il est membre de
> l'organisation qui en a l'**enrollment actif** ; ce qu'il peut faire
> dépend de son rôle. Pour l'historique, il n'accède qu'aux données
> produites pendant les enrollments de ses organisations.

L'accès aux fonctions cloud dépend aussi de l'abonnement de l'organisation
(`facturation.md`) ; jamais le fonctionnement local ni les mises à jour.

À implémenter dans **un seul service** de l'API, et non dispersée dans les
requêtes : c'est ce qui permettra d'ajouter plus tard les délégations
(`AccessGrant`) en ne modifiant qu'un endroit.
