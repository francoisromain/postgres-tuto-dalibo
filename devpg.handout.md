# Développer avec Postgres

[devpg.handout.pdf](https://dali.bo/devpg_pdf)

> - J'ai pas tout compris précisément
> - Tout ce qui est indiqué n'a pas d'utilisation directe, mais permet parfois de comprendre la logique de fonctionnement

## Outils

p.48

- [temBoard](https://labs.dalibo.com/temboard) : supervision, tableaux de bord, la gestion des sessions en temps réel, du bloat, de la configuration et l’analyse des performances.
- [pgModeler](https://pgmodeler.io/) : modélisation, la rétroingénierie d’un schéma existant, la génération de scripts de migration.
- [pgAdmin4](https://www.pgadmin.org/) et [DBeaver](https://dbeaver.io/) : GUI

## Commandes SQL

### Transactions

p.80

Groupe des transaction avec `BEGIN` et `COMMIT` pour valider. `ROLLBACK` pour annuler la transaction.

```sql
BEGIN;
  UPDATE comptes SET solde=solde-200 WHERE proprietaire='Bob';
  UPDATE comptes SET solde=solde+200 WHERE proprietaire='Alice';
COMMIT;

BEGIN;
  CREATE TABLE capitaines (id serial, nom text, age integer);
  INSERT INTO capitaines VALUES (1, 'Haddock', 35);

  SAVEPOINT insert_sp;
    UPDATE capitaines SET age = 45 WHERE nom = 'Haddock';
  ROLLBACK TO SAVEPOINT insert_sp;

COMMIT;
```

Exemple : faire `EXPLAIN (ANALYZE)` et annuler la transaction.

### Instance

p.91

Ensemble d'objets globaux : bases de données, rôles et tablespaces. Ils sont disponibles quelque soit la base de données de connexion.

Chaque base de données contient des objets spécifiques accessibles uniquement lorsque l’utilisateur est connecté à cette base.

Les bases sont des conteneurs hermétiques indépendant des objets globaux.

### Tablespaces

p.92

Répertoire physique contenant les fichiers de données de l’instance. Permet de gérer l’espace disque et les performances.

Exemple: stoker les indexes, fichiers de tri temporaires sur un `tablespace` localisé sur un disque SSD.

```sql
CREATE TABLESPACE chaud LOCATION '/SSD/tbl/chaud';
```

### Schémas

p.94

- groupe les objets d’une base de données
- sépare les utilisateurs
- contrôle les accès aux données
- évite les conflits de noms

Espace de noms par défaut : `search_path`.

```sql
CREATE SCHEMA s1;
CREATE SCHEMA s2;

CREATE TABLE t1 (id integer); -- par défaut dans le schéma `public`

SET search_path TO s1;
CREATE TABLE t2 (id integer); -- dans le schémas s1

SET search_path TO s1, public;
CREATE TABLE s2.t2 (id integer); -- dans le schémas s2

SET search_path TO s2, s1, public;

-- ne montre que la première occurrence de la table t2 (ici celle qui est dans s2)
```

Avant la v.15, pour des raisons de sécurité, il est conseillé de laisser le schéma `public` en fin du `search_path`.

### Tables

p. 98

Par défaut, une table est permanente, journalisée et non partitionnée.

`TEMPORARY TABLE` : visibles que par la session qui les a créées et supprimées à la fin de cette session. Utile pour les migrations et imports de données.

`UNLOGGED TABLE` : non journalisée plus performante. Se vide au redémarrage du serveur.

Il est possible de partitionner les tables par intervalle, par valeur ou par hachage.

### Vues

p. 99

```sql
CREATE ROLE guillaume LOGIN;
ALTER TABLE capitaines ADD COLUMN num_cartecredit text;
INSERT INTO capitaines (nom, age, num_cartecredit)
VALUES ('Robert Surcouf', 20, '1234567890123456');

-- création de la vue avec le numéro de CB anonymisé
CREATE VIEW capitaines_anon AS
  SELECT nom, age, substring(num_cartecredit, 0, 10) || '******' AS num_cc_anon
  FROM capitaines;

-- ajout du droit de lecture à l'utilisateur guillaume
GRANT SELECT ON TABLE capitaines_anon TO guillaume;
-- connexion en tant qu'utilisateur guillaume
SET ROLE TO guillaume;

-- it bien la vue
SELECT * FROM capitaines_anon WHERE nom LIKE '%Surcouf';

nom             | age | num_cc_anon
----------------+-----+-----------------
Robert Surcouf  | 20  | 123456789******

-- tentative de lecture directe de la table
SELECT * FROM capitaines;

ERROR: permission denied for relation capitaines
```

On peut ajouter des colonnes à une vue au moment de sa création. Le contenu de ces colonnes est modifiables sans re-générer la vue.

Il n'est pas possible de modifier le contenu des colonnes de la vue qui proviennent de la table originale. `View columns that are not columns of their base relation are not updatable.`

Les vues matérialisées sont des vues dont le contenu est figé sur disque, permettant de ne pas recalculer leur contenu à chaque appel.

Les vues matérialisées ne sont pas mises à jour automatiquement, il faut demander explicitement le rafraîchissement `REFRESH MATERIALIZED VIEW`. Avec la clause `CONCURRENTLY`, s’il y a un index d’unicité, le rafraîchissement ne bloque pas les sessions lisant en même temps les données d’une vue matérialisée.

### Index

p.103

- standard : `Btree`
- texte : `GIN`
- données géo : `GIST`

Le module [pg_trgm](https://) permet l’index de cas habituellement impossibles, comme les expressions rationnelles et `LIKE '%...%'`.

### Type

p.104

```sql
CREATE TYPE serveur AS (
  nom  text,
  adresse_ip inet,
  administrateur text
);

CREATE TYPE jour_semaine
  AS ENUM ('Lundi', 'Mardi', 'Mercredi', 'Jeudi', 'Vendredi', 'Samedi', 'Dimanche');
```

`TYPE xx AS ENUM` vs contrainte `CHECK` :

- l’information ne se trouve pas dans la table, le type est plus difficiles à lister sur une table
- permet une liste ordonnée

Redéfinir le fonctionnement d'un opérateur avec des types customs.

```sql
CREATE OPERATOR + (
  leftarg = stock,
  rightarg = stock,
  procedure = stock_fusion,
  commutator = +
);

--Il faut au préalable avoir défini le type `stock` et la fonction `stock_fusion`.
```

Les `DOMAIN` sont des types créés par les utilisateurs à partir d’un type de base et en lui ajoutant des contraintes supplémentaires.

```sql
CREATE DOMAIN code_postal_francais AS text CHECK (value ~ '^\d{5}$');
```

Om peut modifier un doamine avec `ALTER DOMAIN`. La contrainte sera vérifiée sur les champs de ce type avant qu'il ne soit considérée comme valide.

### Colonnes à valeur générée

p. 108

Valeur calculée à l’insertion.

`DEFAULT` défini une valeur par défaut si absente lors de l'insertion.

`GENERATED ALWAYS AS ( expression ) STORED` calcule une valeur lors de l'insertion.

```sql
ALTER TABLE capitaines
  ADD COLUMN num_cc_anon text
  GENERATED ALWAYS AS (substring(num_cartecredit, 0, 10) || '******') STORED;

SELECT nom, num_cartecredit, num_cc_anon FROM capitaines;

nom             | num_cartecredit  |  num_cc_anon
----------------+------------------+-----------------
Robert Surcouf  | 1234567890123456 | 123456789******

```

Pour créer des ids signifiantes : `GENERATED { ALWAYS | BY DEFAULT } AS IDENTITY` (permet d’obtenir une colonne d’identité, bien meilleure que ce que le pseudo‑type `SERIAL` propose.) ??

### Langages

p. 110

Les trois langages activés par défaut sont `C`, `SQL` et `PL/pgSQL`.
On peut utiliser d'autres langages. il doivent être activé au niveau de la base.

Exemple. js avec [plv8](https://plv8.github.io/)

```SQL
CREATE EXTENSION plv8;
```

```SQL
CREATE FUNCTION plv8_test(keys TEXT[], vals TEXT[]) RETURNS JSON AS $$
    var o = {};
    for(var i=0; i<keys.length; i++){
        o[keys[i]] = vals[i];
    }
    return o;
$$ LANGUAGE plv8 IMMUTABLE STRICT;

SELECT plv8_test(ARRAY['name', 'age'], ARRAY['Tom', '29']);

plv8_test
---------------------------
{"name":"Tom","age":"29"}
(1 row)
```

### Fonctions et procédures

p. 112

Une fonction renvoie une donnée sur une ou plusieurs colonnes. Elle peut renvoyer avoir plusieurs lignes dans le cas d’une fonction `SETOF` ou `TABLE`.

Une procédure ne renvoie rien mais elle peut valider ou annuler la transaction en cours. Dans ce cas, une nouvelle transaction est ouverte immédiatement après la fin de la transaction précédente.

Voir exemple ci-dessous (# Triggers).

### Opérateur

p.112

```sql
-- définissons une fonction de division en PL/pgSQL
CREATE FUNCTION division0 (p1 integer, p2 integer)
RETURNS integer LANGUAGE plpgsql AS $$
  BEGIN
    IF p2 = 0 THEN
      RETURN NULL;
    END IF;
    RETURN p1 / p2;
  END
$$;

-- créons l'opérateur
CREATE OPERATOR // (FUNCTION = division0, LEFTARG = integer, RIGHTARG = integer);

-- une division par 0 ramène une erreur avec l'opérateur natif
SELECT 10/0;
ERROR:
division by zero

-- une division par 0 renvoie NULL avec notre opérateur
SELECT 10//0;
?column?
----------
```

### Triggers

p. 114

Exemple de trigger et procédure

```sql
ALTER TABLE capitaines ADD COLUMN salaire integer;
CREATE FUNCTION verif_salaire() RETURNS trigger AS $verif_salaire$
BEGIN
  -- Nous verifions que les variables ne sont pas vides
  IF NEW.nom IS NULL THEN
    RAISE EXCEPTION 'Le nom ne doit pas être null.';
  END IF;

  IF NEW.salaire IS NULL THEN 100
    RAISE EXCEPTION 'Le salaire ne doit pas être null.';
  END IF;

  -- pas de baisse de salaires !
  IF NEW.salaire < OLD.salaire THEN
    RAISE EXCEPTION 'Pas de baisse de salaire !';
  END IF;

  RETURN NEW;

END;

$verif_salaire$ LANGUAGE plpgsql;

CREATE TRIGGER verif_salaire BEFORE INSERT OR UPDATE ON capitaines
  FOR EACH ROW EXECUTE PROCEDURE verif_salaire();

UPDATE capitaines SET salaire = 2000 WHERE nom = 'Robert Surcouf';
UPDATE capitaines SET salaire = 3000 WHERE nom = 'Robert Surcouf';
UPDATE capitaines SET salaire = 2000 WHERE nom = 'Robert Surcouf';

ERROR: pas de baisse de salaire !
CONTEXTE : PL/pgSQL function verif_salaire() line 13 at RAISE
```

## Plans d'exécution

### Exécution d'une requête

1. Niveau système

   p.119

   ```mermaid
   flowchart TD
     client-- requête  --> markdown["`serveur
     exécute la requête`"]-- résultat --> client
   ```

2. Niveau SGBD : traitement d'une requête SQL

   p.121

   ```mermaid
   flowchart TD
     Requête-- Parser : analyse syntaxique de la requête -->
     Arbre-- Rewriter : ré-écriture : règles, vues non matérialisées, fonctions SQL  -->
     Arbre-étendu-- Planner : génère les plans d exécutions, calcule le coût de chaque plan, choisit le plan le moins coûteux -->
     markdown[Plan d'exécution]-- Executor: récupère les verrous nécessaires sur les objets ciblés puis exécute la requête selon le plan retenu -->
     résultat
   ```

Pour exécuter une requête, le planificateur utilise des opérations :

- accès aux lignes : parcours de table, d’index, de fonction, etc.
- jointures : ordre, type
- agrégation : brut, trié, haché…

Selon l'opération, elle renvoie les résultats :

- soit d’un coup (p.e.: tri). Utilise de la mémoire, et peut nécessiter d’écrire des données temporaires sur disque.
- soit petit à petit (p.e.: parcours séquentiel). Accélère des opérations comme les curseurs, les sous‑requêtes `IN` et `EXISTS`, la clause `LIMIT`, etc. (??)

Le planificateur peut combiner ces opérations de diverses manières pour parvenir au résultat.

### Optimiseur

Une requête décrit le résultat à obtenir, mais pas la façon de l’obtenir.

L’optimiseur :

- énumère tous les plans d’exécution (ou presque puisqu'il élimine les plus coûteux à la volée).
- évalue leur coût d'exécution à partir de statistiques sur les données, de la configuration et de règles inscrites en dur.
- choisit le plan le plus rapide.

Quelques requêtes échappent à cette séquence. Elles sont vérifiées syntaxiquement, puis exécutées, sans réécriture ni planification :

- opérations `DDL` (modification de la structure de la base)
- instructions `TRUNCATE` et `COPY` (en partie)

### Calcul des coûts

p. 125

L’optimiseur de PostgreSQL utilise un modèle de calcul de coût. Les coûts calculés sont des indications arbitraires de la charge nécessaire pour répondre à une requête. Chaque facteur de coût représente une unité de travail : lecture d’un bloc, manipulation d’une ligne en mémoire, application d’un opérateur sur un champ.

Paramètres arbitraires pour ajuster les coûts relatifs. Ne sont pas liés directement à des caractéristiques physiques du serveur.

- `seq_page_cost` (1) : accès séquentiel à un bloc sur le disque
- `random_page_cost` (4) : accès aléatoire (isolé) à un bloc. Ce sera moins avec un bon disque, voire 1 pour un SSD.
- `cpu_tuple_cost` (0,01) : manipulation d’une ligne en mémoire
- `cpu_index_tuple_cost` (0,005) : manipulation d’une donnée issue d’un index
- `cpu_operator_cost` : application d’un opérateur sur une donnée
- `parallel_setup_cost` (1000) indique le coût de mise en place d’un parcours parallélisé (procédure assez lourde qui ne se rentabilise pas pour les petites requêtes)
- `jit_above_cost` (100 000), `jit_inline_above_cost` (500 000), `jit_optimize_above_cost` (500 000) : différents niveaux d’optimisation de compilation à la volée des requêtes (JIT ou Just In Time).

En général, on ne modifie pas ces paramètres, à part `random_page_cost` si le serveur dispose de bons disques.

Autres paramètres

- `enable_seqscan` (on): active le parcours sequentiel (/ force le parcours d'index) (p. 129)
- `track_io_timing` (off) : chronométrage des entrées/sorties disque. (p.142)

Ces paramètres sont des paramètres de sessions. Ils peuvent être modifiés dynamiquement avec l’ordre `SET` au niveau de l’application en vue d’exécuter des requêtes bien particulières.

L’optimiseur fait statistiques sur les données (p.e.: le nombre de blocs et de lignes d’une table, les valeurs les plus fréquentes pour chaque colonne de chaque table). Avec ces statistiques et le paramétrage, il calcule par exemple le ratio d’un filtre et s’il faut passer par un index, ou le ratio d’une jointure pour choisir la stratégie de jointure. Les statistiques sur les données sont calculées lors de l’exécution de la commande SQL `ANALYZE`. L’autovacuum (??) exécute généralement cette opération en arrière‑plan. Des statistiques périmées ou pas assez fines sont une source de plans non optimaux !

### Affichage du plan d'exécution

p. 130

L’optimiseur transforme une requête en plan d'exécution :

- plusieurs opérations unitaires (trier un ensemble de données, lire une table, parcourir un index, joindre deux ensembles de données, etc.) :
  - sous forme arborescente
  - composé des nœuds d’exécution (produit les données consommées par le nœud parent)
- le nœud final retourne les données à l’utilisateur

`EXPLAIN` affiche le plan d'exécution retenu (parmi toute les possibilités) sans exécuter la requête

Options :

- `EXPLAIN (ANALYZE)` : informe sur l’exécution de la requête (et l'exécute). durée pour récupérer la première ligne et toutes les lignes (`actual time`), nombre de ligne (`rows`), nombre d’exécutions du nœud (`loops`). Multiplier `rows` par `loops` pour obtenir la durée d’exécution du nœud
- `ANALYZE, BUFFERS` : nombre de blocs (??) impactés par chaque nœud
- `SETTINGS` (off) : paramètres d’optimisation qui ne sont pas à leur valeur par défaut
- `ANALYZE, WAL` (off) : nombre d’enregistrements et d’octets écrits dans les journaux de transactions
- `COSTS` (on) : coûts
- `TIMING` (on) : chronométrage et des vues/calculées
- `VERBOSE` (off) : verbeux (schémas, colonnes, workers)
- `SUMMARY` (off, on avec `ANALYZE`) : affichage du temps de planification et exécution (si applicable)
- `FORMAT` : format de sortie (texte, XML, JSON, YAML)

Exemple de problèmes :

p.143

- différence entre le nombre de lignes estimé et réel émet un doute sur les statistiques. Soit elles n’ont pas été réactualisées récemment, soit l’échantillon n’est pas suffisant pour que les statistiques donnent une vue du contenu de la table.
- un accès à une ligne par un index est généralement très rapide, mais répété des millions de fois dans une boucle, le total est parfois plus long qu’une lecture complète de la table indexée. C’est l’enjeu du réglage entre `seq_page_cost` et `random_page_cost`.
- L’option `BUFFERS` d’`EXPLAIN` permet de voir les opérations d’entrées/sorties lourdes. Cette option affiche le nombre de blocs lus en et hors du cache de PostgreSQL. Sachant qu’un bloc fait généralement 8 Ko, il est aisé de déterminer le volume de données manipulé par une requête.

### Nœud d'exécution

p.144

Nœud d'execution courants :

- Parcours

  - Table
    - `Seq Scan` : lecture simple de la table, bloc par bloc, ligne par ligne
    - `Parallel Seq Scan` : variante parallélisée
  - Index
    - `Index Scan`, `Bitmap Scan`, `Index Only Scan`
    - et les variantes parallélisées
  - Autres
    - Function Scan, Values Scan

- Jointures

  - Algorithmes
    - Nested Loop
    - Hash Join
    - Merge Join
  - Parallélisation possible
  - Pour EXISTS, IN et certaines jointures externes
    - Semi Join
    - Anti Join

- Agrégats

  - Un résultat au total
    - Aggregate
  - Un résultat par regroupement
    - Hash Aggregate
    - Group Aggregate
    - Mixed Aggregate
  - Parallélisation
    - Partial Aggregate
    - Finalize Aggregate

- Tri

  - Sort
  - Incremental Sort
  - Limit
  - Unique (DISTINCT)
  - Append (UNION ALL), Except, Intersect
  - Gather (parallélisme)
  - Memoize (14+)

### Outils d'analyse graphique

L’analyse de plans complexes devient très vite fastidieuse. Des outils ont été créés pour mieux visualiser les parties intéressantes des plans.

- pgAdmin
- [explain.depesz.com](https://explain.depesz.com/)
- [explain.dalibo.com](https://explain.dalibo.com)

## Optimisations

### Outils de profiling

p.170

- [pgBadger](https://pgbadger.darold.net) : analyseur de log. Dans les journaux applicatifs, trace toutes les requêtes et leur durée, les analyse et retourne les plus fréquemment exécutées, les plus gourmandes unitairement, les plus gourmandes en temps cumulé
- [pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html) : extension standard. Pour chaque ordre exécuté, trace son nombre d’exécution, sa durée cumulée, etc.
- [PoWA](https://powa.readthedocs.io/) : interface graphique sur `pg_stat_statements`

Exemple avec `pg_stat_statements`.

Déterminer les requêtes dont les temps d’exécution cumulés sont les plus importants .

```SQL
SELECT  r.rolname, d.datname, s.calls, s.total_exec_time,
        s.calls / s.total_exec_time AS avg_time, s.query
FROM pg_stat_statements s
JOIN pg_roles r ON (s.userid=r.oid)
JOIN pg_database d ON (s.dbid = d.oid)
ORDER BY s.total_exec_time DESC
LIMIT 10;
```

Déterminer les requêtes les plus fréquemment appelées :

```SQL
SELECT  r.rolname, d.datname, s.calls, s.total_exec_time,
        s.calls / s.total_exec_time AS avg_time, s.query
FROM pg_stat_statements s
JOIN pg_roles r ON (s.userid=r.oid)
JOIN pg_database d ON (s.dbid = d.oid)
ORDER BY s.calls DESC
LIMIT 10;
```

### Éviter

- `DISTINCT` (p. 175)
- `SELECT *`
- vues non‑relationnelles (p. 178)
