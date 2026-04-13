# Contexte projet

Projet Dolibarr — développement solo.
Version Dolibarr 23.0
objectif : Fournir un dolibarr en mode SaaS à des clients

## Modules personnalisés
- entitydomain → htdocs/custom/entitydomain/
- facturex → htdocs/custom/facturex/

## Organisation des dépôts Git
IMPORTANT : plusieurs dépôts Git distincts dans ce projet.

Dépôt core : fork de github.com/Dolibarr/dolibarr
- Commits directs sur main
- Ne jamais y mettre de logique métier perso
- commit directs sur branch evandsys

Dépôts modules : un dépôt par module dans htdocs/custom/nom_module/
- Commits directs sur main
- Vérifier dans quel dépôt on se trouve avant chaque commit

Format de commit : type: description courte
ex: feat: ajout export CSV commandes
fix: correction calcul TVA avoir
sql: ajout colonne date_echeance llx_monmodule

## Règles absolues
- Eviter le plus possible de modifier htdocs/ hors de htdocs/custom/
- Toute extension du core passe priotéirement par hooks, triggers ou surcharge custom/
- si une modification hors htdocs/custom s'impose la faire précéder par "/* mod_evandsys */" et finir par "/* fin_mod_evandsys */" + ajouter une ligne de commentaire pour expliquer succintement la raison de la modif 
- Scripts SQL toujours idempotents (IF NOT EXISTS / IF EXISTS)

## Skills disponibles
- /php-review       → revue de code PHP
- /php-doc          → génération PHPDoc + Markdown
- /dolibarr-module  → créer un module ou un hook/trigger Dolibarr
