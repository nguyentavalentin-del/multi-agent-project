# Système Multi-Agents — Négociation de Budget

Projet de **système multi-agents** réalisé dans le cadre du Master 1 Intelligence Artificielle Distribuée.

Le projet utilise le framework **JADE** pour simuler une négociation entre deux agents autonomes représentant les principaux acteurs de la production d'un film :

- 🎬 **DirectorAgent** — défend les intérêts artistiques du réalisateur.
- 💰 **ProducerAgent** — défend les intérêts financiers et commerciaux du producteur.

Les deux agents poursuivent un objectif commun — produire le film — tout en ayant des intérêts divergents. Cette opposition permet de simuler une négociation avec des propositions, contre-offres et concessions progressives.

---

## Fonctionnement

La négociation se déroule en **deux phases successives** :

### Phase 1 — Négociation du budget total

Les agents négocient le budget global du film.

Le réalisateur dispose d'un **budget minimum souhaité**, tandis que le producteur possède une **fourchette budgétaire acceptable**, définie par un budget minimum et un budget maximum.

À chaque tour :

1. Un agent envoie une proposition.
2. L'autre agent l'évalue.
3. La proposition peut être acceptée ou rejetée.
4. En cas de rejet, une contre-offre est générée.
5. Les agents réalisent progressivement des concessions.

Le nombre maximal de rounds est fixé à **5**.

La stratégie de concession est progressive :

| Round | Réalisateur | Producteur |
|---|---:|---:|
| 1 | Min + 20 % | Max - 20 % |
| 2 | Min + 15 % | Max - 15 % |
| 3 | Min + 10 % | Max - 10 % |
| 4 | Min + 5 % | Max - 5 % |
| 5 | Min | Max |

---

### Phase 2 — Répartition du budget

Une fois le budget total accepté, les agents négocient sa répartition entre cinq postes :

- **Script**
- **Production**
- **Casting**
- **VFX**
- **Music**

Chaque agent possède :

- des priorités différentes pour chaque poste ;
- des planchers budgétaires minimaux ;
- une stratégie de concession.

Une proposition doit respecter les **planchers minimaux** pour être considérée comme valide.

La première offre de chaque agent est calculée proportionnellement à ses priorités. Les contre-offres rapprochent progressivement la proposition de l'agent de celle de son adversaire, tout en conservant les contraintes minimales.

---

## Architecture

Les agents communiquent exclusivement à travers des **messages ACL de JADE**, conformément au standard FIPA.

Le contenu des messages est sérialisé en **JSON**.

```text
                    ┌─────────────────────┐
                    │      JADE          │
                    │   Agent Platform   │
                    └──────────┬──────────┘
                               │
                 Messages ACL / JSON
                               │
              ┌────────────────┴────────────────┐
              │                                 │
      ┌───────▼────────┐               ┌────────▼───────┐
      │ DirectorAgent  │               │ ProducerAgent  │
      │                │               │                │
      │ Intérêts       │               │ Intérêts       │
      │ artistiques    │               │ commerciaux    │
      └───────┬────────┘               └────────┬───────┘
              │                                 │
              └─────────── Négociation ─────────┘
                         │
                 ┌───────▼───────┐
                 │    Phase 1    │
                 │ Budget total  │
                 └───────┬───────┘
                         │
                 Budget accepté
                         │
                 ┌───────▼───────┐
                 │    Phase 2    │
                 │ Répartition   │
                 └───────────────┘
```

---

## Behaviours JADE

Chaque agent est organisé autour d'un `FSMBehaviour` permettant d'orchestrer les différents états de la négociation.

Les états sont implémentés sous forme de `OneShotBehaviour`.

| Behaviour | Agent | Rôle |
|---|---|---|
| `SendPropositionBehaviour` | Les deux | Envoie une proposition |
| `WaitPropositionBehaviour` | Les deux | Attend et reçoit une proposition |
| `ProducerEvaluateBehaviour` | Producteur | Évalue une proposition en phase 1 |
| `DirectorEvaluateBehaviour` | Réalisateur | Évalue une proposition en phase 1 |
| `AcceptBehaviour` | Les deux | Accepte une proposition |
| `CancelBehaviour` | Les deux | Met fin à la négociation |
| `EndBehaviour` | Les deux | Termine l'agent après un `CANCEL` |
| `Phase2InitBehaviour` | Les deux | Initialise la phase 2 |
| `RepartitionEvaluateBehaviour` | Les deux | Évalue une proposition de phase 2 |

---

## Communication

Les messages ACL utilisent principalement les performatives suivantes :

| Performative | Phase | Utilisation |
|---|---|---|
| `PROPOSE` | 1 & 2 | Proposition ou contre-offre |
| `ACCEPT_PROPOSAL` | 1 & 2 | Acceptation d'une proposition |
| `CANCEL` | 1 & 2 | Rejet et arrêt de la négociation |

Les propositions sont transmises sous forme de données **JSON**.

---

## Stratégies des agents

### 🎬 DirectorAgent

Le réalisateur cherche principalement à maximiser les ressources disponibles pour garantir la qualité artistique et technique du film.

Ses priorités sont notamment :

- Production
- VFX
- Music

Le réalisateur est davantage disposé à faire des concessions sur le **Casting**.

Il définit également un plancher minimal pour chaque poste budgétaire.

### 💰 ProducerAgent

Le producteur cherche principalement à :

- contrôler les coûts ;
- optimiser la rentabilité ;
- maintenir un budget réaliste ;
- favoriser les éléments ayant un fort impact commercial.

Il accorde notamment une forte importance au **Casting**, tandis que la musique et le scénario sont moins prioritaires.

---

## Priorités en phase 2

Les priorités déterminent la résistance d'un agent aux concessions.

| Poste | Réalisateur | Producteur |
|---|---:|---:|
| Script | 25 | 25 |
| Production | 90 | 50 |
| Casting | 50 | 90 |
| VFX | 50 | 50 |
| Music | 25 | 25 |

Plus la priorité est élevée, plus l'agent résiste aux concessions sur le poste concerné.

---

## Planchers budgétaires

Les planchers sont calculés dynamiquement à partir du budget total obtenu en phase 1.

| Poste | Réalisateur | Producteur |
|---|---:|---:|
| Script | 4 % | 3 % |
| Production | 16 % | 9 % |
| Casting | 3 % | 14 % |
| VFX | 11 % | 11 % |
| Music | 6 % | 3 % |

Une proposition passant sous un plancher est considérée comme invalide.

---

## Génération des offres

### Première offre

La première offre est calculée proportionnellement aux priorités de l'agent :

```text
montant du poste =
    priorité du poste
    ----------------- × budget total
    somme des priorités
```

Les montants sont ensuite ajustés afin de respecter les planchers minimaux.

### Contre-offres

Lorsqu'un agent reçoit une proposition, il calcule une nouvelle répartition en se rapprochant progressivement de la proposition adverse.

Le taux de concession augmente avec le numéro d'itération et est plafonné à **80 %**.

```text
écart = valeur_ideale - valeur_reçue

taux_concession =
    taux_base × (itération / max_itération)

nouvelle_valeur =
    valeur_ideale - (écart × taux_concession)
```

Le plancher du poste est toujours respecté.

Un rééquilibrage est ensuite effectué afin que la somme des postes corresponde au budget total.

---

## Interface `Phase2Agent`

Les deux agents implémentent l'interface commune `Phase2Agent`.

Cette interface permet de mutualiser la logique de la phase 2 et d'éviter la duplication du code.

Elle fournit notamment les fonctionnalités suivantes :

```java
setBudgetAccorde(int budget)
getBudgetAccorde()
getDecision2(...)
genererContreOffrePhase2(...)
genererPremiereOffrePhase2()
getPlanchers()
```

Cette approche permet notamment de réutiliser les behaviours de la phase 2 pour les deux agents.

---

## Structure du projet

Le projet est organisé en trois grandes parties :

```text
src/
├── agents/
│   ├── DirectorAgent.java
│   ├── ProducerAgent.java
│   └── Phase2Agent.java
│
├── behaviours/
│   ├── SendPropositionBehaviour.java
│   ├── WaitPropositionBehaviour.java
│   ├── ProducerEvaluateBehaviour.java
│   ├── DirectorEvaluateBehaviour.java
│   ├── AcceptBehaviour.java
│   ├── CancelBehaviour.java
│   ├── EndBehaviour.java
│   ├── Phase2InitBehaviour.java
│   └── RepartitionEvaluateBehaviour.java
│
└── utils/
    ├── Proposition.java
    ├── RepartitionBudget.java
    └── Planchers.java
```

### `agents/`

Contient les agents JADE et l'interface commune utilisée pour la phase 2.

### `behaviours/`

Contient les comportements utilisés pour gérer les différents états de la négociation.

### `utils/`

Contient les structures de données utilisées pendant les échanges, notamment les propositions, les répartitions budgétaires et les planchers.

---

## Technologies utilisées

- **Java**
- **JADE** — Java Agent DEvelopment Framework
- **Messages ACL / FIPA**
