# Indexation et SQL avancé

## Index

### Index BRIN (Block Range INdex)

p. 102

Un index BRIN ne stocke pas les valeurs, mais quelles plages de valeurs se rencontrent dans un ensemble de blocs. Cela réduit la taille de l’index et permet d’exclure un ensemble de blocs lors d’une recherche.

Pour qu’un index BRIN soit utile, il faut :

- que les données soit triées dans la table, et le restent (série temporelle, décisionnel avec imports réguliers…) ;
- que la table soit à « insertion seule » pour conserver l’ordre physique ;
- ou que l’on ait la disponibilité nécessaire pour reconstruire régulièrement la table avec `CLUSTER` et éviter que les performances se dégradent au fil du temps.

Sous ces conditions, les BRIN sont indiqués si l’on a des problèmes de volumétrie, ou de temps d’écritures dus aux index B‑tree, ou pour éviter de partitionner une grosse table dont les requêtes ramènent une grande proportion.

## Partitionnement

p. 187

Intérêts :

- Facilite la maintenance de gros volumes
  - VACUUM (FULL), réindexation, déplacements, sauvegarde logique…
- Améliore les performances
  - parcours complet sur de plus petites tables
  - statistiques par partition plus précises
  - purge par partitions entières
  - pg_dump parallélisable
  - tablespaces différents (données froides/chaudes)
- Attention à la maintenance sur le code


partition sur l'état

état administratif
- actif
- inactif
- tout