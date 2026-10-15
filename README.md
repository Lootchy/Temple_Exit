# Temple_Exit

Jeu d'aventure / puzzle dans un temple en ruine, 100 % Blueprint.
Le joueur doit s'échapper du temple en manipulant l'environnement : objets physiques, mécanismes, pièges et décors destructibles (Chaos).

- **Map principale** : `Content/Map/MainMap.umap`

---

## Mechanics

### Contrôles

| Touche | Action | Input Action |
|---|---|---|
| ZQSD / WASD | Se déplacer | `IA_Move` |
| Souris | Caméra | `IA_Look` / `IA_MouseLook` |
| **E** | Interagir | `IA_Interact` |
| **F** | Utiliser l'objet tenu (lancer) | `IA_UseItem` |
| **R** | Lâcher | `IA_Grab` |

Les indications (`E to interact`, `F to use`, `R to drop`) sont affichées par le widget `UI_Player`.

### Joueur
- **Interaction** (`BPC_Interact`) : line trace depuis la caméra vers l'avant. Si l'acteur touché implémente `BPI_Interact`, il est mis en surbrillance (outline) et peut être utilisé avec **E**.
- **Grab** (`BPC_Grab`) : attrape un objet physique avec un `PhysicsHandle` et le maintient devant la caméra ; **R** le relâche.
- **Lancer** (`BP_LaunchBall`) : un objet tenu peut être lancé avec **F**. La trajectoire est prévisualisée (`PredictProjectilePath`) puis la balle reçoit une impulsion dans la direction de la caméra.

### Puzzles & mécanismes
| Mécanique | Blueprint | Fonctionnement |
|---|---|---|
| Bouton | `BP_Button` → `BP_DemolitionBall` | Le bouton se déclenche quand il est touché par une `BP_LaunchBall` → l'event `E_ButtonPressed` lâche la boule de démolition (physique + constraint). |
| Pilier | `BP_Pillar` | Cassé par la boule de démolition (hit) → son + camera shake. |
| Rocher roulant | `BP_RollingStone` | Passe en simulation physique au déclenchement, vitesse clampée, son de tremblement de terre + camera shake (`CS_Rolling`). |
| Mur destructible | `BP_Wall` | Geometry Collection qui se fracture sous l'impact du rocher (seuil de dégâts), camera shake (`CS_Destruction`). |
| Jump Pad | `BP_JumpPad` | Overlap → `LaunchCharacter` + VFX Niagara (`NS_JumpPa`) animé par Timeline. |
| Sol glissant | `BP_SlipperyFloor` | Overlap → réduit `GroundFriction` / `BrakingDecelerationWalking` du joueur, valeurs restaurées en sortie. Matériau de glace *IceCool*. |
| Pont cassable | `BP_BreakableBridge` | Assemblage de `BP_BreakableGround` + `BP_Anchor` (Field System) pour créer un pont qui s'effondre. |

---

## Architecture

### Pattern commun : état + Event Dispatcher
Chaque mécanisme suit le même schéma :

1. Un **enum d'état** décrit où il en est.
2. Quand l'état change, le Blueprint **broadcast un Event Dispatcher** avec le nouvel état.
3. Les Blueprints abonnés (et le BP lui-même) reçoivent l'event `State` et font un **Switch on Enum** pour réagir.

Le déclencheur ne connaît pas ce qu'il déclenche : c'est la cible qui s'abonne au dispatcher (référence à l'instance placée dans le niveau).

| Blueprint | Enum d'état | Valeurs | Dispatcher |
|---|---|---|---|
| `BP_Button` | `E_ButtonState` | Normal → Pressed | `E_ButtonPressed` |
| `BP_Wall` | `E_Wall` | Normal → Break | `E_StateChanged` |
| `BP_RollingStone` | `E_StoneState` | Freeze → Move | *(fonction `Change State`)* |

- **Joueur → objets** : `BPC_Interact` ne caste jamais vers un type d'objet précis. Il appelle les fonctions de l'interface `BPI_Interact` (`Interact`, `Show Outline`, `Release`) sur l'acteur visé ; `BP_InteractActor` est la classe parent des objets interactifs.
- **Bouton → boule de démolition → pilier** : la balle lancée active le bouton, qui passe en `Pressed` et broadcast `E_ButtonPressed`. `BP_DemolitionBall` est abonnée et active sa physique, puis la collision avec `BP_Pillar` le fait casser.
- **Trigger → rocher → mur** : le Level Blueprint de `MainMap` détecte le joueur dans une `TriggerBox` et appelle `Change State(Move)` sur `BP_RollingStone`. Quand le rocher percute `BP_Wall` (cast vers `BP_RollingStone` sur l'event Hit), le mur passe en `Break` et broadcast `E_StateChanged` pour lancer la fracture.
- **Pièges autonomes** (`BP_JumpPad`, `BP_SlipperyFloor`, `BP_BreakableGround`) : aucun dispatcher. Un overlap avec le joueur agit directement sur lui (`LaunchCharacter`, paramètres du `CharacterMovement`) ou sur leur propre Geometry Collection.

---

## Optimisations

### Gameplay / Blueprints
- **Pas de Event Tick** : la plupart des Blueprints ont `Start with Tick Enabled` désactivé ou enlevé. La logique est **événementielle** (overlap, hit, input, dispatchers) au lieu de vérifier un état à chaque frame.
- **Timers au lieu du Tick** : le line trace d'interaction et la mise à jour du `PhysicsHandle` tournent sur des timers (`SetTimer`), qu'on peut arrêter quand ils ne servent pas.
- **Do Once** sur les déclencheurs à usage unique (bouton, mur, pilier) pour éviter de relancer la logique, les sons et la casse plusieurs fois.
- **Interface + dispatchers** : moins de casts et de références dures entre Blueprints → moins de dépendances chargées en mémoire.

### Physique / Destruction (Chaos)
- Les **Geometry Collections** restent **endormies / ancrées** (`BP_Anchor`, état *Sleeping*) tant qu'elles ne sont pas déclenchées : aucun coût de simulation au repos.
- La casse est déclenchée **à la demande** (`CrumbleActiveClusters`, seuil de dégâts) plutôt que simulée en permanence.
- Après la casse, la collision avec le Pawn est **désactivée** sur les débris pour éviter les calculs de collision inutiles (et que le joueur reste bloqué).

### Rendu / Level
- **Packed Level Actor** (`BPP_PCA_Floor`) : les tuiles du sol sont regroupées en **Hierarchical Instanced Static Meshes** → un draw call par mesh au lieu d'un par tuile.
- **Camera shakes partagés** (`BP_LegacyCameraShake`, `CS_*`) réutilisés par plusieurs Blueprints.