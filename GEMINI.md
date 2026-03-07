# Guide de Rédaction d'Histoire - Innherited Hearts

Ce guide explique comment structurer et ajouter du contenu à l'histoire du projet **Innherited Hearts**. Le projet utilise une architecture basée sur des fichiers YAML.

## 1. Structure du Projet

L'histoire est organisée de manière hiérarchique :
- **Story** (`story.yaml`) : Le point d'entrée global. Contient la liste des chapitres et les variables globales.
- **Chapitre** (`chapters/[uuid]/chapter.yaml`) : Définit un chapitre, ses scènes et les connexions entre elles.
- **Scène** (`chapters/[uuid]/scenes/[uuid].yaml`) : Contient les séquences narratives, les dialogues et les conditions locales.
- **Séquence** : Unité de narration au sein d'une scène (arrière-plan, dialogues, choix).

---

## 2. Ajouter un Chapitre

Pour ajouter un nouveau chapitre :
1. Créez un dossier dans `chapters/` nommé avec un nouvel UUID.
2. Créez un fichier `chapter.yaml` à l'intérieur :
   ```yaml
   uuid: "votre-uuid-chapitre"
   name: "Nom du Chapitre"
   subtitle: "Sous-titre optionnel"
   scenes: [] # Liste des scènes (voir section suivante)
   connections: [] # Graphe de navigation entre scènes
   entry_point: "uuid-de-la-premiere-scene"
   ```
3. Déclarez le chapitre dans `story.yaml` sous la section `chapters`.

---

## 3. Ajouter une Scène

Une scène est un fichier YAML situé dans le dossier `scenes/` d'un chapitre.
1. Créez `chapters/[chapter-uuid]/scenes/[scene-uuid].yaml`.
2. Structure de base :
   ```yaml
   uuid: "votre-uuid-scene"
   name: "Nom de la Scène"
   sequences: [] # Liste des séquences
   conditions: [] # Logique de redirection conditionnelle
   connections: [] # Connexions entre séquences locales
   entry_point: "uuid-premiere-sequence-ou-condition"
   ```
3. Ajoutez la scène dans le `chapter.yaml` correspondant et configurez les `connections`.

---

## 4. Ajouter une Séquence

Les séquences portent le contenu narratif. Elles se trouvent dans le fichier de la scène.

### Structure d'une séquence :
```yaml
- uuid: "uuid-sequence"
  name: "Description interne"
  background: "assets/backgrounds/image.png"
  dialogues:
    - character: "Nom du Personnage"
      text: "Le texte du dialogue."
  ending:
    type: "auto_redirect" # Ou "choices"
    consequence:
      type: "redirect_scene" # Ou "redirect_sequence"
      target: "uuid-cible"
```

### Séquence avec Choix :
```yaml
ending:
  type: "choices"
  choices:
    - text: "Option A"
      consequence:
        type: "redirect_sequence"
        target: "uuid-cible-A"
      effects:
        - variable: "ma_variable"
          operation: "set"
          value: "1"
```

---

## 5. Écrire des Conditions

Les conditions permettent de rediriger le joueur en fonction de variables. Elles sont placées dans le bloc `conditions` d'une scène.

```yaml
conditions:
  - uuid: "uuid-condition"
    name: "Check variable"
    rules:
      - variable: "ma_variable"
        operator: "equals" # exists, not_exists, equals, greater_than, lower_than
        value: "1"
        consequence:
          type: "redirect_sequence"
          target: "uuid-si-vrai"
    default_consequence:
      type: "redirect_sequence"
      target: "uuid-par-defaut"
```

### Contraintes importantes du moteur sur les conditions :

1. **Les conditions ne sont accessibles QUE via `entry_point`**. On ne peut JAMAIS cibler une condition avec `redirect_sequence` ou `redirect_scene`. Toute tentative provoquera une erreur "cible introuvable".
2. **Les conditions ne peuvent rediriger que vers des séquences** de la même scène via `redirect_sequence`, ou vers d'autres scènes via `redirect_scene`.
3. **`redirect_scene` vers la même scène (auto-redirection) ne fonctionne PAS**. Le moteur ne ré-évalue pas le `entry_point` correctement dans ce cas.

---

## 6. Variables et Effets

- **Définition** : Les variables doivent être déclarées dans `story.yaml` avec une `initial_value`.
- **Modification** : Utilisez le bloc `effects` dans une `consequence` ou un `choice`.
  - `operation: "set"`
  - `operation: "add"` (pour les scores/affinités)
  - `operation: "increment"` (incrémente la valeur de la variable)

---

## 7. Internationalisation (i18n)

Le projet utilise `i18n/`. Chaque chaîne de texte utilisée comme `text` dans les dialogues ou les choix sert de **clé** dans les fichiers de traduction.

**Règle d'or** : Chaque nouveau texte ajouté dans un fichier YAML de scène doit être ajouté dans `i18n/fr.yaml` (et autres langues) :
```yaml
"Le texte de votre dialogue": "Le texte de votre dialogue"
```

---

## 8. Pattern "Visiter tout" (Hub)

Quand le joueur doit visiter plusieurs lieux dans un ordre libre (ex : inspecter des chambres, interroger des personnages), il faut utiliser le **pattern Hub** :

### Principe :
- Une **scène hub** centralise la logique (condition + choix).
- Chaque lieu à visiter est un **fichier scène séparé** (pas une séquence dans le hub).
- Les scènes visitées redirigent vers le hub via `redirect_scene`, ce qui relance le `entry_point` (condition).

### Structure du hub :
```yaml
# Scène hub
entry_point: "uuid-condition"

conditions:
  - uuid: "uuid-condition"
    rules:
      - variable: "visited_lieu_A"
        operator: "not_exists"
        value: ""
        consequence:
          type: "redirect_sequence"
          target: "uuid-sequence-choix"
      - variable: "visited_lieu_B"
        operator: "not_exists"
        value: ""
        consequence:
          type: "redirect_sequence"
          target: "uuid-sequence-choix"
    default_consequence:
      type: "redirect_scene"  # Tous visités → scène suivante
      target: "uuid-scene-suivante"

sequences:
  - uuid: "uuid-sequence-choix"
    ending:
      type: "choices"
      choices:
        - text: "Visiter lieu A"
          consequence:
            type: "redirect_scene"
            target: "uuid-scene-lieu-A"  # Fichier scène séparé !
          effects:
            - variable: "visited_lieu_A"
              operation: "set"
              value: "1"
```

### Structure d'une scène lieu :
```yaml
# Fichier scène séparé pour chaque lieu
ending:
  type: "auto_redirect"
  consequence:
    type: "redirect_scene"
    target: "uuid-scene-hub"  # Retour au hub → entry_point → condition
```

### Dans chapter.yaml :
- Ajouter chaque scène lieu dans `scenes`
- Ajouter les `connections` : hub → chaque lieu, chaque lieu → hub (ou scène suivante)

### Exemples dans le projet :
- **Chapitre 1** : `pens1on1` (rencontrer les pensionnaires) → scènes `mary0001`, `lise0001`, etc.
- **Chapitre 3** : `ch3s0005` (inspection des chambres) → scènes `ch3r-mary`, `ch3r-lise`, etc.
- **Chapitre 3** : `ch3s0006` (interrogatoires) → scènes `ch3i-mary`, `ch3i-lise`, etc.

---

## 9. Conseils pour le LLM

- Générez toujours des **UUID uniques** pour chaque nouvel élément.
- Maintenez la cohérence des `connections` pour éviter les culs-de-sac narratifs.
- Vérifiez que les chemins d'assets (`assets/backgrounds/...`) existent dans le projet.
- N'oubliez pas de mettre à jour les fichiers `i18n` après avoir ajouté du texte.
- **Ne jamais cibler une condition avec `redirect_sequence` ou `redirect_scene`** — les conditions ne sont accessibles que via `entry_point`.
- **Ne jamais utiliser `redirect_scene` vers la même scène** — l'auto-redirection ne fonctionne pas.
- Pour les patterns "visiter tout", toujours créer des **fichiers scènes séparés** pour chaque lieu.
