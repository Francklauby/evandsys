---
name: php-doc
description: Générer de la documentation PHP : PHPDoc inline et
  fichiers Markdown. Utiliser quand on demande de documenter,
  "ajoute des commentaires", "génère un README", "documente
  cette classe / cette API / ce endpoint".
---

## Règle générale
Documenter le POURQUOI et les comportements non évidents.
Ne pas paraphraser le code : un commentaire qui dit
"incrémente le compteur" sous $counter++ est inutile.

## PHPDoc — classes & interfaces
```php
/**
 * Gère l'envoi de notifications aux utilisateurs.
 *
 * Supporte plusieurs canaux (email, SMS) selon les préférences
 * de l'utilisateur. Les envois sont mis en file d'attente
 * et traités de manière asynchrone.
 */
class NotificationService { ... }
```
- Une phrase résumé, puis contexte si nécessaire
- Mentionner les effets de bord ou contraintes importantes
- @throws si des exceptions sont levées intentionnellement

## PHPDoc — méthodes & fonctions
```php
/**
 * Envoie une notification via le canal préféré de l'utilisateur.
 *
 * Si le canal principal est indisponible, bascule automatiquement
 * sur le canal de secours défini dans la config.
 *
 * @param User   $user    Destinataire de la notification
 * @param string $message Contenu brut (pas de HTML)
 * @return bool  true si envoyé, false si mis en file d'attente
 * @throws InvalidUserException Si l'utilisateur n'a aucun canal configuré
 */
public function send(User $user, string $message): bool { ... }
```
- @param : type + nom + description courte (uniquement si non évident)
- @return : type + ce que la valeur signifie concrètement
- @throws : uniquement les exceptions intentionnelles (pas les RuntimeException génériques)
- Omettre @param/@return si les type hints sont suffisamment explicites

## PHPDoc — propriétés
- Documenter uniquement si le nom ou le type ne suffit pas
- Préférer un type hint précis à un long commentaire

## Markdown — README de module/composant
Structure à suivre :
```
# Nom du composant
Phrase résumé : ce que c'est et à quoi ça sert.

## Utilisation rapide
Exemple de code minimal fonctionnel.

## Paramètres / Configuration
Tableau : nom | type | défaut | description

## Comportements notables
Points non évidents, limitations connues, cas limites.

## Exemples
Cas d'usage concrets (nominal + cas limite si utile).
```

## Markdown — documentation d'API / endpoint
Pour chaque endpoint :
```
### POST /users

Crée un nouvel utilisateur.

**Corps de la requête**
| Champ    | Type   | Requis | Description          |
|----------|--------|--------|----------------------|
| email    | string | oui    | Email unique         |
| password | string | oui    | Min. 8 caractères    |

**Réponses**
- 201 : utilisateur créé, retourne l'objet User
- 422 : données invalides, retourne les erreurs de validation
- 409 : email déjà utilisé
```

## Ce qu'il faut produire
- Pour PHPDoc : le fichier source modifié avec les blocs ajoutés
- Pour Markdown : le fichier .md complet, prêt à enregistrer
- Signaler si une partie du code est trop obscure pour être
  documentée sans clarification préalable
