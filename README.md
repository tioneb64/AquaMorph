# AquaMorph

Simulation évolutionnaire 2D de créatures marines (« critters »), écrite en C++20.

Des créatures aléatoires, faites d'articulations reliées par des membres, évoluent sur plusieurs générations pour apprendre à nager d'un point A à un point B dans un environnement aquatique simple : traînée de l'eau, gravité et flottabilité.

> **Statut : version 1.0.0 en cours de développement.** Les spécifications sont rédigées (dossier `docs/`) ; l'implémentation suit les jalons du plan de développement (M0 à M8).

## Principe

- **Morphologie** : un critter est un arbre de 3 à 20 joints reliés par des arms de même longueur (1 à 5 arms par joint, branches d'au plus 5 joints).
- **Génome** : il encode la topologie, les masses, les plages d'angle et de résistance, les déphasages et un vecteur de modulation. Il est sérialisable en JSON.
- **Population** : 250 individus par génération, évalués sur K directions de B.
- **Sélection** : les 20 meilleurs deviennent parents (50 couples, 5 enfants par couple), avec croisement gène par gène et mutations rares.
- **Apprentissage** : un réseau moteur partagé par toute la population (MLP écrit à la main, REINFORCE avec baseline) apprend à nager. Il s'améliore par gradient, jamais par mutation.
- **Physique** : masses ponctuelles aux joints, Euler semi-implicite, contraintes de distance (PBD), traînée quadratique sur la vitesse normale de chaque arm.

## Architecture

Deux processus communiquent par ZeroMQ (messages MessagePack) :

- **`aquamorph-ocean`** : la simulation. Elle fonctionne sans graphique (`--headless`) et sert d'horloge maîtresse. Un rendu SFML optionnel (`--render`) permet d'observer.
- **`aquamorph-agent`** : la politique (réseau de neurones, ou contrôleurs de test `sinus`, `still`, `random`) et l'apprentissage.

En entraînement, l'échange est déterministe (mode lockstep). Le mode temps réel sert uniquement à l'affichage.

### Modules

| Module | Rôle | Dépend de |
|---|---|---|
| `core` | Génome, physique, énergie, fitness, sélection, boucle de génération | nlohmann/json |
| `common` | Enveloppe des messages, sérialisation, transport ZeroMQ | nlohmann/json, ZeroMQ |
| `ocean` | Application océan : poignée de main, génération, modes lockstep et temps réel | core, common |
| `agent` | Réseau, politiques, REINFORCE, checkpoints, métriques d'apprentissage | common, Eigen |
| `render` | Fenêtre SFML, caméra, dessin des critters (lecture seule) | SFML, core |

Le moteur (`core`) ne dépend ni de SFML, ni de ZeroMQ, ni d'Eigen : il se teste sans réseau ni graphique.

## Arborescence

```text
aqua-morph/
├── CMakeLists.txt
├── README.md
├── assets/            images et polices
├── data/
│   ├── config/        simulation.json, render.json, learning.json
│   └── results/       manifestes, statistiques, snapshots, checkpoints
├── docs/              documents de conception
├── lib/               dépendances téléchargées par CMake
├── include/aquamorph/ en-têtes (.hpp) : core, common, ocean, agent, render
├── src/               sources (.cpp) : core, common, ocean, agent, render, apps
└── tests/             tests unitaires et d'intégration
```

Le détail complet des fichiers est dans `docs/` (« Plan de développement », section 3).

## Prérequis

- Un compilateur C++20 (GCC, Clang ou MSVC).
- CMake.
- Git et un accès réseau à la première configuration : CMake télécharge les dépendances dans `lib/`.
- Sous Linux, les paquets système de développement requis par SFML (fenêtrage, OpenGL, etc.), que CMake ne télécharge pas.
- Systèmes pris en charge : Linux et Windows.

Dépendances téléchargées automatiquement par CMake (versions épinglées) :

| Bibliothèque | Usage |
|---|---|
| nlohmann/json | JSON et MessagePack |
| Eigen | Algèbre linéaire (agent) |
| SFML | Rendu |
| ZeroMQ (libzmq) | Transport entre l'océan et l'agent |

Le binding C++ de ZeroMQ et le framework de tests restent à choisir.

## Compilation

```bash
git clone <url-du-depot> aqua-morph
cd aqua-morph
cmake -S . -B build
cmake --build build
```

## Utilisation

Dans deux terminaux, lancer l'océan puis l'agent :

```bash
# Terminal 1 : simulation sans graphique
aquamorph-ocean --headless --config data/config

# Terminal 2 : agent avec le contrôleur sinusoïdal (phase 1)
aquamorph-agent --policy=sinus
```

Options principales :

| Programme | Option | Effet |
|---|---|---|
| `aquamorph-ocean` | `--headless` | Simulation sans rendu, aussi vite que le processeur le permet |
| `aquamorph-ocean` | `--render` | Affiche la simulation avec SFML (caméra déplaçable et zoomable) |
| `aquamorph-ocean` | `--config <dossier>` | Dossier des fichiers de configuration |
| `aquamorph-agent` | `--policy=network\|sinus\|still\|random` | Politique utilisée |
| `aquamorph-agent` | `--checkpoint <fichier>` | Reprise depuis un checkpoint du réseau |

Les binaires sont produits dans le dossier de compilation (`build/`).

## Configuration et résultats

Trois fichiers JSON dans `data/config/` :

- `simulation.json` : physique, bornes du génome, transport ;
- `render.json` : fenêtre, caméra, échelle en pixels par unité du monde ;
- `learning.json` : paramètres de REINFORCE, taille des couches, contrôleur sinusoïdal, seuils d'alerte.

Une clé absente ou hors bornes provoque une erreur au lancement : il n'y a pas de valeur par défaut silencieuse.

Les résultats d'une exécution (statistiques par génération au format JSONL, snapshots de population, checkpoints binaires du réseau, manifeste d'exécution) sont écrits dans `data/results/`.

## Reproductibilité

En mode lockstep, une exécution est reproductible bit à bit avec la même graine, le même binaire, la même machine et le même nombre de threads d'Eigen. Le rendu n'a aucune influence sur les résultats.

## Tests

Les tests sont regroupés dans la cible `aquamorph-tests` : génération et réparation des génomes, appariement, physique (conservation de la quantité de mouvement, absence d'énergie gratuite), gradient du réseau par différences finies, déterminisme, échanges océan-agent en `inproc://`.

Le framework de tests (Catch2 ou GoogleTest) n'est pas encore choisi.

## Feuille de route

| Version | Contenu |
|---|---|
| **1.0.0** | Morphologie, physique dans l'eau (traînée, gravité, flottabilité), sélection, reproduction, agent d'apprentissage |
| 2.0 | Plongée, nourriture, prédation, couleurs |
| 3.0 | Espace et temps, interactions entre espèces, comportements de groupe, saisons |

Jalons de la version 1.0.0 : M0 squelette, M1 RNG et génome, M2 physique, M3 transport, M4 phase 1 (contrôleur sinusoïdal), M5 évolution, MR rendu, M6 réseau, M7 REINFORCE, M8 bout en bout.

## Documentation

Les documents de conception sont dans `docs/` :

- Roadmap du projet
- Spécification technique
- Plan de développement
- Schéma du génome
- Protocole et pas de simulation
- Transport ZeroMQ et format des messages
- Spécification de l'agent IA
- Spécification du contrôleur sinusoïdal
- Métriques et journalisation

## Conventions de code

- C++20, C++ Core Guidelines.
- PascalCase pour les classes, camelCase pour les fonctions et variables, snake_case pour les fichiers.
- `.hpp` dans `include/aquamorph/<module>/`, `.cpp` dans `src/<module>/`.
- Le moteur de simulation est séparé du rendu : le rendu ne modifie jamais l'état de la simulation.

## Licence

À définir.
