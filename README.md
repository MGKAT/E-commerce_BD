# Refonte NoSQL de la plateforme LDLC — Architecture Polyglotte

> **Projet NSL NoSQL** | Conception d'une architecture de données distribuée et polyglotte pour l'e-commerce high-tech.

**Binôme :**
* **FONGANG TENE Klein**
* **WAFO Olivier Dylan**

---

## Contexte et Objectifs

**LDLC** est une plateforme e-commerce spécialisée dans le matériel informatique et high-tech gérant :
* **+ 27 000 références produits** réparties sur **+ 600 marques**
* **Des millions de visites mensuelles** avec de forts pics de trafic lors des sorties de nouveaux composants
* **Jusqu'à 25 000 colis expédiés par jour**

Face aux limites d'une base relationnelle monolithique, ce projet propose une **persistance polyglotte** : chaque domaine fonctionnel est associé à la base NoSQL la plus adaptée à son profil d'accès, sa volumétrie, ses exigences de latence et ses garanties de cohérence (théorème CAP).

### Objectifs principaux
1. **Évolutivité du catalogue :** Faire évoluer les fiches produits sans migrations rigides à chaque nouvelle catégorie.
2. **Latence sub-milliseconde :** Servir les paniers et sessions avec expiration automatique (TTL) et partage entre serveurs stateless.
3. **Ingestion massive :** Absorber des millions d'événements de navigation quotidiens et conserver l'historique des commandes.
4. **Recommandations complexes :** Modéliser les compatibilités matérielles et parcours multi-sauts (produits similaires / achetés ensemble).
5. **Découplage clair :** Séparer les sources de vérité des vues dérivées via des flux asynchrones explicites.

---

## Architecture Globale

```text
                      ┌───────────────────────────┐
                      │  Clients Web / Mobile     │
                      │         (CDN)             │
                      └─────────────┬─────────────┘
                                    │
                                    ▼
                      ┌───────────────────────────┐
                      │ Load Balancer/API Gateway │
                      │   (Services Stateless)    │
                      └─────────────┬─────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        │                           │                           │
        ▼                           ▼                           ▼
┌───────────────┐           ┌───────────────┐           ┌───────────────┐
│   Catalogue   │           │ Panier/Session│           │Tracking & Cmd │
│    Service    │           │    Service    │           │    Service    │
└───────┬───────┘           └───────┬───────┘           └───────┬───────┘
        │                           │                           │
        ▼                           ▼                           ▼
┌───────────────┐           ┌───────────────┐           ┌───────────────┐
│    MongoDB    │           │     Redis     │           │   Cassandra   │
│  (ReplicaSet) │           │(Cluster/Sent.)│           │  (RF=3, AP)   │
│      CP       │           │   Sub-ms TTL  │           │ Logs & Orders │
└───────┬───────┘           └───────┬───────┘           └───────┬───────┘
        │                           │                           │
        │ Change Streams            │ Checkout (Sync)           │ Batch / Events
        └───────────────────────────┼───────────────────────────┘
                                    │
                                    ▼
                            ┌───────────────┐
                            │     Neo4j     │
                            │(CausalCluster)│
                            │Recommandations│
                            └───────────────┘


##  Décisions Technologiques par Lot

### Lot 1 – Catalogue produits (MongoDB)

Gestion des fiches produits avec des caractéristiques variables selon les catégories.
Fonctionnalités :
- Stockage des produits et marques.
- Gestion des caractéristiques techniques.
- Gestion des avis clients.
- Recherche et filtrage des produits.
- Gestion des compatibilités.

### Lot 2 – Paniers et sessions (Redis)

Gestion des données temporaires nécessitant des performances élevées.
Fonctionnalités :
- Gestion des sessions utilisateurs.
- Stockage des paniers.
- Gestion des paniers anonymes.
- Expiration automatique des données avec TTL.
- Fusion des paniers lors de la connexion.

### Lot 3 – Logs, analytics et commandes (Cassandra)

Gestion des événements de navigation et des historiques.
Fonctionnalités :
- Enregistrement des événements utilisateurs.
- Suivi des consultations produits.
- Analyse des tendances de navigation.
- Historique des commandes clients.
- Gestion des données volumineuses et de leur rétention.

### Lot 4 – Recommandations produits (Neo4j)

Modélisation des relations entre les produits, clients et caractéristiques techniques.
Fonctionnalités :
- Recommandation de produits similaires.
- Identification des produits fréquemment achetés ensemble.
- Recherche de compatibilités entre composants.
- Recommandations collaboratives.
