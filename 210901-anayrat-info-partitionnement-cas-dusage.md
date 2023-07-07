# Cas d’usages du partitionnement

[blog.anayrat.info](https://blog.anayrat.info/2021/09/01/cas-dusages-du-partitionnement-natif-dans-postgresql/#fn:1)

Les index BRIN présentent des bénéfices proches du partitionnement ou sharding en évitant la complexité de mise en oeuvre.

## Lors de la suppression de données

La suppression massive de données entraîne de la fragmentation dans les tables.

En partitionnant par date. Supprimer les anciennes données revient à supprimer une partition complète. L’opération sera rapide et les tables ne seront pas fragmentées

## contrôler la fragmentation des index

L’ajout et modification de données dans une table fragmente les index au fil du temps: on ne peut pas récupérer l’espace libre dans un bloc tant qu’il n’est pas vide. Avec le temps, les splits d’index créent du “vide” dans ce dernier (bloat).

Pour contrôler le bloat, on peut reconstruire l’index à intervalles réguliers avec `REINDEX CONCURRENTLY`. C'est contraignante, car il faut régulièrement reconstruire l’intégralité de l’index.

Si la majorité des modifications sont faites sur les données récentes, par exemple: table de logs, commandes clients, rendez-vous… On peut faire un partitionnement par mois. Ainsi, à chaque début de mois on crée une table “neuve” et on peut ré-indexer la précédente table pour supprimer le bloat.

On peut en profiter pour faire un `CLUSTER` sur la table : re-organise logiquement les données sur le stockage.

## Faciliter l’exécution de requête lorsque la cardinalité est faible

Une table de commande comprend un statut de livraison. Au bout de quelques années 99% des commandes sont livrées et très peu en cours de paiement ou livraison.

Si on souhaite récupérer 100 commandes en cours de livraison, on crée un index sur le statut (voir même un index partiel sur ce statut particulier). Problème : cet index va se fragmenter assez vite au fur et à mesure que les commandes seront livrées.

Dans ce cas on peut faire un partitionnement sur le statut. Ainsi, récupérer 100 commandes en cours de livraison revient à lire 100 enregistrements de la partition.

## Obtenir de meilleures statistiques

Si je cherche une commande en cours de livraison il y a plus de 6 mois je ne devrais pas avoir de résultat. Inversement, si je cherche des commandes en cours de livraison sur le dernier mois, je devrais obtenir 10% de la table. Or, le moteur ne le sait pas, pour lui les commandes en cours de livraison sont réparties sur toute la table.

Avec un partitionnement par date, le planner peut estimer qu’il n’y a pas de commande en cours de livraisons de plus d’un mois. Ce type d’approche permet surtout de réduire une erreur d’estimation dans un plan d’exécution.

## Stockage avec tiering

Si on veut stocker une partie de la table sur un stockage différent
