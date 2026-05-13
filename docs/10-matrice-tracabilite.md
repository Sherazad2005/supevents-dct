# §10 — Matrice de traçabilité

La présente matrice croise l'ensemble des 21 exigences du cahier des charges SupEvents (13 exigences fonctionnelles EF + 8 exigences non-fonctionnelles ENF) avec les éléments produits dans la DCT au cours des TP 1.4 à 1.6. Pour chaque exigence, la matrice indique le ou les modules de §7 qui l'implémentent, les sections de la DCT qui la documentent, et les ADR qui justifient les choix architecturaux associés. Une exigence non encore couverte par la DCT est explicitement marquée ⚠️ avec indication du TP de traitement prévu. La lecture par colonne permet de vérifier qu'aucun module n'est orphelin et qu'aucune exigence n'est fantôme — non liée à une implémentation concrète.

---

## §10.1 — Matrice principale

| Réf. | Intitulé (court) | Module(s) | Section(s) DCT | ADR |
|------|-----------------|-----------|----------------|-----|
| EF-01 | Création d'événements | EventModule | §6.4 (Event, Organizer), §8 (`POST /api/v1/events`) | — |
| EF-02 | Catalogue public | EventModule | §6.4 (Event, Category), §8 (`GET /api/v1/events`, `GET /api/v1/events/search`, `GET /api/v1/events/:id`) | — |
| EF-03 | Inscription à un événement | TicketModule | §6.4 (Ticket, WaitingListEntry), §7.1 (TicketModule), §8 (`POST /api/v1/tickets`) | ADR-001 |
| EF-04 | Paiement en ligne | PaymentModule | §6.4 (Payment), §7.2 (PaymentModule), §8 (`POST /api/v1/payments/initiate`, `POST /api/v1/payments/webhook`) | ADR-002 |
| EF-05 | Gestion de la jauge | TicketModule, EventModule | §6.4 (Event.seats_available), §7.1 (algorithme allocation concurrente) | ADR-001 |
| EF-06 | Annulation de ticket | TicketModule, PaymentModule | §6.4 (Ticket.status), §7.1 (cas limite annulation post-paiement), §8 (`DELETE /api/v1/tickets/:id`) | — |
| EF-07 | Notifications utilisateur | NotificationModule | §6.4 (Notification), §8 (événements `ticket.confirmed`, `payment.failed`) | ADR-003 |
| EF-08 | Authentification SSO | AuthModule | §6.4 (User, RefreshToken), §7.3 (AuthModule), §8 (`GET /api/v1/auth/callback`, `POST /api/v1/auth/refresh`, `POST /api/v1/auth/logout`) | ADR-002 |
| EF-09 | Gestion des rôles | AuthModule, UserModule | §6.4 (User.role, Organizer.status), §7.3 (propagation du contexte d'authentification), §8 (colonne Auth du tableau synoptique) | — |
| EF-10 | Tableau de bord organisateur | EventModule, TicketModule | §8 (`GET /api/v1/organizer/events`, `GET /api/v1/organizer/events/:id/participants`, `GET /api/v1/organizer/events/:id/kpi`) | — |
| EF-11 | Export participants CSV | EventModule, TicketModule | §8 (`GET /api/v1/organizer/events/:id/export`), §6.4 (convention nommage S3) | — |
| EF-12 | Validation organisateur | UserModule | §6.4 (Organizer.status), §8 (`PATCH /api/v1/admin/organizers/:id/validate`, `PATCH /api/v1/admin/organizers/:id/revoke`) | — |
| EF-13 | Liste d'attente | TicketModule | §6.4 (WaitingListEntry), §8 (`POST /api/v1/tickets` — cas sold out), §7.1 (cas limite dernière place) | ADR-001 |
| ENF-01 | Performance p95 < 500 ms | TicketModule, EventModule, AuthModule | §9.2 (ENF-01 latence, ENF-03 pic concurrence), §6.4 (index PostgreSQL) | ADR-001 |
| ENF-02 | Disponibilité 99,5 % | Infrastructure transverse | §9.2 (ENF-02 disponibilité, budget d'erreur 3h36) | — |
| ENF-03 | 500 utilisateurs simultanés | TicketModule, PaymentModule | §9.2 (ENF-03 capacité pic), §7.1 (verrou pessimiste), §7.2 (idempotence paiement) | ADR-001, ADR-002 |
| ENF-04 | Sécurité des paiements PCI-DSS | PaymentModule | §6.4 (Payment — rappel CDC, pas de stockage carte), §7.2 (délégation Stripe), §9.1.1 (STRIDE flux inscription payante) | ADR-002 |
| ENF-05 | Conformité RGPD | UserModule, AuthModule | §6.4 (marquage RGPD), §9.3 (registre, mécanismes, procédures) | — |
| ENF-06 | Sécurité applicative | AuthModule, PaymentModule | §9.1.1 (STRIDE inscription payante), §9.1.2 (STRIDE SSO), §7.3 (validation OIDC, CSRF) | ADR-002 |
| ENF-07 | Traçabilité des opérations | Tous les modules | §9.3.c (audit log immuable), §7.1 (R — Repudiation), §7.2 (R — Repudiation) | — |
| ENF-08 | Scalabilité horizontale | Infrastructure transverse | §9.2 (ENF-02 multi-instance), §6.4 (justification stockage — Redis sessions, RabbitMQ) | ADR-003 |

---

## §10.2 — Synthèse

**Couverture EF : 13 / 13** — toutes les exigences fonctionnelles sont tracées vers au moins un module et une section DCT.

**Couverture ENF : 8 / 8** — toutes les exigences non-fonctionnelles sont tracées.

**Exigences couvertes partiellement (documentation incomplète) :**

| Réf. | Lacune identifiée | Traitement prévu |
|------|------------------|-----------------|
| EF-01 | Le cycle de vie complet de l'événement (`draft → published → cancelled → archived`) n'est pas détaillé en §7 — `EventModule` n'a pas de fiche complète (seul `TicketModule`, `PaymentModule`, `AuthModule` ont été traités en TP 1.5) | TP 1.8 — compléter la fiche `EventModule` en §7.4 |
| EF-07 | `NotificationModule` n'a pas de fiche §7 détaillée — seuls les événements publiés/consommés sont documentés en §8 | TP 1.8 — compléter la fiche `NotificationModule` en §7.5 |
| EF-09 | `UserModule` n'a pas de fiche §7 — la gestion des rôles est documentée en §6.4 et §8 mais sans architecture interne ni cas limites | TP 1.8 — compléter la fiche `UserModule` en §7.6 |
| EF-12 | Même lacune que EF-09 — validation organisateur dépend de `UserModule` non détaillé | TP 1.8 (même fiche) |
| ENF-02 | La stratégie de déploiement multi-instance et le failover PostgreSQL/Redis ne sont pas formalisés dans un ADR | TP 1.7 — ADR-004 sur la stratégie de déploiement et haute disponibilité |
| ENF-08 | Aucun ADR ne documente la stratégie de scalabilité horizontale (choix de conteneurisation, orchestration) | TP 1.7 — ADR-004 (même ADR que ENF-02) |

---

**Vérification par colonne — apparitions des modules dans la matrice :**

| Module | Nombre de lignes | Statut |
|--------|-----------------|--------|
| `TicketModule` | 6 (EF-03, EF-05, EF-06, EF-10, EF-11, EF-13, ENF-01, ENF-03) | ✅ Responsabilité cohérente |
| `PaymentModule` | 4 (EF-04, EF-06, ENF-03, ENF-04) | ✅ Responsabilité cohérente |
| `AuthModule` | 5 (EF-08, EF-09, ENF-01, ENF-05, ENF-06) | ✅ Responsabilité cohérente |
| `EventModule` | 5 (EF-01, EF-02, EF-05, EF-10, EF-11, ENF-01) | ✅ Responsabilité cohérente |
| `NotificationModule` | 1 (EF-07) | ⚠️ Fiche §7 manquante — à compléter en TP 1.8 |
| `UserModule` | 2 (EF-09, EF-12, ENF-05) | ⚠️ Fiche §7 manquante — à compléter en TP 1.8 |
