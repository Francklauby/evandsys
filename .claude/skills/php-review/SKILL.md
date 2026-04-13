---
name: php-review
description: Revue de code PHP selon les bonnes pratiques.
  Utiliser quand on demande une revue, un audit, "est-ce que
  ce code est bon ?", "que penses-tu de ce code ?"
---

## Checklist de revue

### Lisibilité & nommage
- Noms de variables, fonctions et classes explicites (pas $a, $tmp, $data)
- Fonctions courtes avec une seule responsabilité
- Pas de commentaires redondants (le code doit s'expliquer seul)
- Commentaires présents uniquement pour les "pourquoi", pas les "quoi"

### Qualité & robustesse
- Type hints sur les paramètres et valeurs de retour
- Gestion explicite des erreurs (pas d'@ pour masquer les warnings)
- Pas de valeurs magiques : utiliser des constantes nommées
- Validation des entrées utilisateur avant tout traitement
- Pas de code mort ou de blocs commentés inutiles

### Sécurité
- Échappement des sorties HTML (htmlspecialchars ou équivalent)
- Requêtes SQL via prepared statements (jamais de concaténation directe)
- Pas de données sensibles en clair (mots de passe, tokens, clés API)
- Vérification des permissions avant les actions sensibles

### Performance
- Pas de requêtes en boucle
- Utilisation de structures de données adaptées (tableau vs objet)
- Pas de calculs répétitifs identiques (mettre en cache / variable)

### Maintenabilité
- Pas de duplication de logique (DRY)
- Dépendances injectées plutôt que créées en dur (new dans les méthodes)
- Couplage faible entre les classes

## Format de sortie
Pour chaque problème trouvé :
1. Niveau : [BLOQUANT / IMPORTANT / MINEUR]
2. Localisation : fichier + ligne si possible
3. Problème constaté
4. Suggestion concrète (avec exemple de code si utile)

Terminer par un résumé : nb de points par niveau + appréciation globale.
