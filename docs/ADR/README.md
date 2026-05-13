# Index des Architecture Decision Records — SupEvents

Ce dossier contient les décisions d'architecture structurantes du projet SupEvents, documentées au format Nygard. Chaque ADR est immutable une fois accepté : une décision révisée donne lieu à un nouvel ADR qui remplace le précédent (statut "Remplacé par ADR-XXX").

---

## Index

| N° | Titre | Statut | Date | Auteurs | Lien |
|----|-------|--------|------|---------|------|
| ADR-001 | La concurrence sur la jauge de places est gérée par verrou pessimiste PostgreSQL | Accepté | 2025-05-13 | Sherazad | [ADR-001.md](./ADR-001.md) |
| ADR-002 | L'idempotence des opérations d'écriture est garantie par déduplication serveur via Redis | Accepté | 2025-05-13 | Sherazad | [ADR-002.md](./ADR-002.md) |
| ADR-003 | Les événements asynchrones RabbitMQ utilisent un backoff exponentiel avec dead letter queue dédiée | Accepté | 2025-05-13 | Sherazad | [ADR-003.md](./ADR-003.md) |

---

## Liens vers la DCT

- §6 — Vue des données : `docs/06-vue-donnees.md`
- §7 — Conception détaillée par module : `docs/07-modules.md`
- §8 — Interfaces & contrats d'API : `docs/08-interfaces-api.md`

---

## Template

Le template Nygard utilisé pour tous les ADR est disponible dans [`template.md`](./template.md).
