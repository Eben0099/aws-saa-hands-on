# Notes examen : Projet 01

## Ordre d'évaluation des policies

- Règle : **explicit deny > allow > implicit deny**.
- Il n'y a pas d'ordre chronologique : AWS évalue toutes les policies qui s'appliquent ensemble.
- L'implicit deny n'est pas une policy écrite. C'est ce qui reste quand aucune policy ne correspond. Message : "no identity-based policy allows...".
- Le Deny explicite gagne même si un Allow existe pour la même action et la même ressource.
- Pourquoi : least privilege et secure by default. Un oubli donne un refus visible, pas un accès en trop. Un garde-fou (Deny) ne peut pas être contourné par un Allow ajouté plus tard.

## S3 : deux niveaux de ressource

- `s3:ListBucket` : ARN du bucket, `arn:aws:s3:::nom-du-bucket`
- `s3:GetObject` : ARN des objets, `arn:aws:s3:::nom-du-bucket/*`
- `s3:ListAllMyBuckets` : liste tous les buckets du compte. C'est une action différente de `ListBucket`. `aws s3 ls` sans nom de bucket en a besoin.

## Rôle vs access keys

- Rôle sur EC2 : instance profile, rôle, STS, credentials temporaires renouvelés automatiquement. Rien n'est stocké sur disque.
- Access keys : credentials permanents, valides jusqu'à révocation manuelle.
- Le message d'erreur `assumed-role/NomDuRole/i-...` confirme que l'instance utilise le rôle.
- Trust policy du rôle EC2 : principal `ec2.amazonaws.com`, action `sts:AssumeRole`.

## Root

- Le root a des pouvoirs qu'aucune policy IAM ne peut limiter, pas même un Deny explicite. Seules les SCP d'une Organization le peuvent.
- Root réservé aux opérations qui l'exigent, par exemple activer l'accès IAM à la facturation.

## Groupes IAM

- Les permissions d'un rôle métier vont sur un groupe, pas sur chaque utilisateur.

## Copie S3 vers S3

- `aws s3 cp` a trois combinaisons valides : local vers S3, S3 vers local, S3 vers S3.
- Une copie S3 vers S3 exige `s3:GetObject` sur la source **et** `s3:PutObject` sur la destination.

## EC2 Instance Connect

- Pas de clé privée à gérer. Une clé temporaire (environ 60 secondes) est poussée, et la connexion passe par IAM.

## À compléter

<!-- TODO: à compléter par Ebene - identity-based vs resource-based -->
