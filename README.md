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

```

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

## Livrables

- Document de conception technique.
- Schéma d'architecture globale.
- Modèles de données des différentes bases NoSQL.
- Requêtes types MongoDB, Redis, Cassandra et Neo4j.
- Justification des choix technologiques et des compromis.
- Prototype éventuel de l'architecture.

## Structure du projet

```text
NSL-NoSQL-LDLC/
├── README.md
├── docs/
│   ├── conception_architecture.pdf
│   └── architecture_globale.png
├── mongodb/
│   ├── schemas/
│   └── queries/
├── redis/
│   └── commands/
├── cassandra/
│   ├── schemas/
│   └── queries/
├── neo4j/
│   ├── schemas/
│   └── queries/
└── docker-compose.yml
```
Cette structure représente l'organisation prévue du dépôt et peut être adaptée selon l'avancement du prototype.

## Exemples de requêtes
Les exemples ci-dessous illustrent les opérations principales prévues pour chaque base. Ils sont à adapter aux données et aux schémas effectivement déployés.

### MongoDB — Catalogue

**Insérer un produit**

```javascript
db.products.insertOne({
  _id: "CPU-7800X3D",
  name: "AMD Ryzen 7 7800X3D",
  brand: { id: "amd", name: "AMD" },
  categories: ["composants", "processeurs"],
  price: { amount: 389.95, currency: "EUR" },
  status: "active",
  schema_version: 2,
  attributes: [
    { k: "socket", v: "AM5" },
    { k: "cores", v: 8 }
  ],
  specs: { architecture: "Zen 4", threads: 16, cache_l3_mb: 96 },
  compatible_with: ["MB-B650-TUF"],
  rating: { count: 2, sum: 9 },
  created_at: new Date(),
  updated_at: new Date()
})
```

**Lister les processeurs AMD à moins de 500 €**

```javascript
db.products.find({
  categories: "processeurs",
  "brand.id": "amd",
  "price.amount": { $lt: 500 },
  status: "active"
}).sort({ "price.amount": 1 })
```

**Rechercher les produits ayant un socket AM5**

```javascript
db.products.find({
  attributes: { $elemMatch: { k: "socket", v: "AM5" } }
})
```

**Mettre à jour le prix**

```javascript
db.products.updateOne(
  { _id: "CPU-7800X3D" },
  { $set: { "price.amount": 369.95, updated_at: new Date() } }
)
```

### Redis — Paniers et sessions

**Créer une session et définir son expiration**

```redis
HSET session:9f3a1c user_id u42 ip 127.0.0.1 csrf token123
EXPIRE session:9f3a1c 1800
```

**Ajouter un produit au panier**

```redis
HINCRBY cart:u:42 CPU-7800X3D 1
EXPIRE cart:u:42 604800
```

**Lire le panier**

```redis
HGETALL cart:u:42
```

**Retirer un produit**

```redis
HDEL cart:u:42 CPU-7800X3D
```

**Consulter le TTL**

```redis
TTL cart:u:42
```

### Cassandra — Logs et commandes

**Créer une table d'événements par acteur et par jour**

```sql
CREATE TABLE events_by_actor_day (
    actor_id text,
    day date,
    event_ts timeuuid,
    event_type text,
    product_id text,
    payload text,
    PRIMARY KEY ((actor_id, day), event_ts)
) WITH CLUSTERING ORDER BY (event_ts DESC);
```

**Enregistrer un événement**

```sql
INSERT INTO events_by_actor_day (
    actor_id, day, event_ts, event_type, product_id, payload
) VALUES (
    'u42', '2026-11-18', now(), 'product_view',
    'CPU-7800X3D', '{"source":"catalogue"}'
);
```

**Lire les événements d'un utilisateur pour une journée**

```sql
SELECT event_ts, event_type, product_id, payload
FROM events_by_actor_day
WHERE actor_id = 'u42'
  AND day = '2026-11-18';
```

**Créer une table d'historique de commandes par client**

```sql
CREATE TABLE orders_by_customer (
    customer_id text,
    order_date timestamp,
    order_id text,
    status text,
    total decimal,
    PRIMARY KEY (customer_id, order_date, order_id)
) WITH CLUSTERING ORDER BY (order_date DESC);
```

**Lire les commandes d'un client**

```sql
SELECT order_date, order_id, status, total
FROM orders_by_customer
WHERE customer_id = 'u42';
```

### Neo4j — Recommandations

**Créer des produits et une relation de compatibilité**

```cypher
CREATE (cpu:Product {sku: 'CPU-7800X3D', name: 'AMD Ryzen 7 7800X3D', type: 'cpu'})
CREATE (mb:Product {sku: 'MB-B650-TUF', name: 'ASUS B650 TUF', type: 'motherboard'})
CREATE (cpu)-[:COMPATIBLE_WITH]->(mb);
```

**Rechercher les produits souvent achetés avec un processeur**

```cypher
MATCH (p:Product {sku: 'CPU-7800X3D'})-[r:BOUGHT_WITH]->(o:Product)
RETURN o.name, r.count
ORDER BY r.count DESC
LIMIT 5;
```

**Rechercher les cartes mères compatibles via le socket**

```cypher
MATCH (cpu:Product {sku: 'CPU-7800X3D'})-[:USES_SOCKET]->(s:Socket)
MATCH (mb:Product {type: 'motherboard'})-[:USES_SOCKET]->(s)
RETURN mb.name, mb.price
ORDER BY mb.price;
```

**Recommandation collaborative**

```cypher
MATCH (p:Product {sku: 'CPU-7800X3D'})<-[:BOUGHT]-(c:Customer)-[:BOUGHT]->(o:Product)
WHERE o <> p
RETURN o.name, count(DISTINCT c) AS customers
ORDER BY customers DESC
LIMIT 5;
```

## ⚙️rincipes architecturaux

- **Persistance polyglotte :** chaque technologie est sélectionnée selon les besoins du domaine.
- **Scalabilité :** possibilité d'évolution vers une architecture distribuée.
- **Dénormalisation :** adaptée aux modèles NoSQL et aux accès métier.
- **Cohérence à terme :** acceptée pour les statistiques et recommandations.
- **Sources de vérité :** chaque domaine possède une base de référence.
- **Sécurité :** accès contrôlés, réseau privé et gestion des secrets.
- **Évolutivité :** architecture pouvant évoluer vers Docker Compose ou Kubernetes.

## Conclusion

Ce projet propose une architecture NoSQL moderne permettant de répondre aux contraintes de performance, de flexibilité et de volumétrie d'une plateforme e-commerce telle que LDLC.
L'utilisation combinée de MongoDB, Redis, Cassandra et Neo4j permet de répondre aux besoins spécifiques de chaque domaine fonctionnel tout en assurant une architecture évolutive et adaptée aux enjeux du commerce électronique.
