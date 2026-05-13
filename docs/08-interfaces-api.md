# §8 — Interfaces & contrats d'API

## Tableau synoptique des endpoints REST

| Méthode | Chemin | Description | Auth | Codes retour | Dépendances aval |
|---------|--------|-------------|------|--------------|-----------------|
| `GET` | `/api/v1/auth/callback` | Callback OIDC après authentification SSO | Public | 302, 400, 401, 500 | AuthService, UserService |
| `POST` | `/api/v1/auth/refresh` | Renouvellement de l'access token via refresh token | Public (cookie httpOnly) | 200, 401, 403, 429, 500 | AuthService, Redis |
| `POST` | `/api/v1/auth/logout` | Révocation du refresh token et invalidation de session | JWT | 204, 401, 500 | AuthService, Redis |
| `GET` | `/api/v1/events` | Liste paginée des événements publiés (filtres : catégorie, date, lieu, prix) | Public | 200, 400, 500 | EventService |
| `GET` | `/api/v1/events/search` | Recherche full-text dans les événements | Public | 200, 400, 500 | EventService |
| `GET` | `/api/v1/events/:id` | Détail d'un événement | Public | 200, 404, 500 | EventService |
| `POST` | `/api/v1/events` | Création d'un nouvel événement | `JWT + role:organizer` | 201, 400, 401, 403, 422, 500 | EventService, S3/MinIO |
| `PATCH` | `/api/v1/events/:id` | Modification partielle d'un événement | `JWT + role:organizer` | 200, 400, 401, 403, 404, 409, 422, 500 | EventService |
| `DELETE` | `/api/v1/events/:id` | Annulation d'un événement (transition → cancelled) | `JWT + role:organizer` | 200, 401, 403, 404, 409, 500 | EventService, RabbitMQ |
| `POST` | `/api/v1/tickets` | Inscription à un événement (création de ticket) | JWT | 201, 400, 401, 409, 422, 500 | TicketService, EventService, PaymentService, RabbitMQ |
| `GET` | `/api/v1/tickets/:id` | Statut et détail d'un ticket | JWT | 200, 401, 403, 404, 500 | TicketService |
| `DELETE` | `/api/v1/tickets/:id` | Annulation d'un ticket | JWT | 200, 400, 401, 403, 404, 409, 500 | TicketService, PaymentService, RabbitMQ |
| `POST` | `/api/v1/payments/initiate` | Initiation d'un PaymentIntent Stripe | JWT | 201, 400, 401, 402, 409, 422, 500 | PaymentService, Stripe, Redis |
| `POST` | `/api/v1/payments/webhook` | Réception des événements Stripe (payment_intent.succeeded, payment_intent.payment_failed, etc.) | **HMAC** | 200, 400, 401, 500 | PaymentService, TicketService, RabbitMQ |
| `GET` | `/api/v1/organizer/events` | Liste des événements de l'organisateur connecté | `JWT + role:organizer` | 200, 401, 403, 500 | EventService |
| `GET` | `/api/v1/organizer/events/:id/participants` | Liste des participants inscrits à un événement | `JWT + role:organizer` | 200, 401, 403, 404, 500 | TicketService, UserService |
| `GET` | `/api/v1/organizer/events/:id/export` | Export CSV des participants | `JWT + role:organizer` | 200, 401, 403, 404, 500 | TicketService, UserService, S3/MinIO |
| `GET` | `/api/v1/organizer/events/:id/kpi` | KPIs de l'événement (taux de remplissage, revenus) | `JWT + role:organizer` | 200, 401, 403, 404, 500 | EventService, PaymentService |
| `GET` | `/api/v1/admin/organizers` | Liste de tous les organisateurs (filtre par statut) | `JWT + role:admin` | 200, 401, 403, 500 | OrganizerService |
| `PATCH` | `/api/v1/admin/organizers/:id/validate` | Validation d'un organisateur (pending → active) | `JWT + role:admin` | 200, 401, 403, 404, 409, 500 | OrganizerService |
| `PATCH` | `/api/v1/admin/organizers/:id/revoke` | Révocation d'un organisateur (active → revoked) | `JWT + role:admin` | 200, 401, 403, 404, 409, 500 | OrganizerService |

> **Note critique** : Le endpoint `POST /api/v1/payments/webhook` est authentifié par **signature HMAC-SHA256** vérifiée côté serveur (header `Stripe-Signature`), et non par JWT. Toute requête dont la signature ne correspond pas est rejetée avec un `401`.

---

## Conventions transverses

### Format des erreurs (RFC 7807 Problem Details)

Toutes les erreurs retournent un corps JSON conforme à la RFC 7807 :

```json
{
  "type": "https://supevents.io/errors/ticket-already-exists",
  "title": "Conflict",
  "status": 409,
  "detail": "Un ticket actif existe déjà pour cet utilisateur et cet événement.",
  "instance": "/api/v1/tickets",
  "traceId": "a1b2c3d4-..."
}
```

### Stratégie de versioning

Toutes les routes sont préfixées par `/api/v1/`. En cas de breaking change, une nouvelle version `/api/v2/` est déployée en parallèle. L'ancienne version est maintenue pendant 6 mois minimum avec un header de dépréciation `Deprecation: true` et `Sunset: <date>`.

### Rate limiting et quotas

| Scope | Limite | Fenêtre | Action |
|-------|--------|---------|--------|
| Auth (login / refresh) | 10 req | 1 min / IP | 429 Too Many Requests |
| Création de ticket | 5 req | 1 min / user | 429 Too Many Requests |
| API publique (lecture) | 100 req | 1 min / IP | 429 Too Many Requests |
| API organisateur | 200 req | 1 min / user | 429 Too Many Requests |

Le rate limiting est implémenté via Redis (token bucket). Le header `Retry-After` est inclus dans les réponses 429.

---

## Événements asynchrones

### `ticket.confirmed`

#### Fiche descriptive

| Champ | Valeur |
|-------|--------|
| **Nom** | `ticket.confirmed` |
| **Producteur** | `TicketService` — publié après confirmation atomique : décrément de `seats_available` + création du `Ticket` en status `confirmed` (soit après capture Stripe pour un ticket payant, soit immédiatement pour un ticket gratuit) |
| **Topic / exchange** | `tickets.events.v1` (RabbitMQ, exchange type `topic`, routing key `ticket.confirmed`) |
| **Consommateurs connus** | `NotificationService` : envoie un email de confirmation avec QR code au détenteur du ticket ; `AnalyticsService` : incrémente les compteurs de conversion et les revenus temps réel ; `WaitingListService` : aucune action directe (déclenché plutôt par annulation) |
| **Garantie de livraison** | `at-least-once` — le producteur confirme uniquement après `basic.ack` RabbitMQ. Les consommateurs doivent être idempotents : la présence du `ticket_id` en Redis permet de détecter et ignorer les doublons. |
| **Stratégie de retry** | Backoff exponentiel : 1s, 2s, 4s, 8s, max 5 tentatives. En cas d'échec définitif, message routé vers la dead letter queue `tickets.events.dlq.v1` pour analyse manuelle et alerte. |

#### Schéma JSON du payload

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "TicketConfirmedEvent",
  "type": "object",
  "required": [
    "event_id",
    "event_name",
    "event_type",
    "ticket_id",
    "user_id",
    "ticket_type",
    "amount_paid",
    "currency",
    "confirmed_at",
    "occurred_at",
    "correlation_id"
  ],
  "properties": {
    "event_id": {
      "type": "string",
      "format": "uuid",
      "description": "Identifiant de l'événement métier SupEvents"
    },
    "event_name": {
      "type": "string",
      "const": "ticket.confirmed"
    },
    "event_type": {
      "type": "string",
      "const": "domain_event"
    },
    "ticket_id": {
      "type": "string",
      "format": "uuid",
      "description": "Identifiant unique du ticket confirmé"
    },
    "user_id": {
      "type": "string",
      "format": "uuid",
      "description": "Identifiant de l'utilisateur détenteur du ticket"
    },
    "supevents_event_id": {
      "type": "string",
      "format": "uuid",
      "description": "Identifiant de l'événement (Event) SupEvents concerné"
    },
    "ticket_type": {
      "type": "string",
      "enum": ["free", "paid"],
      "description": "Type de ticket"
    },
    "amount_paid": {
      "type": "number",
      "minimum": 0,
      "description": "Montant payé (0 pour les tickets gratuits)"
    },
    "currency": {
      "type": "string",
      "pattern": "^[A-Z]{3}$",
      "description": "Code ISO 4217 de la devise"
    },
    "payment_intent_id": {
      "type": ["string", "null"],
      "description": "Référence Stripe, null pour les tickets gratuits"
    },
    "confirmed_at": {
      "type": "string",
      "format": "date-time",
      "description": "Date ISO 8601 de confirmation du ticket"
    },
    "occurred_at": {
      "type": "string",
      "format": "date-time",
      "description": "Date ISO 8601 de publication de l'événement"
    },
    "correlation_id": {
      "type": "string",
      "format": "uuid",
      "description": "Identifiant de corrélation pour le tracing distribué"
    }
  },
  "additionalProperties": false
}
```

---

### `payment.failed`

#### Fiche descriptive

| Champ | Valeur |
|-------|--------|
| **Nom** | `payment.failed` |
| **Producteur** | `PaymentService` — publié à la réception du webhook Stripe `payment_intent.payment_failed` ou `payment_intent.canceled`, après mise à jour du `Payment` en status `failed` et du `Ticket` en status `pending` (remis disponible pour retry) |
| **Topic / exchange** | `payments.events.v1` (RabbitMQ, exchange type `topic`, routing key `payment.failed`) |
| **Consommateurs connus** | `NotificationService` : envoie un email d'échec de paiement à l'utilisateur avec un lien pour réessayer ; `TicketService` : remet `seats_available` à +1 sur l'`Event` correspondant si le ticket est définitivement abandonné (après expiration du délai de retry utilisateur) ; `AnalyticsService` : comptabilise les taux d'échec de paiement par événement |
| **Garantie de livraison** | `at-least-once` — idempotence garantie côté consommateurs via vérification du statut courant du `Payment` avant toute action (si déjà `succeeded`, le message est ignoré). |
| **Stratégie de retry** | Backoff linéaire : 5s entre chaque tentative, max 3 tentatives. Dead letter queue : `payments.events.dlq.v1`. Une alerte PagerDuty est déclenchée pour tout message en DLQ. |

#### Schéma JSON du payload

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "PaymentFailedEvent",
  "type": "object",
  "required": [
    "event_id",
    "event_name",
    "event_type",
    "payment_id",
    "ticket_id",
    "user_id",
    "payment_intent_id",
    "amount",
    "currency",
    "failure_code",
    "failure_message",
    "failed_at",
    "occurred_at",
    "correlation_id"
  ],
  "properties": {
    "event_id": {
      "type": "string",
      "format": "uuid",
      "description": "Identifiant unique de l'événement métier"
    },
    "event_name": {
      "type": "string",
      "const": "payment.failed"
    },
    "event_type": {
      "type": "string",
      "const": "domain_event"
    },
    "payment_id": {
      "type": "string",
      "format": "uuid",
      "description": "Identifiant interne du Payment SupEvents"
    },
    "ticket_id": {
      "type": "string",
      "format": "uuid",
      "description": "Identifiant du Ticket associé au paiement échoué"
    },
    "user_id": {
      "type": "string",
      "format": "uuid",
      "description": "Identifiant de l'utilisateur concerné"
    },
    "supevents_event_id": {
      "type": "string",
      "format": "uuid",
      "description": "Identifiant de l'événement (Event) SupEvents concerné"
    },
    "payment_intent_id": {
      "type": "string",
      "description": "Référence Stripe du PaymentIntent échoué"
    },
    "amount": {
      "type": "number",
      "minimum": 0,
      "description": "Montant tenté"
    },
    "currency": {
      "type": "string",
      "pattern": "^[A-Z]{3}$",
      "description": "Code ISO 4217"
    },
    "failure_code": {
      "type": "string",
      "description": "Code d'erreur Stripe (ex: card_declined, insufficient_funds)"
    },
    "failure_message": {
      "type": "string",
      "description": "Message d'erreur localisé pour l'utilisateur final"
    },
    "is_retryable": {
      "type": "boolean",
      "description": "Indique si l'utilisateur peut réessayer le paiement"
    },
    "failed_at": {
      "type": "string",
      "format": "date-time",
      "description": "Date ISO 8601 de l'échec du paiement"
    },
    "occurred_at": {
      "type": "string",
      "format": "date-time",
      "description": "Date ISO 8601 de publication de l'événement"
    },
    "correlation_id": {
      "type": "string",
      "format": "uuid",
      "description": "Identifiant de corrélation pour le tracing distribué"
    }
  },
  "additionalProperties": false
}
```
