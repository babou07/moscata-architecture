# Moscata Modulo — MQTT Topics

> Les valeurs de QoS/retained, la validité des commandes (`expiresAt`), le
> regroupement de la télémétrie et le LWT sont des propositions du
> 27 septembre 2026, à valider avec le firmware.

## Convention

Tous les topics d'un contrôleur Moscata sont préfixés par son identifiant :

```text
moscata/{deviceId}/...
```

Exemple :

```text
moscata/abc123/status
```

Le nom d'utilisateur MQTT d'un contrôleur est son `deviceId` : les ACL du
broker (dépôt `moscata-mqtt`) en dépendent. Un `deviceId` ne doit contenir ni
`/`, ni `+`, ni `#`.

Les payloads sont encodés en **JSON UTF-8**.

---

## Topics

| Topic | Direction | QoS | Retained | Description |
|---|---|---|---|---|
| `moscata/{deviceId}/status` | Moscata → Cloud | 1 | oui | Disponibilité et informations générales du contrôleur |
| `moscata/{deviceId}/state` | Moscata → Cloud | 1 | oui | État courant des zones et périphériques |
| `moscata/{deviceId}/telemetry` | Moscata → Cloud | 0 | non | Mesures des capteurs |
| `moscata/{deviceId}/event` | Moscata → Cloud | 1 | non | Événements, alertes et résultats de commandes |
| `moscata/{deviceId}/command` | Cloud → Moscata | 1 | non | Commandes envoyées au contrôleur |
| `moscata/{deviceId}/config` | Cloud → Moscata | 1 | oui | Configuration du contrôleur |

Les contrôleurs utilisent une session persistante : les messages QoS 1
publiés pendant une déconnexion leur sont livrés à la reconnexion. C'est pour
cela que les commandes portent une durée de validité (voir `command`), et
qu'elles ne sont jamais retained : un message retained serait rejoué à
chaque nouvel abonnement.

---

## `command`

Une commande utilise une enveloppe commune :

```ts
type Command<T = unknown> = {
  id: string;
  type: string;
  timestamp: string; // ISO 8601, date d'émission
  expiresAt: string; // ISO 8601, au-delà la commande ne doit plus être exécutée
  payload: T;
};
```

Exemple :

```json
{
  "id": "cmd-123",
  "type": "zone.start",
  "timestamp": "2026-09-27T05:25:00Z",
  "expiresAt": "2026-09-27T05:26:00Z",
  "payload": {
    "zoneId": "serre-1",
    "duration": 300
  }
}
```

Les différents `type` pourront notamment être :

```ts
type CommandType =
  | "zone.start"
  | "zone.stop"
  | "device.set"
  | "scenario.start"
  | "scenario.stop";
```

Règles côté contrôleur :

- Une commande dont `expiresAt` est dépassé n'est pas exécutée : le
  contrôleur publie un `command.result` avec `status: "rejected"` et
  `error: "expired"`.
- Une commande dont l'`id` a déjà été traité est ignorée : QoS 1 garantit au
  moins une livraison, donc des doublons sont possibles.
- Évaluer `expiresAt` suppose une horloge synchronisée (SNTP). Le
  comportement du firmware quand l'heure n'est pas valide reste à confirmer.

Le cloud choisit un `expiresAt` court, adapté à la commande.

### Suivi, retry et alerte côté cloud

`moscata-cloud-api` enregistre chaque commande envoyée (`id`, statut) et
s'abonne à `event` pour la faire évoluer selon le `command.result` reçu
(`accepted`/`rejected`/`failed`). Si aucun résultat n'arrive avant
`expiresAt` (plus une marge réseau), elle passe en `timed_out`.

- **Commande ponctuelle** (déclenchée par un humain) : retry possible, avec
  un **nouvel** `id` et un **nouveau** `expiresAt` — jamais réutiliser
  l'ancien message. Le firmware dédoublonnant par `id` et rejetant les
  commandes expirées, une éventuelle livraison tardive de l'ancienne
  commande ne fait pas de mal. Nombre de tentatives limité (2-3, avec
  backoff), puis alerte humaine plutôt qu'un dernier essai automatique :
  renvoyer une commande comme `zone.start` en boucle sans limite est
  dangereux si l'appareil reste injoignable puis se reconnecte d'un coup.
- **Scénario récurrent** (arrosage quotidien, etc.) : ne doit **pas**
  dépendre de ce mécanisme. Conformément au principe d'autonomie du firmware
  (voir `CLAUDE.md` de `moscata-mqtt`), un scénario récurrent est configuré
  localement (via `config`, poussé une fois, retained) et exécuté par
  l'horloge du contrôleur, même si le cloud ou le broker sont injoignables
  ce jour-là.

Ce suivi applicatif rend inutile toute sauvegarde de la persistance
Mosquitto (`mosquitto/data`) pour se prémunir d'une commande perdue : une
commande en attente non confirmée est de toute façon détectée par le
timeout et retentée ou remontée en alerte, y compris si le dossier de
persistance du broker est perdu (panne disque du VPS). La vérité
applicative vit dans PostgreSQL (`moscata-cloud-api`), dont la sauvegarde
est un sujet séparé.

---

## `event`

Les événements utilisent également une enveloppe commune :

```ts
type Event<T = unknown> = {
  type: string;
  timestamp: string;
  payload: T;
};
```

Résultat d'une commande :

```ts
type CommandResultPayload = {
  commandId: string;
  status: "accepted" | "rejected" | "failed";
  error?: string;
};
```

Exemple :

```json
{
  "type": "command.result",
  "timestamp": "2026-09-27T05:25:01Z",
  "payload": {
    "commandId": "cmd-123",
    "status": "accepted"
  }
}
```

L'acceptation d'une commande ne représente pas nécessairement l'état physique final. Celui-ci est publié sur `state`.

---

## `state`

Représente l'état courant connu par le contrôleur.

```ts
type State = {
  timestamp: string;
  zones: Record<string, ZoneState>;
};

type ZoneState = {
  status: "idle" | "watering" | "error";
  startedAt?: string;
};
```

Exemple :

```json
{
  "timestamp": "2026-09-27T05:25:02Z",
  "zones": {
    "serre-1": {
      "status": "watering",
      "startedAt": "2026-09-27T05:25:01Z"
    }
  }
}
```

---

## `telemetry`

Transport des mesures provenant des capteurs, indépendamment de leur interface physique : Modbus/RS-485, 4–20 mA, LoRaWAN, etc.

Un seul message regroupe les mesures de tous les capteurs, indexées par `sensorId`, pour limiter le nombre de messages sur la 4G (un message par minute).

```ts
type Telemetry = {
  timestamp: string;
  sensors: Record<string, Record<string, number | boolean | string>>;
};
```

Exemple :

```json
{
  "timestamp": "2026-09-27T05:30:00Z",
  "sensors": {
    "soil-1": {
      "temperature": 18.7,
      "humidity": 43.2
    },
    "level-1": {
      "level": 62.5
    }
  }
}
```

Le `deviceId` désigne toujours le contrôleur (présent dans le topic) ; les capteurs sont identifiés par `sensorId`.

---

## `status`

État général du contrôleur Moscata.

```ts
type Status = {
  online: boolean;
  firmwareVersion?: string;
  uptime?: number;
  network?: "wifi" | "4g" | "ethernet";
};
```

Exemple :

```json
{
  "online": true,
  "firmwareVersion": "0.1.0",
  "uptime": 86400,
  "network": "4g"
}
```

Ce topic est utilisé avec le **MQTT Last Will (LWT)** pour signaler la perte de connexion du contrôleur :

- À chaque connexion, le contrôleur publie sur `status` un message `online: true` (QoS 1, retained).
- Il enregistre à la connexion un LWT sur le même topic : payload `{ "online": false }`, QoS 1, retained. C'est le broker qui le publie si la connexion est perdue sans déconnexion propre.

---

## `config`

Configuration envoyée par le cloud au contrôleur.

Le schéma précis reste à définir avec le modèle de configuration Moscata (`zones`, `devices`, scénarios, etc.).

```ts
type Config = {
  version: number;
  zones: unknown[];
  devices: unknown[];
};
```

---

## Principes

- Les topics restent peu nombreux et stables.
- La nature d'une commande ou d'un événement est portée par `type`, pas par un nouveau topic.
- Les périphériques physiques ne sont pas exposés dans la structure des topics.
- `command` représente une intention.
- `event/command.result` indique si cette intention a été acceptée ou refusée.
- `state` représente l'état effectivement connu du système.
- `telemetry` transporte les mesures.
- Les schémas pourront être partagés entre `moscata-cloud-api`, `moscata-cloud-web` et les outils TypeScript ; le firmware implémentera le même contrat côté ESP32.
