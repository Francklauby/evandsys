---
name: dolibarr-module
description: Créer un module Dolibarr (squelette complet) ou ajouter
  un hook / trigger sur le core. Utiliser quand on demande de créer
  un module, une page Dolibarr, un hook, un trigger, ou d'étendre
  le comportement du core sans le modifier.
---

## Environnement
Base de données : MySQL / MariaDB.
Préfixe des tables : llx_ (toujours respecté, jamais de table sans ce préfixe).
Encodage : UTF-8.

## Structure d'un module
htdocs/custom/nom_module/
admin/
setup.php
class/
nom_module.class.php
actions_nom_module.class.php
core/triggers/
interface_99_modNomModule_NomModule.class.php
langs/
fr_FR/nom_module.lang
en_US/nom_module.lang
sql/
llx_nom_module.sql
llx_nom_module.key.sql
lib/
nom_module.lib.php
nom_module.class.php

## Conventions de code Dolibarr
Chaque fichier PHP doit commencer par :
```php
<?php
if (!defined('NOTOKENRENEWAL') && !defined('NOREQUIREMENU')) {
    require '../main.inc.php';
}
```

Objets globaux : $db, $user, $langs, $conf

Requêtes SQL :
- Toujours via $db->query() et $db->escape()
- Jamais de concaténation directe de variables dans le SQL
- Transactions : $db->begin() / $db->commit() / $db->rollback()

Traductions :
- $langs->load('nom_module@nom_module') en début de page
- $langs->trans('CleTraduction') dans les templates

Droits :
Utiliser $user->hasRight() — méthode moderne (Dolibarr 13+) :
```php
// Affichage
if (!$user->hasRight('nom_module', 'lire')) {
    accessforbidden();
}

// Écriture
if (!$user->hasRight('nom_module', 'creer')) {
    accessforbidden();
}

// Suppression
if (!$user->hasRight('nom_module', 'supprimer')) {
    accessforbidden();
}
```
$user->rights->module->perms est deprecated depuis Dolibarr 13+.
Ne plus utiliser dans les nouveaux modules.

## Créer un hook
```php
class ActionsNomModule {
    public function doActions($parameters, &$object, &$action, $hookmanager) {
        return 0;
    }
}
```
Contextes courants : invoicecard, ordercard, thirdpartycard, productcard, mainmenu

## Créer un trigger
```php
class InterfaceNomModule extends DolibarrTriggers {
    public function runTrigger($action, $object, $user, $langs, $conf) {
        if ($action === 'BILL_VALIDATE') {
            // logique
        }
        return 0;
    }
}
```
Événements courants : BILL_VALIDATE, ORDER_VALIDATE, COMPANY_CREATE, PAYMENT_ADD

## Scripts SQL
```sql
CREATE TABLE IF NOT EXISTS llx_nom_module (
  rowid  INT(11) NOT NULL AUTO_INCREMENT,
  entity INT(11) DEFAULT 1 NOT NULL,
  fk_soc INT(11),
  label  VARCHAR(255),
  datec  DATETIME,
  tms    TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (rowid)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Ce qu'il faut produire
- Pour un nouveau module : tous les fichiers du squelette
- Pour un hook : fichier actions_*.class.php + modification du descripteur
- Pour un trigger : fichier interface_*.class.php complet
- Toujours indiquer le chemin exact de chaque fichier produit
