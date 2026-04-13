# SGBD 

---

## I. Qu'est-ce que UML et à quoi servent ses diagrammes principaux ?

**Question** : "Qu'est-ce que UML et quels sont les diagrammes que vous utilisez le plus concrètement pour modéliser un système logiciel ?"

**Réponse** :

**UML (Unified Modeling Language)** est un langage de modélisation standardisé (ISO) qui permet de représenter visuellement la structure et le comportement d'un système logiciel. Il ne prescrit pas une méthode de développement — c'est un outil de communication entre parties prenantes techniques et non techniques.

UML distingue deux grandes familles de diagrammes.

**Diagrammes structurels** (ce que le système *est*) :

**Diagramme de classes** : le plus utilisé en conception objet. Représente les classes, leurs attributs, leurs méthodes, et les relations entre elles. Les relations clés à maîtriser :
- **Association** : lien entre deux classes (`Commande` — `Client`)
- **Agrégation** : relation "a un" faible — la partie peut exister sans le tout (`Equipe` ◇— `Joueur`)
- **Composition** : relation "a un" forte — la partie n'existe pas sans le tout (`Maison` ◆— `Piece`)
- **Héritage** : relation "est un" (`Admin` ——▷ `Utilisateur`)
- **Dépendance** : utilisation temporaire (`Service` ---> `Logger`)

```
+------------------+          +-------------------+
|   Utilisateur    |          |     Commande       |
|------------------|  1    *  |-------------------|
| - id: int        |----------| - id: int          |
| - email: str     |          | - date: datetime   |
| + login()        |          | - total: float     |
+------------------+          | + calculer_total() |
         △                    +-------------------+
         |                             |  composition
+------------------+                  |
|     Admin        |         +-------------------+
|------------------|         |   LigneCommande    |
| + gerer_users()  |         |-------------------|
+------------------+         | - quantite: int    |
                             | - prix_unitaire    |
                             +-------------------+
```

**Diagrammes comportementaux** (ce que le système *fait*) :

**Diagramme de séquence** : montre les interactions entre objets dans le temps, message par message. Indispensable pour documenter les flux d'une API ou d'un cas d'usage complexe.

**Diagramme de cas d'utilisation (Use Case)** : représente les acteurs (utilisateurs, systèmes externes) et les fonctionnalités qu'ils peuvent déclencher. Utilisé en phase d'analyse pour aligner les équipes sur le périmètre fonctionnel.

**Diagramme d'activité** : équivalent UML d'un flowchart, utile pour modéliser les processus métier ou les algorithmes complexes.

---

## II. Qu'est-ce que la normalisation et pourquoi normaliser une base de données ?

**Question** : "Pourquoi normalise-t-on une base de données relationnelle ? Quels problèmes cherche-t-on à éviter ?"

**Réponse** :

La **normalisation** est un processus de structuration des tables d'une base de données relationnelle pour éliminer la redondance des données et prévenir les anomalies de mise à jour.

**Les trois anomalies qu'on cherche à éviter** :

**Anomalie d'insertion** : on ne peut pas insérer une donnée sans en insérer une autre qui n'existe pas encore. Exemple : on ne peut pas enregistrer un cours sans avoir d'étudiant inscrit si le cours et l'étudiant sont dans la même table.

**Anomalie de modification** : une information stockée en plusieurs endroits oblige à la modifier partout — oublier un endroit crée une incohérence silencieuse.

**Anomalie de suppression** : supprimer une ligne efface accidentellement des informations qu'on voulait conserver. Exemple : supprimer le dernier étudiant d'un cours efface l'information sur le cours lui-même.

**Le principe** : chaque fait doit être stocké une seule fois, à un seul endroit, dans la table qui lui correspond sémantiquement.

---

## III. Qu'est-ce que la 1NF, 2NF et 3NF ?

**Question** : "Expliquez les trois premières formes normales avec un exemple concret de table non normalisée et les étapes pour l'amener en 3NF."

**Réponse** :

**Table de départ — non normalisée** :

| commande_id | client_nom | client_ville | produits | categories |
|---|---|---|---|---|
| 1 | Dupont | Paris | Livre, Stylo | Papeterie, Papeterie |
| 2 | Martin | Lyon | Ordinateur | Informatique |

**1NF — Première Forme Normale** : chaque cellule contient une valeur atomique (indivisible). Pas de listes, pas de groupes répétitifs dans une colonne. Chaque ligne est identifiable par une clé primaire.

Violation : la colonne `produits` contient "Livre, Stylo" — valeur non atomique.

```
Après 1NF :
| commande_id | client_nom | client_ville | produit    | categorie   |
|-------------|------------|--------------|------------|-------------|
| 1           | Dupont     | Paris        | Livre      | Papeterie   |
| 1           | Dupont     | Paris        | Stylo      | Papeterie   |
| 2           | Martin     | Lyon         | Ordinateur | Informatique|

Clé primaire composée : (commande_id, produit)
```

**2NF — Deuxième Forme Normale** : être en 1NF + tous les attributs non-clés dépendent de **toute** la clé primaire (pas d'une partie seulement). Élimine les dépendances partielles.

Violation : `client_nom` et `client_ville` dépendent uniquement de `commande_id`, pas du couple `(commande_id, produit)`.

```
Après 2NF — on sépare en deux tables :

Table Commandes :          Table LignesCommande :
| commande_id | client_nom | client_ville |    | commande_id | produit    | categorie    |
|-------------|------------|--------------|    |-------------|------------|--------------|
| 1           | Dupont     | Paris        |    | 1           | Livre      | Papeterie    |
| 2           | Martin     | Lyon         |    | 1           | Stylo      | Papeterie    |
                                              | 2           | Ordinateur | Informatique |
```

**3NF — Troisième Forme Normale** : être en 2NF + aucun attribut non-clé ne dépend d'un autre attribut non-clé (pas de dépendance transitive).

Violation : `categorie` dépend de `produit`, pas directement de la clé `(commande_id, produit)`.

```
Après 3NF — on isole les dépendances transitives :

Table LignesCommande :     Table Produits :
| commande_id | produit_id |    | produit_id | nom         | categorie    |
|-------------|------------|    |------------|-------------|--------------|
| 1           | 101        |    | 101        | Livre       | Papeterie    |
| 1           | 102        |    | 102        | Stylo       | Papeterie    |
| 2           | 103        |    | 103        | Ordinateur  | Informatique |
```

**BCNF (Boyce-Codd)** : forme plus stricte que la 3NF — toute dépendance fonctionnelle doit avoir une super-clé en déterminant. Utile quand plusieurs clés candidates se chevauchent.

---

## IV. Qu'est-ce que les propriétés ACID ?

**Question** : "Qu'est-ce que les propriétés ACID d'une transaction ? Donnez un exemple concret qui illustre pourquoi chaque propriété est indispensable."

**Réponse** :

**ACID** est l'ensemble des propriétés qui garantissent la fiabilité des transactions dans un SGBD relationnel. Une transaction est une séquence d'opérations traitée comme une unité indivisible.

**A — Atomicité** : une transaction est tout ou rien. Si une étape échoue, toutes les opérations précédentes de la transaction sont annulées (rollback). Exemple : un virement bancaire décrédite le compte A et crédite le compte B. Si le crédit échoue après le débit, l'atomicité garantit que le débit est annulé — l'argent ne disparaît pas.

```sql
BEGIN;
  UPDATE comptes SET solde = solde - 500 WHERE id = 1;  -- débit
  UPDATE comptes SET solde = solde + 500 WHERE id = 2;  -- crédit
COMMIT;  -- les deux ou aucun
-- Si erreur entre les deux : ROLLBACK automatique
```

**C — Cohérence** : une transaction fait passer la base d'un état valide à un autre état valide. Toutes les contraintes d'intégrité (clés étrangères, contraintes CHECK, unicité) doivent être respectées avant et après la transaction. Si une contrainte est violée, la transaction est annulée.

**I — Isolation** : les transactions concurrentes s'exécutent comme si elles étaient séquentielles. Une transaction en cours ne voit pas les modifications non encore commitées par une autre transaction. Les niveaux d'isolation (du plus permissif au plus strict) :

| Niveau | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| Read Uncommitted | Possible | Possible | Possible |
| Read Committed | Impossible | Possible | Possible |
| Repeatable Read | Impossible | Impossible | Possible |
| Serializable | Impossible | Impossible | Impossible |

En production, PostgreSQL utilise **Read Committed** par défaut — bon équilibre entre cohérence et performance.

**D — Durabilité** : une fois qu'une transaction est commitée, ses modifications sont permanentes même en cas de crash système. Garanti par le **WAL (Write-Ahead Log)** : chaque modification est d'abord écrite dans un journal persistant avant d'être appliquée aux données. En cas de crash, le SGBD rejoue le WAL pour retrouver l'état cohérent.

---

## V. Comment les index améliorent-ils les performances ?

**Question** : "Qu'est-ce qu'un index en base de données, comment fonctionne-t-il internement, et quand ne faut-il pas en créer ?"

**Réponse** :

Un **index** est une structure de données auxiliaire qui permet de localiser rapidement des lignes sans parcourir toute la table (full table scan).

**Structure interne — B+Tree** : la plupart des SGBD (PostgreSQL, MySQL) utilisent un arbre B+ comme structure d'index. Les nœuds internes contiennent des clés de tri, les feuilles contiennent les clés + pointeurs vers les enregistrements. Toutes les feuilles sont reliées en liste chaînée pour les scans de plage (`BETWEEN`, `ORDER BY`). Complexité : O(log n) pour la recherche.

```
Index B+Tree sur la colonne "email" :

             [M]
           /     \
        [D, J]   [R, Z]
       /  |  \   /  \
    [A-D][E-J][K-M][N-R][S-Z]
      |    |    |    |    |
   rows  rows rows rows rows  <-- feuilles liées entre elles
```

**Types d'index courants** :
- **B-Tree** : recherche par égalité et plage (`=`, `<`, `>`, `BETWEEN`, `LIKE 'abc%'`)
- **Hash** : uniquement pour l'égalité exacte (`=`), plus rapide en lecture
- **GIN** : pour les recherches full-text et les colonnes JSONB
- **Composite** : index sur plusieurs colonnes — l'ordre des colonnes est crucial (la colonne la plus sélective en premier)

**Coûts des index** : chaque index doit être mis à jour à chaque INSERT, UPDATE, DELETE. Un excès d'index ralentit les écritures et augmente l'espace disque.

**Quand ne pas créer d'index** :
- Tables de petite taille (un full scan est plus rapide)
- Colonnes avec très peu de valeurs distinctes (faible cardinalité — ex: colonne booléenne)
- Tables qui subissent beaucoup plus d'écritures que de lectures
- Colonnes rarement utilisées dans les clauses `WHERE` ou `JOIN`

```sql
-- Voir si un index est utilisé
EXPLAIN ANALYZE SELECT * FROM commandes WHERE client_id = 42;

-- Créer un index composite (ordre important)
CREATE INDEX idx_commandes_client_date
ON commandes(client_id, created_at DESC);
```

---

## VI. Quelle différence entre SQL et NoSQL ?

**Question** : "Comment choisissez-vous entre une base de données SQL et NoSQL pour un projet ? Quelles sont les garanties sacrifiées avec le NoSQL ?"

**Réponse** :

**SQL (relationnel)** : données structurées en tables avec un schéma fixe, relations via clés étrangères, transactions ACID complètes. Idéal quand les données ont des relations complexes et que la cohérence forte est requise (finances, e-commerce, ERP).

**NoSQL** : famille de bases de données qui sacrifient certaines garanties ACID pour gagner en scalabilité, flexibilité du schéma, ou performance à grande échelle. Il existe quatre grandes familles.

**Document (MongoDB, Firestore)** : données stockées en documents JSON/BSON. Schéma flexible — chaque document peut avoir des champs différents. Idéal pour les catalogues de produits, les profils utilisateurs, les données hétérogènes.

**Clé-Valeur (Redis, DynamoDB)** : accès par clé unique, latence sub-milliseconde. Idéal pour le cache, les sessions, les compteurs en temps réel.

**Colonne (Cassandra, HBase)** : optimisé pour les écritures massives et les lectures par plage de temps. Idéal pour les séries temporelles, les logs, l'IoT.

**Graphe (Neo4j)** : nœuds et relations comme structure native. Idéal pour les réseaux sociaux, les systèmes de recommandation, la détection de fraude.

**Le théorème CAP** : un système distribué ne peut garantir simultanément que deux des trois propriétés suivantes.

| Propriété | Signification |
|---|---|
| **C**onsistency | Tous les nœuds voient les mêmes données au même moment |
| **A**vailability | Chaque requête reçoit une réponse (sans garantie d'être à jour) |
| **P**artition tolerance | Le système continue de fonctionner malgré des pannes réseau |

Les bases SQL privilégient **CP** (cohérence + tolérance aux partitions). Les bases NoSQL comme Cassandra ou DynamoDB privilégient **AP** (disponibilité + tolérance aux partitions) avec une cohérence éventuelle — les données finissent par converger, mais pas instantanément.

**Modèle hybride** : en production, on combine souvent les deux — PostgreSQL pour les données transactionnelles, Redis pour le cache, DynamoDB pour les sessions ou les événements.
