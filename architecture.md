# Architecture Moscata

**Statut : validée le 27 septembre 2026** (proposée le 25 septembre). Ce
document décrit l'architecture cible actée.

## Architecture générale

```
                    ┌──────────────────────┐
                    │     Moscata Cloud    │
                    │                      │
                    │  Next.js  (frontend) │
                    │  NestJS   (API)      │
                    │  PostgreSQL / Prisma │
                    └──────────┬───────────┘
                               │
                              MQTT
                               │
                    ┌──────────▼───────────┐
                    │      Mosquitto       │
                    │     MQTT Broker      │
                    │   VPS / serveur      │
                    └──────────┬───────────┘
                               │
                         MQTT over TLS
                               │
                  Wi-Fi / 4G   │
                               │
                    ┌──────────▼───────────┐
                    │       Moscata        │
                    │      Firmware        │
                    │ ESP32-S3 / ESP-IDF   │
                    │                      │
                    │ GPIO / TCA9539       │
                    │ RS-485 / Modbus      │
                    │ LoRaWAN              │
                    │ capteurs / vannes    │
                    └──────────────────────┘
```

En résumé :

```
ESP-IDF → Mosquitto ← NestJS ← Next.js
```

## Découpage des dépôts git

```
moscata-firmware/
    ESP-IDF
    firmware du Moscata

moscata-mqtt/
    Mosquitto
    config broker
    ACL
    certificats
    Docker / déploiement VPS

moscata-cloud-api/
    NestJS
    Prisma
    PostgreSQL
    MQTT client
    API pour le frontend

moscata-cloud-web/
    Next.js
    interface utilisateur
```

## Répartition des responsabilités

- **Firmware (ESP32-S3)** : reste autonome pour les fonctions locales
  (vannes, scénarios, sécurité, configuration locale). Il se connecte au
  broker en MQTT over TLS, en Wi-Fi et/ou 4G.
- **Mosquitto** : ne contient aucune logique métier. Il transporte les
  messages et applique l'authentification, le TLS et les ACL.
- **NestJS (`moscata-cloud-api`)** : fait le lien entre MQTT, les appareils,
  les utilisateurs et la base de données (PostgreSQL via Prisma). Il agit
  comme client MQTT du broker et expose l'API consommée par le frontend.
- **Next.js (`moscata-cloud-web`)** : fournit l'interface utilisateur cloud.
  Il parle à l'API NestJS, pas directement au broker.

## État actuel (27 septembre 2026)

- **`moscata-mqtt`** : implémenté et déployé sur le VPS OVH, en continu via
  GitHub Actions. Broker accessible en TLS sur `mqtt.moscatamodulo.com:8883`,
  authentification par mot de passe, ACL par appareil actives. Deux comptes
  provisionnés : `proto-1` (prototype de test) et `moscata-api` (droits déjà
  définis pour le futur client MQTT de NestJS, compte créé par avance, non
  utilisé). Détails dans le README de ce dépôt.
- **`moscata-firmware`**, **`moscata-cloud-api`**, **`moscata-cloud-web`** :
  pas encore démarrés.

## Points à définir

- Détails précis du contrat de topics — QoS, retained, validité des
  commandes (`expiresAt`), Last Will (LWT) — proposés dans
  [`MQTT-Topics.md`](MQTT-Topics.md), **à valider avec le firmware** une fois
  celui-ci développé.
- Modèle de configuration Moscata (`zones`, `devices`, scénarios) transporté
  par le topic `config`, non défini (voir `MQTT-Topics.md`). Le modèle de
  données du domaine est en cours dans [`modele/`](modele/README.md)
  (comptes et contrôleurs en première version depuis le 29 septembre 2026).
- Exposition HTTPS de `moscata-cloud-web` et `moscata-cloud-api` sur le VPS
  du broker, et coexistence avec le renouvellement du certificat MQTT (voir
  proposition ci-dessous).

Réglés depuis la version précédente de ce document : l'authentification et
les droits du client MQTT de NestJS (compte `moscata-api`, ACL dédiée), ainsi
que le modèle de sécurité du broker (TLS, mot de passe, ACL par appareil).

### Proposition (28 septembre 2026, non actée) : reverse proxy et port 80

**Problème.** Le certificat Let's Encrypt de `mqtt.moscatamodulo.com` est
renouvelé par certbot en mode standalone (HTTP-01) : certbot écoute
lui-même sur le port 80 pendant la validation. Si `moscata-cloud-web` (et
l'API exposée publiquement) sont déployés sur le même VPS, un reverse proxy
occupera 80/443 en permanence et le renouvellement du broker échouera
silencieusement, jusqu'à l'expiration du certificat (~90 jours) et la
déconnexion des appareils.

**Option recommandée.**
- Un seul reverse proxy, **Caddy**, en conteneur, sur 80/443. Il gère
  automatiquement les certificats du web et de l'API (sous-domaines à
  définir, par exemple `app.moscatamodulo.com` et `api.moscatamodulo.com`).
- Le certificat du broker reste géré par certbot sur l'hôte, mais passe du
  mode standalone au mode **webroot** : certbot écrit le fichier de
  challenge dans un dossier partagé, que Caddy sert sur
  `http://mqtt.moscatamodulo.com/.well-known/acme-challenge/`. Le hook de
  déploiement existant (copie dans `/opt/moscata-mqtt/certs`, redémarrage de
  Mosquitto) est inchangé.
- Les conteneurs web et API ne publient plus de port sur l'hôte : Caddy les
  joint par un réseau Docker commun (aujourd'hui l'API écoute sur
  `127.0.0.1:3000` uniquement, en attendant).
- Avantage : aucun nouveau secret sur le VPS. Contrepartie : le
  renouvellement du broker dépend de Caddy (risque faible : certbot retente
  deux fois par jour à partir de 30 jours avant l'expiration).

**Alternatives écartées pour l'instant.**
- **DNS-01 via l'API OVH** (plugin `certbot-dns-ovh`) : le broker ne dépend
  plus d'aucun port, mais un jeton OVH capable de modifier la zone DNS doit
  être stocké sur le VPS ; compromettre le VPS permettrait de rediriger tout
  `moscatamodulo.com`.
- **Arrêt du proxy pendant le renouvellement** (hooks pre/post certbot) :
  coupure du site à chaque renouvellement, fragile.

**Reste à trancher.** Où vit la configuration du reverse proxy (commune au
web, à l'API et au challenge du broker) : dans `moscata-mqtt`, qui
deviendrait le dépôt « infra VPS », ou dans un dépôt dédié (par exemple
`moscata-proxy`). À décider au moment du déploiement de
`moscata-cloud-web`.
