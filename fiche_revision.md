# Fiche de Révision — Entretien Technique Senior
**Cabinet Athoria — AI Engineer**
*Manuel de Référence — Niveau Senior*

---

## PILIER I — Mécanismes Internes de Python

---

### 1.1 Le GIL — Global Interpreter Lock

**Définition formelle**
Le **GIL** est un mutex (verrou d'exclusion mutuelle) implémenté dans CPython qui garantit qu'un seul thread natif du système d'exploitation exécute du **bytecode Python** à un instant donné, quel que soit le nombre de threads créés dans le programme.

**Pourquoi le GIL existe**
- CPython gère la mémoire via le **comptage de références** (voir 1.2). Ce compteur est une variable partagée entre tous les threads.
- Sans verrou global, deux threads pouvant incrémenter/décrémenter simultanément `ob_refcnt` provoqueraient des **race conditions**, menant à une libération prématurée d'objet ou à une fuite mémoire.
- Le GIL est donc une solution pragmatique pour garantir la **thread-safety du gestionnaire de mémoire** sans protéger chaque objet individuellement (ce qui serait trop coûteux).

**Mécanisme interne de commutation**
- CPython utilise un compteur d'instructions bytecode (le **sys.getswitchinterval()**, par défaut 5 ms depuis Python 3.2).
- Toutes les `N` ms, le thread courant libère le GIL, permettant à un autre thread de l'acquérir.
- La commutation est coopérative pour les **I/O-bound operations** : un thread libère le GIL dès qu'il appelle une opération I/O bloquante (socket, fichier), car la C-extension sous-jacente peut s'exécuter sans bytecode Python.

**Impact sur le parallélisme**

| Scénario | Threads (`threading`) | Processus (`multiprocessing`) |
|---|---|---|
| **CPU-bound** (calcul pur) | Pas de gain réel — le GIL sérialise l'exécution | Gain linéaire — chaque processus a son propre interpréteur et GIL |
| **I/O-bound** (réseau, disque) | Gain effectif — le GIL est libéré pendant l'attente I/O | Overhead de fork inutile |

**Termes académiques associés** : mutex, race condition, bytecode, ob_refcnt, thread-safety, coopérative vs préemptive scheduling, CPython, Jython (pas de GIL), GIL-free Python (PEP 703 — nogil branch, Python 3.13 expérimental).

---

### 1.2 Gestion Mémoire & Garbage Collector

#### Comptage de Références (Reference Counting)

**Définition formelle**
Chaque objet Python possède un champ `ob_refcnt` (entier) dans sa structure C (`PyObject`). Ce compteur est incrémenté à chaque nouvelle référence vers l'objet, et décrémenté à chaque suppression de référence. Quand `ob_refcnt == 0`, la mémoire est **immédiatement libérée** par `Py_DECREF()`.

**Mécanisme interne**
```
ob_refcnt += 1  →  lors de :  x = obj, passage en argument, ajout en liste
ob_refcnt -= 1  →  lors de :  del x, sortie de scope, remplacement de variable
ob_refcnt == 0 →  __del__() appelé, mémoire retournée à l'allocateur
```

**Limite fondamentale : les cycles de références**
- Si A référence B et B référence A, les deux `ob_refcnt` ne tombent jamais à 0 même si aucune variable externe ne pointe vers eux.
- Le comptage de références est **incapable de détecter ces cycles** → nécessite un GC cyclic supplémentaire.

#### Generational Garbage Collection (GC Cyclique)

**Pourquoi trois générations ?**
Basé sur l'**hypothèse générationnelle** (weak generational hypothesis) : *"la majorité des objets meurent jeunes"*. Empiriquement validé sur de nombreux programmes. L'organisation en générations permet de ne scanner fréquemment que les objets récents (plus susceptibles d'être des déchets), et rarement les anciens (plus susceptibles d'être des références longues durée).

| Génération | Contenu | Fréquence de collecte | Seuil par défaut |
|---|---|---|---|
| **Gen 0** | Nouveaux objets créés | Très fréquente | 700 allocations nettes |
| **Gen 1** | Survivants de Gen 0 | Modérée | 10 collectes de Gen 0 |
| **Gen 2** | Survivants de Gen 1 | Rare | 10 collectes de Gen 1 |

**Algorithme de détection des cycles**
1. Le GC maintient une **liste doublement chaînée** de tous les objets *container* (listes, dicts, instances) dans chaque génération.
2. Pour chaque collecte de génération G, le GC calcule un `gc_refs` (copie locale de `ob_refcnt`).
3. Il parcourt tous les objets de G et décrémente `gc_refs` de chaque objet référencé (simule la suppression des références internes au groupe).
4. Tout objet dont `gc_refs == 0` après ce parcours est **inatteignable** (ses seules références sont internes au cycle) → il est collecté.
5. Les survivants sont promus en génération G+1.

**Termes académiques associés** : `PyObject`, `Py_INCREF`/`Py_DECREF`, weak generational hypothesis, tracing GC, mark-and-sweep, cyclic reference, `gc.collect()`, `gc.disable()`, finalizers, `__del__`.

---

### 1.3 Concepts Avancés Python

#### Décorateurs & Clôtures (Closures)

**Définition formelle — Closure**
Une **clôture** est une fonction imbriquée qui capture et conserve des références aux variables de son **environnement lexical** (scope englobant), même après que ce scope ait terminé son exécution. Formellement : une clôture = une fonction + son environnement de liaison de variables libres.

**Mécanisme interne**
- Python stocke les variables capturées dans l'attribut `__closure__` de la fonction, sous forme de cellules (`cell` objects).
- La variable libre est partagée par référence via la cellule : modifier la variable dans la closure affecte l'environnement d'origine et vice-versa.
- En Python, les closures ont accès en **lecture-écriture** aux variables du scope englobant avec le mot-clé `nonlocal`.

**Définition formelle — Décorateur**
Un **décorateur** est un **callable** qui prend une fonction en argument et retourne une nouvelle fonction (ou callable). C'est du **sucre syntaxique** pour la composition de fonctions de haut ordre.

```python
@decorator
def func(): ...
# équivalent strict à :
func = decorator(func)
```

**Pattern fondamental** (préservation des métadonnées avec `functools.wraps`) :
```python
import functools
def decorator(func):
    @functools.wraps(func)  # copie __name__, __doc__, __annotations__
    def wrapper(*args, **kwargs):
        # pré-traitement
        result = func(*args, **kwargs)
        # post-traitement
        return result
    return wrapper
```

**Termes académiques associés** : higher-order function, lexical scoping, free variable, `__closure__`, `cell` object, `nonlocal`, `functools.wraps`, `__wrapped__`, first-class function.

---

#### Générateurs & Protocole d'Itération

**Définition formelle — Protocole d'itération**
Le **protocole d'itération** Python repose sur deux interfaces :
- **Iterable** : tout objet implémentant `__iter__()` qui retourne un **itérateur**.
- **Iterator** : tout objet implémentant `__iter__()` (retourne self) ET `__next__()` qui retourne l'élément suivant ou lève `StopIteration`.

**Définition formelle — Générateur**
Un **générateur** est une fonction contenant le mot-clé `yield`. Son appel ne l'exécute pas mais retourne un **objet générateur** (qui implémente le protocole iterator). L'exécution est **suspendue** à chaque `yield` et reprend à l'appel suivant de `__next__()`.

**Mécanisme interne**
- Lorsqu'une fonction générateur est appelée, Python crée un **frame d'exécution** (`PyFrameObject`) mais ne l'exécute pas immédiatement.
- À chaque `__next__()`, Python reprend l'exécution du frame depuis le dernier point de suspension.
- L'état local complet (variables locales, pointeur d'instruction) est préservé dans le frame entre deux appels.
- `yield` est une expression bidirectionnelle : `x = yield value` permet d'envoyer une valeur dans le générateur via `gen.send(value)` → base des **coroutines**.

**Avantage fondamental** : évaluation **lazy** (paresseuse) — la mémoire n'est allouée que pour un élément à la fois, indépendamment de la taille du flux.

**Termes académiques associés** : lazy evaluation, `PyFrameObject`, coroutine, `send()`, `throw()`, `close()`, `yield from`, `StopIteration`, `GeneratorExit`, `asyncio` (coroutines basées sur les générateurs), PEP 255, PEP 342, PEP 380.

---

#### MRO — Method Resolution Order

**Définition formelle**
Le **MRO** est l'ordre linéaire dans lequel Python recherche une méthode ou un attribut dans la hiérarchie de classes lors d'un héritage multiple. Depuis Python 3, il est calculé par l'algorithme **C3 Linearization**.

**Algorithme C3 — Mécanisme**
Pour une classe `C(B1, B2, ...)`, la linéarisation L[C] est calculée récursivement :

```
L[C] = C + merge(L[B1], L[B2], ..., [B1, B2, ...])
```

La fonction `merge` prend la tête de la première liste si cette tête n'apparaît **pas dans la queue** d'une autre liste. Sinon, elle essaie la liste suivante.

**Exemple canonique du Problème du Diamant** :
```
    A
   / \
  B   C
   \ /
    D
```
`L[D] = D → B → C → A → object`

C3 garantit deux propriétés : **monotonie** (si A précède B dans L[C], alors A précède B dans toute sous-classe de C) et **préservation de l'ordre local** (l'ordre déclaré des parents est respecté).

**Consultation** : `ClassName.__mro__` ou `ClassName.mro()`.

**Termes académiques associés** : C3 linearization, Diamond Problem, cooperative multiple inheritance, `super()`, MRO, monotonicity, local precedence ordering, `__mro__`.

---

## PILIER II — Architecture IA & GenAI (Théorie Pure)

---

### 2.1 Transformer — Architecture Interne

#### Scaled Dot-Product Attention

**Définition formelle**
L'attention est un mécanisme permettant à chaque token d'une séquence de pondérer dynamiquement sa dépendance à tous les autres tokens. La formule canonique de **Scaled Dot-Product Attention** est :

$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right) V$$

**Décomposition des termes**
- **Q (Query)** : représentation de ce que le token courant "cherche".
- **K (Key)** : représentation de ce que chaque token "offre".
- **V (Value)** : représentation de ce que chaque token "transmet" si sélectionné.
- **$d_k$** : dimension des vecteurs clés (typiquement 64 dans les configs d'origine).
- **$QK^T$** : matrice de scores de compatibilité bruts (dot-product).
- **Facteur $\sqrt{d_k}$** : facteur d'échelle pour **contrer la saturation du softmax**.

**Pourquoi diviser par $\sqrt{d_k}$ ?**
En grande dimension, le produit scalaire $QK^T$ tend à avoir une variance croissante proportionnelle à $d_k$ (si les composantes sont i.i.d. de moyenne 0 et variance 1, alors $\text{Var}(q \cdot k) = d_k$). Des valeurs trop grandes poussent le softmax dans des zones à **gradient quasi-nul** (saturation), bloquant l'apprentissage. La division par $\sqrt{d_k}$ ramène la variance à 1.

**Multi-Head Attention**
Plutôt qu'une seule attention, le Transformer exécute `h` attentions en parallèle sur des **projections linéaires** différentes de Q, K, V :

$$\text{MultiHead}(Q, K, V) = \text{Concat}(\text{head}_1, ..., \text{head}_h) W^O$$

Chaque tête apprend à capturer un type de dépendance différent (syntaxique, sémantique, positionnelle).

**Complexité computationnelle** : $O(n^2 \cdot d)$ en temps et en mémoire — quadratique en longueur de séquence, ce qui explique les efforts sur **Flash Attention** (chunking sur GPU pour économiser la VRAM) et **Sparse Attention**.

---

#### Positional Encoding — Pourquoi Sinus/Cosinus ?

**Problème** : l'attention est une opération **invariante à la permutation** (un ensemble, pas une séquence). Sans encodage positionnel, le modèle est incapable de distinguer "le chat mange la souris" de "la souris mange le chat".

**Solution de Vaswani et al. (2017)** — encodage sinusoïdal fixe :

$$PE_{(pos, 2i)} = \sin\left(\frac{pos}{10000^{2i/d_{model}}}\right)$$
$$PE_{(pos, 2i+1)} = \cos\left(\frac{pos}{10000^{2i/d_{model}}}\right)$$

**Justifications formelles du choix sinus/cosinus**

1. **Représentation relative** : pour tout décalage fixe $k$, $PE_{pos+k}$ peut s'exprimer comme une **combinaison linéaire** de $PE_{pos}$ (propriété de translation des fonctions trigonométriques). Cela permet au modèle d'apprendre des relations relatives entre positions.

2. **Fréquences multiples** : les différentes dimensions $i$ correspondent à des fréquences géométriquement espacées (de $1$ à $1/10000$). Les basses fréquences encodent les positions globales, les hautes fréquences les positions locales — analogue à un système de numération en base mixte.

3. **Déterminisme et généralisation** : l'encodage est fixe (non appris), permettant de généraliser à des séquences plus longues que celles vues à l'entraînement (contrairement aux embeddings positionnels appris).

4. **Norme bornée** : $|PE| \leq 1$ par construction, ce qui évite de perturber les embeddings sémantiques.

**Alternatives modernes** : RoPE (Rotary Position Embedding — LLaMA), ALiBi (Attention with Linear Biases — BLOOM), positions relatives (T5).

---

#### Architectures Encoder-only vs Decoder-only

| Propriété | **Encoder-only (BERT)** | **Decoder-only (GPT)** |
|---|---|---|
| **Mécanisme d'attention** | **Bidirectionnel** — chaque token voit tous les autres | **Masqué (causal)** — chaque token ne voit que les tokens précédents |
| **Masque d'attention** | Pas de masque (sauf padding) | **Causal mask** : triangulaire inférieur |
| **Pré-entraînement** | Masked Language Modeling (MLM) — prédire tokens masqués | Causal Language Modeling (CLM) — prédire le prochain token |
| **Cas d'usage** | Classification, NER, QA extractif | Génération de texte, completion, agent |
| **Représentation** | Contextuelle bidirectionnelle (riche en compréhension) | Contextuelle causale (riche en génération) |

**Architecture Encoder-Decoder (T5, BART)** : l'encoder traite la séquence source bidirectionnellement, le decoder génère la sortie token par token avec **cross-attention** sur les sorties de l'encoder.

**Termes académiques associés** : self-attention, cross-attention, causal masking, MLM, CLM, layer normalization, residual connection, feed-forward network, tokenization (BPE, WordPiece, SentencePiece), temperature, top-k, top-p (nucleus sampling).

---

### 2.2 Composants du RAG — Retrieval-Augmented Generation

#### L'Espace Latent

**Définition formelle**
L'**espace latent** (ou espace d'embedding) est un espace vectoriel de dimension $d$ (typiquement $d \in \{384, 768, 1536\}$) dans lequel un modèle d'encodage (embedding model) projette des données sémantiques discrètes (texte, images) en vecteurs denses continus. La **proximité géométrique** dans cet espace est supposée refléter la **similarité sémantique** dans l'espace original.

**Propriété fondamentale** : l'espace latent est appris par le modèle d'encodage lors de son pré-entraînement (contrastive learning, MNRL — Multiple Negatives Ranking Loss), de sorte que des représentations sémantiquement similaires soient proches selon la métrique choisie.

---

#### Métriques de Distance dans les Vector Stores

**Similarité Cosinus**
$$\cos(\theta) = \frac{u \cdot v}{\|u\| \cdot \|v\|}$$

- Mesure l'**angle** entre deux vecteurs, invariante à leur norme.
- Range : $[-1, 1]$, où 1 = même direction (identique), 0 = orthogonal, -1 = opposé.
- **Utilisation** : universelle en NLP — les embeddings textuels diffèrent souvent en magnitude (longueur du texte) mais la direction encode la sémantique.

**Distance Euclidienne (L2)**
$$d_E(u, v) = \sqrt{\sum_i (u_i - v_i)^2}$$

- Mesure la **distance géométrique** absolue entre deux points.
- Sensible à la norme des vecteurs — deux vecteurs parallèles mais de normes différentes auront une grande distance L2.
- **Utilisation** : recommandée quand les vecteurs sont normalisés (L2-normalization), auquel cas distance L2 et similarité cosinus sont **monotoniquement équivalentes**.

**Inner Product (Produit Scalaire)**
$$\langle u, v \rangle = \sum_i u_i \cdot v_i = \|u\| \cdot \|v\| \cdot \cos(\theta)$$

- Capture à la fois l'angle et la magnitude.
- **Utilisation** : modèles entraînés avec Maximum Inner Product Search (MIPS), comme les modèles d'embedding OpenAI (text-embedding-ada-002). Incorrecte si les vecteurs ne sont pas normalisés et si l'espace latent ne garantit pas de magnitude uniforme.

**Choix pratique** : toujours utiliser la métrique prescrite par le modèle d'embedding utilisé — un mauvais choix dégrade significativement le rappel du système de recherche.

---

#### Stratégies de Chunking

**Définition** : le chunking est le processus de découpage des documents sources en **segments** (chunks) avant leur vectorisation et indexation dans le vector store.

| Stratégie | Mécanisme | Avantages | Limitations |
|---|---|---|---|
| **Fixed-size** | Découpage par nombre fixe de tokens/caractères | Simple, prédictible | Coupe les phrases, perd le contexte |
| **Sentence-based** | Découpage aux limites de phrases (NLTK, spaCy) | Préserve la cohérence sémantique locale | Chunks de taille variable |
| **Recursive** (LangChain) | Essaie successivement des séparateurs (`\n\n`, `\n`, `. `, ` `) | Préserve la structure du document | Paramétrage dépendant du corpus |
| **Semantic** | Regroupement par similarité d'embeddings entre phrases adjacentes | Chunks thématiquement cohérents | Coûteux en inférence |
| **Agentic / Contextual** | LLM génère un contexte global pour chaque chunk (Anthropic) | Très haute précision | Très coûteux |

**Paramètre critique : overlap** — le chevauchement entre chunks consécutifs (e.g., 20% de recouvrement) évite de couper une unité sémantique à la frontière de deux chunks.

---

#### Re-ranking & Cross-Encoder

**Architecture du pipeline RAG à deux étapes**

```
Query → [Bi-Encoder] → Top-K candidats (rappel) → [Cross-Encoder] → Top-N rerankés (précision)
```

**Bi-Encoder (retrieval)**
- Encode la query et chaque document **indépendamment**.
- La similarité est calculée **après** l'encodage (dot-product sur embeddings précomputés).
- Complexité : $O(1)$ par document à l'inférence (embeddings stockés).
- Avantage : rapide et scalable. Limitation : pas d'interaction fine entre query et document.

**Cross-Encoder (re-ranking)**
- Prend la **paire** (query, document) comme entrée concaténée dans un seul modèle.
- Le modèle produit un score de pertinence en modélisant les interactions **token-à-token** entre query et document.
- Complexité : $O(K)$ inférences complètes (coûteux mais appliqué sur un petit K).
- Avantage : précision très supérieure au bi-encoder car l'attention bidirectionnelle capture les relations fines. Utilisé pour reranker les top-K résultats du retrieval initial.

**Termes académiques associés** : embedding model, vector store, HNSW (Hierarchical Navigable Small World — algorithme d'indexation ANN), ANN (Approximate Nearest Neighbor), IVF (Inverted File Index), FAISS, ChromaDB, Pinecone, Weaviate, RAGas (framework d'évaluation RAG), faithfulness, answer relevance, context recall.

---

## PILIER III — Génie Logiciel & Architecture

---

### 3.1 Principes REST — Fielding Constraints

**Définition formelle**
**REST** (Representational State Transfer) est un **style architectural** défini par Roy Fielding dans sa thèse de doctorat (2000). Ce n'est pas un protocole — c'est un ensemble de **6 contraintes architecturales**. Une API est dite "RESTful" si et seulement si elle respecte ces 6 contraintes.

#### La Ressource — Définition Formelle
Une **ressource** est toute information nommée et adressable. C'est une abstraction conceptuelle, pas sa représentation. Une ressource est identifiée par un **URI** (Uniform Resource Identifier). La **représentation** (JSON, XML, HTML) est un snapshot de l'état de la ressource à un instant donné, transféré dans la réponse.

Exemple : `/users/42` identifie la ressource "utilisateur d'ID 42". Le JSON retourné est sa représentation, pas la ressource elle-même.

#### Les 6 Contraintes de Fielding

**1. Client-Server**
Séparation stricte des responsabilités entre le client (interface utilisateur) et le serveur (stockage et logique métier). Améliore la portabilité du client et la scalabilité du serveur.

**2. Stateless (Sans état)**
Chaque requête du client doit contenir **toute l'information nécessaire** pour être comprise par le serveur. Le serveur ne conserve aucun contexte de session entre deux requêtes. La session est maintenue côté client (JWT, cookie). Conséquence : haute scalabilité horizontale (n'importe quel nœud peut traiter n'importe quelle requête).

**3. Cacheable**
Les réponses doivent se qualifier elles-mêmes comme cachéables ou non-cachéables (via les headers HTTP : `Cache-Control`, `ETag`, `Last-Modified`). Un cache valide réduit les interactions client-serveur et améliore les performances.

**4. Uniform Interface**
Contrainte centrale de REST. Composée de 4 sous-contraintes :
- **Identification des ressources** : via URI.
- **Manipulation via représentations** : le client manipule une ressource via sa représentation + les métadonnées nécessaires.
- **Messages auto-descriptifs** : chaque message contient assez d'information pour se décrire (Content-Type, etc.).
- **HATEOAS** (Hypermedia As The Engine Of Application State) : le serveur fournit dans ses réponses les **liens hypertextes** vers les transitions d'état disponibles. Le client ne doit pas avoir connaissance a priori de l'API — il la découvre dynamiquement.

**5. Layered System**
Le client ne peut pas distinguer s'il communique directement avec le serveur final ou avec un intermédiaire (load balancer, CDN, proxy, gateway). Permet d'insérer des couches de sécurité, de cache, ou d'équilibrage de charge de manière transparente.

**6. Code-On-Demand (optionnelle)**
Le serveur peut envoyer du code exécutable au client (JavaScript, applets). Seule contrainte optionnelle de Fielding.

#### Idempotence des Méthodes HTTP

**Définition** : une méthode est **idempotente** si appliquer la même requête N fois produit le même résultat qu'une seule application.

| Méthode | Sûre ? | Idempotente ? |
|---|---|---|
| **GET** | Oui | Oui |
| **HEAD** | Oui | Oui |
| **OPTIONS** | Oui | Oui |
| **PUT** | Non | Oui (remplace la ressource entière) |
| **DELETE** | Non | Oui (supprimer déjà supprimé = même état) |
| **POST** | Non | **Non** (chaque appel crée une nouvelle ressource) |
| **PATCH** | Non | **Non** (en général — dépend de l'implémentation) |

**Sûre** = sans effet de bord côté serveur (lecture seule).

---

### 3.2 POO & Design Patterns

#### Polymorphisme

**Polymorphisme statique (compile-time / overloading)**
- Aussi appelé **surcharge** : plusieurs méthodes portent le même nom mais avec des **signatures différentes** (nombre ou types de paramètres).
- Résolu à la **compilation** — le compilateur sélectionne la bonne méthode selon le type des arguments.
- Python ne supporte pas nativement la surcharge (duck typing) — simulé via `*args`, `**kwargs`, ou `functools.singledispatch`.

**Polymorphisme dynamique (runtime / overriding)**
- Aussi appelé **substitution** (Principe de substitution de Liskov — LSP).
- Une sous-classe **redéfinit** une méthode de sa superclasse. L'appel est résolu **à l'exécution** selon le type réel de l'objet.
- Mécanisme : **vtable** (virtual dispatch table) en langages compilés ; en Python, résolution via le MRO à chaque appel.

#### Abstraction vs Interface

**Classe abstraite** (`ABC` en Python)
- Peut contenir à la fois des **méthodes abstraites** (sans implémentation, `@abstractmethod`) et des méthodes **concrètes** (avec implémentation).
- Peut avoir un **état** (attributs d'instance).
- Exprime une relation **"est-un"** (is-a) : une sous-classe partage la nature de la classe abstraite.
- En Python : `from abc import ABC, abstractmethod`.

**Interface (protocole)**
- Contient **uniquement** des signatures de méthodes (contrat pur) — pas d'implémentation, pas d'état.
- Exprime une relation **"peut-faire"** (can-do) : un objet implémentant l'interface possède une certaine capacité.
- Python : via `typing.Protocol` (duck typing structurel) ou convention de classe abstraite pure.

**Différence fondamentale** : l'abstraction factoriseune implémentation partielle partagée ; l'interface définit un contrat comportemental sans implémentation.

#### Design Pattern — Strategy

**Définition formelle (GoF)**
Définit une **famille d'algorithmes**, encapsule chacun, et les rend interchangeables. Permet de faire varier l'algorithme indépendamment des clients qui l'utilisent. Catégorie : **comportemental**.

**Structure** :
- `Context` : maintient une référence vers une `Strategy` et la délègue.
- `Strategy` (interface/ABC) : déclare le contrat de l'algorithme.
- `ConcreteStrategyA/B` : implémentations concrètes de l'algorithme.

**Avantage clé** : élimine les conditionnelles (`if/elif`) pour sélectionner un comportement — conforme au principe **Ouvert/Fermé** (OCP).

#### Design Pattern — Singleton

**Définition formelle (GoF)**
Garantit qu'une classe n'a qu'**une seule instance** et fournit un point d'accès global à cette instance. Catégorie : **créationnel**.

**Mécanismes d'implémentation en Python**

*Via `__new__`* :
```python
class Singleton:
    _instance = None
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
```

*Via métaclasse* : la métaclasse intercepte `__call__` pour contrôler l'instanciation.

**Critiques académiques** : le Singleton est souvent considéré un **anti-pattern** (difficile à tester, couplage global implicite, problèmes de thread-safety sans verrou). Préférer l'**injection de dépendance**.

---

### 3.3 Modélisation — UML & Design Thinking

#### UML — Agrégation vs Composition

Les deux sont des formes de la relation **"tout-partie"** (whole-part), mais diffèrent sur le cycle de vie.

**Agrégation**
- Relation **"a-un"** faible : la partie peut exister indépendamment du tout.
- Le tout et la partie ont des **cycles de vie indépendants**.
- Notation UML : losange **creux** côté tout.
- Exemple : `Université` agrège `Professeur` — le professeur existait avant l'université et existera après sa dissolution.

**Composition**
- Relation **"est-composé-de"** forte : la partie **ne peut pas exister** sans le tout.
- Si le tout est détruit, les parties sont détruites.
- Le tout est **responsable** du cycle de vie de ses parties.
- Notation UML : losange **plein** côté tout.
- Exemple : `Maison` compose `Pièce` — une pièce n'a pas d'existence indépendante d'une maison.

**Distinction clé** : la composition implique une **appartenance exclusive et un cycle de vie commun**.

#### Design Thinking — Stanford Model (5 phases)

**Définition** : processus de résolution créative de problèmes centré sur l'humain, développé par la d.school de Stanford. Itératif et non-linéaire.

| Phase | Objectif | Livrable |
|---|---|---|
| **1. Empathize** | Comprendre les utilisateurs dans leur contexte réel (interviews, observations) | Insights utilisateurs bruts |
| **2. Define** | Synthétiser les insights en un **Point de Vue** (POV) — énoncé du problème centré utilisateur | "HMW" (How Might We?) statement |
| **3. Ideate** | Générer un maximum de solutions sans jugement (brainstorming, SCAMPER, mind-mapping) | Portfolio d'idées |
| **4. Prototype** | Construire des représentations basse-fidélité rapides pour tester les idées | Prototype testable (wireframe, maquette, jeu de rôle) |
| **5. Test** | Tester les prototypes avec de vrais utilisateurs, recueillir des retours, itérer | Insights pour la prochaine itération |

**Distinction critique** : ce n'est pas un processus **waterfall** séquentiel — on peut revenir en arrière entre phases selon les apprentissages.

---

## PILIER IV — Ops & Git Internals

---

### 4.1 Git — Fonctionnement Interne

#### Le DAG — Directed Acyclic Graph

**Définition formelle**
Git stocke son historique comme un **graphe acyclique dirigé (DAG)** d'objets immuables. Il existe 4 types d'objets Git, tous identifiés par leur **hash SHA-1** (40 caractères hexadécimaux) :

| Type | Description |
|---|---|
| **blob** | Contenu brut d'un fichier (sans métadonnées) |
| **tree** | Répertoire : liste de pointeurs vers des blobs et d'autres trees |
| **commit** | Snapshot : pointeur vers un tree racine + métadonnées (auteur, message) + pointeur(s) vers commit(s) parent(s) |
| **tag** | Pointeur nommé et signé vers un commit |

**Structure d'un commit** :
```
commit <SHA-1>
├── tree <SHA-1>          → état complet du dépôt à cet instant
├── parent <SHA-1>        → commit précédent (0, 1 ou plusieurs pour un merge)
├── author <nom> <date>
├── committer <nom> <date>
└── message
```

**Pourquoi acyclique ?** : un commit pointe vers ses parents mais jamais vers ses enfants. Un commit ne peut pas être son propre ancêtre. Cette propriété garantit l'intégrité et l'immuabilité de l'historique — modifier un commit change son SHA-1, et invalide tous les commits descendants.

**Branches et HEAD** : une branche n'est qu'un fichier texte contenant un SHA-1 (pointeur léger vers un commit). `HEAD` est un pointeur vers la branche courante (ou directement vers un commit en mode "detached HEAD").

---

#### git merge vs git rebase

**git merge**
- Crée un **commit de fusion** (merge commit) avec **deux parents** — préserve la totalité de l'historique des deux branches.
- L'historique reflète fidèlement le **parallélisme réel** du développement.
- L'historique résultant est non-linéaire (graphe avec convergences).
- **Opération non-destructrice** : aucun commit existant n'est modifié.

```
      A---B---C  (feature)
     /         \
D---E---F---G---H  (main, après merge)
```

**git rebase**
- **Réécrit l'historique** : rejoue les commits d'une branche sur une nouvelle base, créant de **nouveaux commits** (nouveaux SHA-1) avec le même contenu différentiel.
- L'historique résultant est **linéaire**.
- Règle d'or (**The Golden Rule of Rebasing**) : **ne jamais rebaser une branche publique/partagée** — la réécriture des SHA-1 orpheline les commits des collaborateurs.

```
      A---B---C  (feature, avant rebase)
     /
D---E---F---G  (main)

Après git rebase main :
D---E---F---G---A'---B'---C'  (feature, commits réécrit)
```

**Différence fondamentale** : `merge` intègre sans réécrire ; `rebase` réécriture pour intégrer. Le choix est une question de politique d'historique : linéarité lisible (rebase) vs traçabilité exacte (merge).

**`git merge --squash`** : condense tous les commits d'une branche en un seul commit avant l'intégration — compromis entre lisibilité et traçabilité.

**Termes académiques associés** : fast-forward merge, three-way merge, SHA-1, content-addressable storage, reflog, cherry-pick, interactive rebase (`-i`), force-push, orphaned commits, garbage collection.

---

### 4.2 CI/CD pour l'IA — Spécificités

#### Data Versioning — DVC

**Définition formelle**
**DVC** (Data Version Control) est un système de versionnement de données et de modèles ML, conçu pour fonctionner en complément de Git. Il résout le problème fondamental de Git avec les **large binary files** (datasets, modèles entraînés) : Git n'est pas conçu pour les stocker efficacement.

**Mécanisme interne**
- DVC stocke dans Git des **fichiers `.dvc`** (métadonnées légères : hash MD5/SHA256 du fichier, taille) en lieu et place des fichiers réels.
- Les fichiers réels sont stockés dans un **remote storage** (S3, GCS, Azure Blob, SFTP, etc.) ou dans un cache local.
- `dvc push` / `dvc pull` synchronisent les fichiers entre le remote et le cache local.
- Le hash du fichier garantit l'**immuabilité et la reproductibilité** : un même hash correspond toujours au même contenu.

**Pipeline de reproduction**
DVC permet de définir des **pipelines DAG** (dvc.yaml) qui trackent les dépendances entre étapes (données → entraînement → évaluation). `dvc repro` réexécute uniquement les étapes dont les dépendances ont changé.

**Articulation Git + DVC** :
```
git commit  → versionne le code + les fichiers .dvc (métadonnées)
dvc push    → versionne les données réelles vers le remote storage
```

---

#### Monitoring du Model Drift

**Définition formelle**
Le **model drift** (dérive du modèle) désigne la dégradation des performances d'un modèle en production due à un changement dans la distribution des données réelles par rapport à la distribution d'entraînement.

**Taxonomie des dérives**

**1. Data Drift (Covariate Shift)**
- La distribution des **features en entrée** $P(X)$ change, mais la relation conditionnelle $P(Y|X)$ reste stable.
- Exemple : les habitudes d'achat changent saisonnièrement — les entrées changent mais la logique de décision reste valide.

**2. Concept Drift**
- La relation conditionnelle $P(Y|X)$ change — le **concept cible** évolue.
- Exemple : la définition de "transaction frauduleuse" évolue avec les nouvelles méthodes de fraude.
- **Sous-types** : drift soudain (abrupt), graduel, récurrent, incrémental.

**3. Label Drift (Prior Probability Shift)**
- La distribution des **labels** $P(Y)$ change.
- Exemple : augmentation du taux réel de fraude dans la population.

**Méthodes de détection statistique**

| Méthode | Type de drift détecté | Principe |
|---|---|---|
| **KS Test** (Kolmogorov-Smirnov) | Data drift (variables continues) | Compare les distributions cumulées |
| **Chi-squared test** | Data drift (variables catégorielles) | Teste l'indépendance des fréquences |
| **PSI** (Population Stability Index) | Data drift | $\sum (Actual\% - Expected\%) \cdot \ln(Actual\%/Expected\%)$ |
| **ADWIN** | Concept drift | Fenêtre glissante adaptative |
| **DDM** (Drift Detection Method) | Concept drift | Détecte l'augmentation du taux d'erreur |

**Seuils PSI** :
- PSI < 0.1 : pas de drift significatif
- 0.1 ≤ PSI < 0.2 : drift modéré (surveillance accrue)
- PSI ≥ 0.2 : drift sévère (réentraînement nécessaire)

**Architecture de monitoring en production**

```
[Modèle en prod] → logs prédictions + features
       ↓
[Feature Store / Data Warehouse]
       ↓
[Monitoring Service] → calcul PSI, KS, métriques métier
       ↓
[Alerting] → Prometheus / Grafana → PagerDuty / Slack
       ↓
[Déclenchement du réentraînement] → MLflow / Kubeflow Pipeline
```

**Termes académiques associés** : covariate shift, concept drift, prior probability shift, statistical process control (SPC), reference window vs detection window, shadow deployment, champion/challenger, A/B testing, canary release, MLflow Model Registry, Evidently AI, Seldon, Arize AI.

---

---

## PILIER V — Python Avancé Complémentaire

---

### 5.0 Système de Modules Python — `__init__.py`, `__main__`, Packages

#### Module vs Package vs Bibliothèque

**Définition formelle**
- **Module** : tout fichier `.py` unique. L'importation d'un module exécute son code de niveau supérieur une seule fois puis le cache dans `sys.modules`.
- **Package** : répertoire contenant un fichier `__init__.py` (package régulier) ou sans ce fichier (namespace package, PEP 420). Permet d'organiser les modules en espace de noms hiérarchique.
- **Bibliothèque / Distribution** : ensemble de packages et modules distribués (via PyPI, `pip install`). Le terme "bibliothèque" est informel — la structure réelle est celle du package.

---

#### `__init__.py` — Rôle et Mécanisme

**Définition formelle**
`__init__.py` est le fichier exécuté par l'interpréteur **au moment de l'import du package**. Sa présence transforme un répertoire en package importable. Son contenu définit ce qui est exposé comme **interface publique** du package.

**Ce qu'il contrôle**

1. **Exécution à l'import** : tout le code de niveau supérieur de `__init__.py` s'exécute à la première importation du package (et uniquement la première — ensuite `sys.modules` sert de cache).

2. **Espace de noms du package** : ce qui est défini ou importé dans `__init__.py` est directement accessible depuis le package.
```python
# mypackage/__init__.py
from .core import MyClass      # MyClass accessible via mypackage.MyClass
from .utils import helper_fn   # helper_fn accessible via mypackage.helper_fn
__version__ = "1.0.0"
```

3. **`__all__`** : liste explicite des noms exportés lors d'un `from package import *`. Sans `__all__`, `import *` importe tout ce qui ne commence pas par `_`.
```python
__all__ = ["MyClass", "helper_fn"]  # contrat d'API publique
```

4. **Import relatif vs absolu** :
   - `from .module import X` : import relatif (depuis le même package) — recommandé intra-package.
   - `from mypackage.module import X` : import absolu — recommandé depuis l'extérieur.

**`__init__.py` vide** : valide — déclare le répertoire comme package sans rien exposer directement. Les sous-modules doivent être importés explicitement.

**Namespace Package (PEP 420 — sans `__init__.py`)** : permet à un package d'être réparti sur plusieurs répertoires. Cas d'usage : plugins distribués séparément mais partageant le même espace de noms (`mycompany.plugin_a`, `mycompany.plugin_b` dans des distributions distinctes).

---

#### `__main__` — Module Principal et Exécution Directe

**`if __name__ == "__main__"`**

**Mécanisme interne** : l'interpréteur Python assigne automatiquement la variable globale `__name__` de chaque module :
- `__name__ == "__main__"` : quand le fichier est **exécuté directement** (`python myscript.py`).
- `__name__ == "mymodule"` (nom du module) : quand le fichier est **importé** par un autre module.

**Pourquoi ce pattern est fondamental** :
```python
def main():
    # logique principale

if __name__ == "__main__":
    main()
```
- **Testabilité** : le code dans `main()` ne s'exécute pas lors de l'import dans les tests.
- **Réutilisabilité** : le module peut être à la fois un script exécutable ET une bibliothèque importable.
- **Séparation des responsabilités** : la logique est dans des fonctions testables, l'entrée du programme dans le guard.

**`__main__.py` dans un package**

Permet d'exécuter un **package entier** comme script :
```
mypackage/
├── __init__.py
├── __main__.py    ← point d'entrée
└── core.py
```
```bash
python -m mypackage   # exécute mypackage/__main__.py
```
Le flag `-m` cherche un module ou un `__main__.py` dans un package et l'exécute avec `__name__ == "__main__"`.

**`python -m` vs exécution directe**
- `python script.py` : ajoute le répertoire du script à `sys.path[0]`.
- `python -m module` : ajoute le **répertoire courant** à `sys.path[0]` — recommandé car évite les problèmes d'imports relatifs.

---

#### Système d'Import — Mécanisme Interne

**`sys.modules` — Cache d'import**
- Premier import : l'interpréteur localise le module (finder), charge le fichier (loader), exécute le code, stocke l'objet module dans `sys.modules[module_name]`.
- Imports suivants : retour direct depuis `sys.modules` sans réexécuter le code.
- Conséquence : un module modifié après import n'est pas rechargé automatiquement (`importlib.reload()` pour forcer).

**Ordre de recherche (sys.path)**
1. Répertoire du script courant (ou répertoire courant si `-m`).
2. `PYTHONPATH` (variable d'environnement).
3. Répertoires standard de l'installation Python (stdlib).
4. Répertoires des packages tiers installés (`site-packages`).

**Import circulaire** : A importe B, B importe A. Python gère partiellement (l'objet module est dans `sys.modules` avant d'être complètement initialisé) mais peut causer des `ImportError` ou des `AttributeError` selon l'ordre d'exécution. Résolution : restructurer les dépendances, ou importer localement (dans la fonction) plutôt qu'au niveau module.

**Termes académiques associés** : `sys.modules`, `sys.path`, finder/loader (importlib), namespace package, `__all__`, `__name__`, `__file__`, `__package__`, `__spec__`, `importlib.import_module()`, `importlib.reload()`, import relatif vs absolu, PEP 328, PEP 420, PEP 451.

---

### 5.1 Asyncio — Programmation Asynchrone

**Définition formelle**
`asyncio` est un framework de programmation **concurrente coopérative** basé sur une **boucle d'événements** (event loop) et des **coroutines**. Contrairement au threading (parallélisme préemptif), asyncio utilise un modèle **single-threaded** où la commutation entre tâches est explicite et coopérative.

**Mécanisme interne — Event Loop**
- L'**event loop** est une boucle infinie qui maintient une file de **callbacks** et d'**I/O events** à traiter.
- À chaque itération, elle interroge le système d'exploitation (via `select`, `epoll` sur Linux, `kqueue` sur macOS) pour savoir quels descripteurs de fichiers sont prêts.
- Les coroutines en attente d'I/O **suspendent** volontairement leur exécution (via `await`), rendant le contrôle à la boucle.
- Quand l'I/O est prête, la boucle reprend la coroutine suspendue.

**Hiérarchie des primitives asyncio**

| Primitive | Rôle |
|---|---|
| **Coroutine** | Fonction `async def` — unité de base, non planifiée tant que non `await`ée |
| **Task** | Coroutine planifiée et exécutée par l'event loop (`asyncio.create_task()`) |
| **Future** | Objet représentant un résultat non encore disponible (bas niveau) |
| **Event Loop** | Ordonnanceur qui exécute les tâches et gère les I/O |

**`async def` vs `def` vs `threading`**
- `def` : fonction synchrone bloquante.
- `async def` : coroutine — retourne un objet coroutine si appelée sans `await`. Ne s'exécute que dans un contexte `await` ou via l'event loop.
- `threading` : vrai parallélisme OS, mais GIL-limité pour CPU-bound. Préemptif.
- `asyncio` : concurrence sur un seul thread. Idéal pour I/O-bound massivement concurrent (serveurs réseau, APIs).

**Patterns courants**
```python
# Exécution concurrente (≠ parallèle) de plusieurs coroutines
results = await asyncio.gather(coro1(), coro2(), coro3())

# Avec timeout
result = await asyncio.wait_for(coro(), timeout=5.0)

# Création de tâche (planification immédiate)
task = asyncio.create_task(coro())
```

**`asyncio` vs `threading` vs `multiprocessing`**

| | asyncio | threading | multiprocessing |
|---|---|---|---|
| **Modèle** | Coopératif / single-thread | Préemptif / multi-thread | Multi-processus |
| **GIL** | Non concerné | Limité par GIL | Contourné (processus séparés) |
| **Cas d'usage** | I/O-bound massivement concurrent | I/O-bound, simplicité | CPU-bound |
| **Overhead** | Minimal (pas de context switch OS) | Moyen (context switch threads) | Élevé (fork, IPC) |

**Termes académiques associés** : event loop, coroutine, cooperative multitasking, non-blocking I/O, `epoll`, `select`, `Future`, `Task`, `await`, `async for`, `async with`, `asyncio.Queue`, backpressure, PEP 492, PEP 3156.

---

### 5.2 Context Managers & Protocole `with`

**Définition formelle**
Un **context manager** est un objet qui définit un **contexte d'exécution** en implémentant les méthodes `__enter__` et `__exit__`. Le mot-clé `with` garantit l'exécution de `__exit__` même en cas d'exception — implémentation du pattern **RAII** (Resource Acquisition Is Initialization).

**Mécanisme interne**
```python
with expr as var:
    body
# équivalent strict à :
mgr = expr
var = mgr.__enter__()
try:
    body
except:
    if not mgr.__exit__(*sys.exc_info()):
        raise
else:
    mgr.__exit__(None, None, None)
```

**Signature de `__exit__`**
```python
def __exit__(self, exc_type, exc_val, exc_tb) -> bool:
    # exc_type/val/tb : informations sur l'exception (None si pas d'exception)
    # Retourner True supprime l'exception
    # Retourner False/None la propage
```

**`contextlib.contextmanager`** — implémentation via générateur :
```python
from contextlib import contextmanager

@contextmanager
def managed_resource():
    resource = acquire()
    try:
        yield resource      # point de suspension = corps du `with`
    finally:
        release(resource)   # garanti même si exception
```

**Termes académiques associés** : RAII, resource management, exception safety, `__enter__`/`__exit__`, `contextlib`, `suppress`, `ExitStack`.

---

### 5.3 Méthodes Spéciales (Dunder Methods)

**Définition formelle**
Les **méthodes dunders** (double underscore) sont des méthodes spéciales invoquées implicitement par des opérateurs ou des fonctions built-in. Elles constituent le **protocole d'objets** Python et permettent d'intégrer des classes utilisateur dans le modèle de données Python.

**Catégories fondamentales**

**Représentation**
- `__repr__` : représentation non-ambiguë, orientée développeur (doit idéalement permettre de recréer l'objet). Appelé par `repr()`.
- `__str__` : représentation lisible, orientée utilisateur. Appelé par `str()` et `print()`. Fallback vers `__repr__` si absent.

**Comparaison & Hashabilité**
- `__eq__` : opérateur `==`. Si défini sans `__hash__`, l'objet devient **non-hashable** (Python met `__hash__` à `None`).
- `__hash__` : valeur de hachage pour utilisation dans `dict` et `set`. **Contrat fondamental** : si `a == b` alors `hash(a) == hash(b)`.
- `__lt__`, `__le__`, `__gt__`, `__ge__` : opérateurs de comparaison. `functools.total_ordering` peut dériver les 4 depuis `__eq__` et `__lt__`.

**Cycle de vie**
- `__init__` : initialisation de l'instance (après sa création).
- `__new__` : **création** de l'instance — appelé avant `__init__`, retourne l'instance. Utilisé pour les Singletons, les classes immuables (sous-classer `int`, `str`).
- `__del__` : finalizer — appelé quand `ob_refcnt == 0`. **Non garanti** d'être appelé (cycles, interpréteur en cours de fermeture).

**Protocoles conteneurs**
- `__len__`, `__getitem__`, `__setitem__`, `__delitem__`, `__contains__` (`in`), `__iter__`, `__next__`.

**Descripteurs** (protocole avancé)
- `__get__`, `__set__`, `__delete__` : permettent de contrôler l'accès aux attributs d'une classe. Base de `property`, `classmethod`, `staticmethod`.
- **Data descriptor** : implémente `__set__` ou `__delete__` — priorité sur le `__dict__` de l'instance.
- **Non-data descriptor** : uniquement `__get__` — le `__dict__` de l'instance a la priorité.

**Termes académiques associés** : data model, descriptor protocol, `__slots__` (optimisation mémoire, désactive `__dict__`), `__init_subclass__`, `__class_getitem__`, metaclass.

---

### 5.4 Structures de Données Python — Complexités

**Complexités asymptotiques des structures built-in**

| Opération | `list` | `dict` | `set` | `deque` |
|---|---|---|---|---|
| Accès par index | O(1) | — | — | O(n) |
| Recherche (`in`) | O(n) | O(1) amortized | O(1) amortized | O(n) |
| Insertion en fin | O(1) amortized | O(1) amortized | O(1) amortized | O(1) |
| Insertion au début | O(n) | — | — | O(1) |
| Suppression par valeur | O(n) | O(1) amortized | O(1) amortized | O(n) |
| Tri | O(n log n) | — | — | — |

**Implémentation interne**
- **`list`** : tableau dynamique (dynamic array) en C. Redimensionnement par **sur-allocation** (over-allocation) : quand la capacité est atteinte, Python alloue ~1.125x la taille actuelle pour amortir l'insertion.
- **`dict`** (Python 3.7+) : table de hachage **à insertion ordonnée** (preserve insertion order, garanti depuis 3.7). Adressage ouvert avec **sondage quadratique**. Facteur de charge : resize à 2/3.
- **`set`** : table de hachage sans valeurs associées.
- **`deque`** : liste doublement chaînée de blocs fixes — O(1) aux deux extrémités.

**`heapq`** — Min-heap : arbre binaire complet stocké dans un tableau. `heappush`/`heappop` en O(log n).

**Termes académiques associés** : dynamic array, amortized complexity, hash table, open addressing, separate chaining, load factor, hash collision, `__hash__`, `__eq__`.

---

## PILIER VI — Fondamentaux ML & Mathématiques

---

### 6.1 Optimisation — Descente de Gradient

**Définition formelle**
La **descente de gradient** est un algorithme d'optimisation du premier ordre qui minimise une fonction objectif $\mathcal{L}(\theta)$ en itérant dans la direction opposée au gradient :

$$\theta_{t+1} = \theta_t - \eta \nabla_\theta \mathcal{L}(\theta_t)$$

où $\eta$ est le **taux d'apprentissage** (learning rate).

**Variants**

| Algorithme | Gradient calculé sur | Avantages | Limitations |
|---|---|---|---|
| **Batch GD** | Tout le dataset | Gradient exact, convergence stable | Trop lent sur grands datasets |
| **SGD** (Stochastic) | 1 exemple | Rapide, bruit utile pour échapper aux minima locaux | Gradient bruité, oscillations |
| **Mini-batch GD** | Batch de taille B | Compromis efficacité/stabilité | Choix de B critique |

**Optimiseurs Adaptatifs**

**Adam** (Adaptive Moment Estimation) :
$$m_t = \beta_1 m_{t-1} + (1-\beta_1) g_t \quad \text{(1er moment = moyenne)}$$
$$v_t = \beta_2 v_{t-1} + (1-\beta_2) g_t^2 \quad \text{(2ème moment = variance non-centrée)}$$
$$\hat{m}_t = \frac{m_t}{1-\beta_1^t}, \quad \hat{v}_t = \frac{v_t}{1-\beta_2^t} \quad \text{(correction du biais de démarrage)}$$
$$\theta_{t+1} = \theta_t - \frac{\eta}{\sqrt{\hat{v}_t} + \epsilon} \hat{m}_t$$

- $\beta_1 = 0.9$, $\beta_2 = 0.999$, $\epsilon = 10^{-8}$ par défaut.
- Adapte le learning rate **par paramètre** en fonction de l'historique des gradients.
- **RMSprop** : uniquement le 2ème moment (pas de momentum). **AdaGrad** : accumule les gradients passés (peut sur-décroître le LR).

**Problèmes de convergence**
- **Vanishing gradient** : gradients → 0 dans les couches profondes (tanh, sigmoid saturés). Solutions : ReLU, BatchNorm, connexions résiduelles.
- **Exploding gradient** : gradients → ∞. Solutions : gradient clipping.
- **Saddle points** : gradient = 0 mais pas un minimum — problème en haute dimension (plus fréquent que les vrais minima locaux).

---

### 6.2 Rétropropagation (Backpropagation)

**Définition formelle**
La **rétropropagation** est l'application de la **règle de dérivation en chaîne** (chain rule) pour calculer efficacement le gradient de la fonction de perte par rapport à chaque paramètre du réseau.

**Mécanisme — Forward Pass / Backward Pass**

*Forward pass* : calcul de la perte $\mathcal{L}$ en propageant l'entrée couche par couche. Chaque opération intermédiaire est stockée dans un **graphe de calcul** (computation graph).

*Backward pass* : à partir de $\frac{\partial \mathcal{L}}{\partial \text{output}}$, application récursive de la chain rule :
$$\frac{\partial \mathcal{L}}{\partial W^{(l)}} = \frac{\partial \mathcal{L}}{\partial a^{(l)}} \cdot \frac{\partial a^{(l)}}{\partial z^{(l)}} \cdot \frac{\partial z^{(l)}}{\partial W^{(l)}}$$

**Autodiff** : les frameworks modernes (PyTorch, JAX) implémentent la **différentiation automatique** (autograd) — construction dynamique du graphe de calcul à chaque forward pass, gradient calculé par traversée inverse du graphe.

**Termes académiques associés** : chain rule, computation graph, dynamic vs static graph (PyTorch vs TensorFlow 1.x), `backward()`, `grad_fn`, `requires_grad`, Jacobian, Hessian.

---

### 6.3 Fonctions d'Activation

**Définition** : fonctions non-linéaires appliquées à la sortie d'un neurone. Sans elles, un réseau profond serait équivalent à une régression linéaire (composition de transformations linéaires = transformation linéaire).

| Fonction | Formule | Range | Problème connu |
|---|---|---|---|
| **Sigmoid** | $\sigma(x) = \frac{1}{1+e^{-x}}$ | (0, 1) | Vanishing gradient, non zero-centered |
| **Tanh** | $\tanh(x) = \frac{e^x - e^{-x}}{e^x + e^{-x}}$ | (-1, 1) | Vanishing gradient (atténué) |
| **ReLU** | $\max(0, x)$ | [0, ∞) | **Dying ReLU** : neurones à 0 permanent si gradient toujours négatif |
| **Leaky ReLU** | $\max(\alpha x, x)$, $\alpha \approx 0.01$ | (-∞, ∞) | Corrige le Dying ReLU |
| **GELU** | $x \cdot \Phi(x)$ (CDF normale) | (-∞, ∞) | Standard dans les Transformers (BERT, GPT) |
| **Softmax** | $\frac{e^{x_i}}{\sum_j e^{x_j}}$ | (0,1), somme=1 | Utilisé en dernière couche multi-classe uniquement |

**Dying ReLU** : si le gradient est négatif pour un neurone à chaque batch, sa sortie reste 0, son gradient reste 0, ses poids ne se mettent plus à jour → le neurone est "mort".

---

### 6.4 Fonctions de Perte (Loss Functions)

**Régression**
- **MSE** (Mean Squared Error) : $\frac{1}{n}\sum(y_i - \hat{y}_i)^2$. Pénalise fortement les outliers (terme au carré). Différentiable partout.
- **MAE** (Mean Absolute Error) : $\frac{1}{n}\sum|y_i - \hat{y}_i|$. Robuste aux outliers, non-différentiable en 0.
- **Huber Loss** : MSE pour petites erreurs, MAE pour grandes — combine les avantages des deux.

**Classification**
- **Binary Cross-Entropy** : $-\frac{1}{n}\sum [y_i \log(\hat{p}_i) + (1-y_i)\log(1-\hat{p}_i)]$. Pour classification binaire avec sigmoid.
- **Categorical Cross-Entropy** : $-\sum y_i \log(\hat{p}_i)$. Pour multi-classe avec softmax.
- **Focal Loss** : pondère les exemples difficiles, résout le déséquilibre de classes.

**Pourquoi Cross-Entropy et non MSE pour la classification ?**
MSE avec sigmoid produit des **gradients qui s'annulent** quand le modèle est confiant mais faux (plateau de la sigmoïde). La cross-entropy, dérivée de la **log-vraisemblance** (MLE), maintient un gradient proportionnel à l'erreur même en zone saturée.

---

### 6.5 Régularisation & Biais-Variance

**Biais-Variance Decomposition**

$$\mathbb{E}[(y - \hat{f}(x))^2] = \text{Bias}[\hat{f}(x)]^2 + \text{Var}[\hat{f}(x)] + \sigma^2_\epsilon$$

- **Biais** : erreur due aux hypothèses simplificatrices du modèle (underfitting → biais élevé).
- **Variance** : sensibilité aux fluctuations du dataset d'entraînement (overfitting → variance élevée).
- **Bruit irréductible** ($\sigma^2_\epsilon$) : bruit inhérent aux données, incompressible.

**Regularisation L1 (Lasso)**
$$\mathcal{L}_{total} = \mathcal{L} + \lambda \sum_j |w_j|$$
- Pénalise la norme L1 des poids. Induit la **sparsité** : certains poids convergent exactement vers 0 (sélection de features implicite). Non-différentiable en 0.

**Régularisation L2 (Ridge / Weight Decay)**
$$\mathcal{L}_{total} = \mathcal{L} + \lambda \sum_j w_j^2$$
- Pénalise la norme L2 des poids. Décourage les grands poids sans les annuler. Différentiable partout. Équivalent à une **prior gaussienne** sur les poids (MAP estimation).

**Dropout**
- À chaque forward pass d'entraînement, chaque neurone est désactivé avec probabilité $p$ (typiquement 0.5).
- Force le réseau à **ne pas dépendre de neurones spécifiques** — apprentissage de représentations redondantes et robustes.
- Équivalent à entraîner un ensemble de $2^n$ réseaux partagés avec bagging implicite.
- À l'inférence : tous les neurones actifs, poids multipliés par $(1-p)$ (ou scaling à l'entraînement — "inverted dropout").

**Batch Normalization**
- Normalise l'activation de chaque couche pour avoir moyenne ≈ 0 et variance ≈ 1 par batch.
- Réduit l'**internal covariate shift** — stabilise l'entraînement, permet des learning rates plus élevés.
- Introduit deux paramètres apprenables $\gamma$ et $\beta$ pour restaurer la capacité d'expression.

**Termes académiques associés** : overfitting, underfitting, cross-validation, k-fold, bias-variance tradeoff, L1/L2 norm, elastic net, early stopping, data augmentation, weight decay, Layer Norm (Transformers), Group Norm.

---

### 6.6 Algorithmes ML Classiques — Mécanismes

**Régression Logistique**
Modèle linéaire généralisé : $P(y=1|x) = \sigma(w^T x + b)$. Entraînement par MLE (maximisation de la log-vraisemblance) ↔ minimisation de la cross-entropy. **Frontière de décision linéaire**.

**SVM (Support Vector Machine)**
Cherche l'**hyperplan de marge maximale** séparant les classes. La marge est $\frac{2}{\|w\|}$. Seuls les **vecteurs de support** (points les plus proches de l'hyperplan) déterminent la solution.
- **Kernel trick** : $\langle \phi(x_i), \phi(x_j) \rangle = K(x_i, x_j)$ — opère dans un espace de grande dimension sans le calculer explicitement (RBF, polynomial).

**Arbres de Décision**
Partitionnement récursif de l'espace des features selon un critère de **pureté** :
- **Gini impurity** : $1 - \sum_k p_k^2$ — probabilité qu'un élément choisi au hasard soit mal classé.
- **Entropie** : $-\sum_k p_k \log_2(p_k)$ — mesure d'information (Information Gain = réduction d'entropie).

**Random Forest**
Ensemble de $T$ arbres de décision entraînés sur des **bootstrap samples** (bagging) avec **feature subsampling** (sélection aléatoire d'un sous-ensemble de features à chaque nœud). Réduit la variance sans augmenter le biais. Prédiction par **vote majoritaire** (classification) ou **moyenne** (régression).

**Gradient Boosting (XGBoost)**
Ensemble **additif** d'arbres : chaque nouvel arbre corrige les **résidus** du modèle précédent. Entraînement séquentiel (ne peut pas être parallélisé sur les arbres). XGBoost ajoute : régularisation L1/L2 sur les poids des feuilles, second ordre Taylor pour l'optimisation de la perte.

**k-Means Clustering**
Algorithme EM (Expectation-Maximization) :
1. **E-step** : assigner chaque point au centroïde le plus proche.
2. **M-step** : recalculer les centroïdes comme moyenne des points assignés.
Convergence vers un minimum local — sensible à l'initialisation (k-means++ améliore l'initialisation).

**PCA (Principal Component Analysis)**
Décomposition en **composantes principales** : projette les données dans un sous-espace de dimension réduite qui maximise la variance expliquée. Mécanisme : SVD (Singular Value Decomposition) de la matrice de covariance centrée. Les composantes principales sont les vecteurs propres associés aux plus grandes valeurs propres.

---

## PILIER VII — Bases de Données & SQL

---

### 7.1 Propriétés ACID

**Définition** : les propriétés **ACID** garantissent la fiabilité des transactions dans un SGBD.

**A — Atomicité**
Une transaction est une unité indivisible : soit **toutes** les opérations réussissent (commit), soit **aucune** n'est appliquée (rollback). Implémenté via des **journaux de transaction** (Write-Ahead Log — WAL) : les modifications sont d'abord écrites dans le journal avant d'être appliquées aux données.

**C — Cohérence**
La transaction amène la base d'un état **valide** vers un autre état **valide** selon les contraintes définies (clés étrangères, contraintes NOT NULL, CHECK, triggers). Responsabilité partagée entre le SGBD et l'application.

**I — Isolation**
Les transactions concurrentes sont exécutées comme si elles étaient **séquentielles**. Niveaux d'isolation (du plus permissif au plus strict) :

| Niveau | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| **Read Uncommitted** | Possible | Possible | Possible |
| **Read Committed** | Impossible | Possible | Possible |
| **Repeatable Read** | Impossible | Impossible | Possible |
| **Serializable** | Impossible | Impossible | Impossible |

- **Dirty Read** : lire des données non encore committées par une autre transaction.
- **Non-Repeatable Read** : une ligne lue deux fois dans la même transaction retourne des valeurs différentes (modifiée entre-temps).
- **Phantom Read** : une requête retourne des lignes différentes deux fois (lignes insérées/supprimées entre-temps).

**D — Durabilité**
Une fois le commit effectué, les modifications sont **persistées** même en cas de crash. Implémenté via le WAL : le journal sur disque est fsync'd avant le commit.

---

### 7.2 Index — Mécanismes Internes

**Définition** : structure de données auxiliaire qui accélère la recherche au prix d'espace disque supplémentaire et d'un overhead sur les écritures.

**B-Tree Index (standard)**
- Structure arborescente **balancée** où chaque nœud contient des clés triées et des pointeurs vers les nœuds enfants.
- Hauteur : $O(\log_b n)$ où $b$ = facteur de branchement (typiquement 100-1000).
- **B+-Tree** (standard dans les SGBD) : les données sont uniquement dans les **feuilles**, qui sont chaînées en liste doublement liée pour les **range scans** efficaces.
- Supporte : `=`, `<`, `>`, `BETWEEN`, `ORDER BY`, `LIKE 'prefix%'`.

**Hash Index**
- Table de hachage — lookup en O(1).
- Uniquement pour `=` — pas de range scan.
- Utilisé par défaut dans PostgreSQL pour les jointures de hachage (hash join).

**Composite Index (Index Multi-colonnes)**
- Index sur `(col1, col2, col3)` — utilisable pour les prédicats sur `col1`, `(col1, col2)`, `(col1, col2, col3)` mais **pas** uniquement sur `col2` ou `col3`.
- Règle du **préfixe le plus à gauche** (leftmost prefix rule).

**Index Couvrant (Covering Index)**
- Index qui contient toutes les colonnes nécessaires à une requête → le SGBD peut satisfaire la requête depuis l'index sans lire la table principale (**index-only scan**).

---

### 7.3 SQL Avancé

**JOIN Types**

| JOIN | Comportement |
|---|---|
| **INNER JOIN** | Lignes correspondant dans les deux tables |
| **LEFT JOIN** | Toutes les lignes de gauche + correspondances de droite (NULL si absent) |
| **RIGHT JOIN** | Toutes les lignes de droite + correspondances de gauche |
| **FULL OUTER JOIN** | Toutes les lignes des deux tables |
| **CROSS JOIN** | Produit cartésien |
| **SELF JOIN** | Join d'une table avec elle-même |

**Algorithmes de Join**
- **Nested Loop Join** : O(n*m) — efficace si une table est petite et l'autre indexée.
- **Hash Join** : O(n+m) — build une hash table sur la petite table, probe avec la grande. Optimal pour grandes tables non indexées.
- **Merge Join** : O(n log n + m log m) — nécessite les deux tables triées.

**Window Functions**
Appliquent un calcul sur un **ensemble de lignes en relation** (la "fenêtre") sans réduire le nombre de lignes (contrairement à `GROUP BY`).

```sql
SELECT
    employee_id,
    salary,
    department,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) as rank,
    SUM(salary) OVER (PARTITION BY department) as dept_total,
    AVG(salary) OVER (ORDER BY hire_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) as rolling_avg
FROM employees;
```

- `PARTITION BY` : définit les groupes (équivalent GROUP BY mais sans agréger).
- `ORDER BY` dans `OVER` : définit l'ordre dans la fenêtre.
- **Frame** : `ROWS/RANGE BETWEEN ... AND ...` — sous-ensemble de la partition.
- Fonctions : `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `LEAD()`, `LAG()`, `FIRST_VALUE()`, `LAST_VALUE()`.

**Différence RANK vs DENSE_RANK vs ROW_NUMBER** :
- `ROW_NUMBER()` : numéro unique par ligne, pas de doublons.
- `RANK()` : les ex-aequo reçoivent le même rang, le rang suivant est sauté (1, 1, 3).
- `DENSE_RANK()` : les ex-aequo reçoivent le même rang, sans saut (1, 1, 2).

---

### 7.4 CAP Theorem & Bases NoSQL

**Théorème CAP (Brewer, 2000)**
Dans un système distribué, il est **impossible** de garantir simultanément les trois propriétés suivantes :

- **C — Cohérence** (Consistency) : toute lecture reçoit la donnée la plus récente ou une erreur.
- **A — Disponibilité** (Availability) : toute requête reçoit une réponse (pas d'erreur), mais potentiellement pas la plus récente.
- **P — Tolérance aux partitions** (Partition Tolerance) : le système continue à fonctionner malgré des pertes/retards de communication entre nœuds.

**Dans la pratique** : la partition réseau est inévitable dans tout système distribué → le choix se fait entre **CP** (cohérence au détriment de la disponibilité) et **AP** (disponibilité au détriment de la cohérence).

| Type | Exemples | Choix CAP |
|---|---|---|
| **SGBD relationnels** | PostgreSQL, MySQL | CP |
| **Document** | MongoDB, CouchDB | AP (configurable) |
| **Clé-valeur** | Redis, DynamoDB | AP / CP (configurable) |
| **Colonnes larges** | Cassandra, HBase | AP (Cassandra) / CP (HBase) |
| **Graph** | Neo4j | CP |

**BASE vs ACID** (NoSQL)
- **BA**sically Available : disponibilité prioritaire.
- **S**oft state : l'état peut changer même sans entrée (propagation de mises à jour).
- **E**ventual consistency : la cohérence est atteinte **à terme** si aucune nouvelle mise à jour.

---

## PILIER VIII — Principes SOLID & Clean Architecture

---

### 8.1 Principes SOLID (Robert C. Martin)

**S — Single Responsibility Principle (SRP)**
Une classe doit avoir **une seule raison de changer** — elle ne doit avoir qu'une seule responsabilité (qu'un seul acteur qui la fait évoluer). Une "responsabilité" = un ensemble cohérent de fonctionnalités liées à un même acteur métier.

**O — Open/Closed Principle (OCP)**
Les entités logicielles doivent être **ouvertes à l'extension** mais **fermées à la modification**. Ajouter un nouveau comportement ne doit pas nécessiter de modifier le code existant — mais d'ajouter du nouveau code (extension par héritage, composition, ou injection).

**L — Liskov Substitution Principle (LSP)**
Si $S$ est un sous-type de $T$, tout objet de type $T$ peut être remplacé par un objet de type $S$ **sans altérer les propriétés du programme**. Formellement : les préconditions ne peuvent qu'être affaiblies, les postconditions ne peuvent qu'être renforcées, les invariants doivent être préservés.

**I — Interface Segregation Principle (ISP)**
Les clients ne doivent pas être forcés de **dépendre d'interfaces** qu'ils n'utilisent pas. Préférer plusieurs interfaces spécialisées à une unique interface générale (fat interface).

**D — Dependency Inversion Principle (DIP)**
Les modules de haut niveau ne doivent pas dépendre des modules de bas niveau. **Les deux doivent dépendre d'abstractions.** Les abstractions ne doivent pas dépendre des détails — les détails doivent dépendre des abstractions.

**Mécanisme** : injection de dépendance (Dependency Injection) — les collaborateurs sont fournis à l'objet (par constructeur, setter, ou méthode) plutôt que créés en interne.

---

### 8.2 Clean Architecture (Hexagonale / Ports & Adapters)

**Principe central** : les **règles métier** (domain layer) sont totalement **indépendantes** des frameworks, bases de données, interfaces utilisateur, et services externes. La dépendance ne va que vers l'intérieur.

**Couches (de l'intérieur vers l'extérieur)**

```
┌─────────────────────────────────────────┐
│              Infrastructure             │  ← DB, API externe, UI, Frameworks
│  ┌───────────────────────────────────┐  │
│  │          Application              │  │  ← Use Cases, Orchestration
│  │  ┌─────────────────────────────┐  │  │
│  │  │          Domain             │  │  │  ← Entities, Value Objects, Domain Logic
│  │  └─────────────────────────────┘  │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

- **Domain** : entités métier, règles invariantes, value objects — aucune dépendance externe.
- **Application** : use cases (orchestration des entités), ports (interfaces abstraites vers l'infrastructure).
- **Infrastructure** : adapters (implémentations concrètes des ports) — DB repositories, HTTP clients, message brokers.

**Règle de dépendance** : les imports ne vont que vers l'intérieur. L'infrastructure dépend de l'application ; l'application dépend du domain ; le domain ne dépend de rien.

**Ports & Adapters** : les "ports" sont les interfaces (abstractions) définis dans la couche application. Les "adapters" sont les implémentations concrètes dans l'infrastructure. **Testabilité** : en tests unitaires, les adapters réels sont remplacés par des fakes/stubs.

---

## PILIER IX — Infrastructure & Conteneurisation

---

### 9.1 Docker — Mécanismes Internes

**Définition formelle**
Docker est une plateforme de **conteneurisation** qui isole des processus applicatifs en utilisant des primitives du noyau Linux : **namespaces** et **cgroups**. Un conteneur n'est **pas** une VM — il partage le noyau de l'hôte.

**Namespaces (isolation)**
Chaque conteneur a ses propres namespaces :
- **PID** : isolation de la vue des processus (le conteneur ne voit que ses processus).
- **NET** : interface réseau propre (stack TCP/IP virtuelle).
- **MNT** : système de fichiers propre (points de montage isolés).
- **UTS** : hostname et domainname propres.
- **IPC** : isolation de la mémoire partagée et des sémaphores.
- **USER** : mapping d'UIDs (rootless containers).

**cgroups (Control Groups) — limitation des ressources**
Permettent de limiter, mesurer et prioriser les ressources (CPU, mémoire, I/O, réseau) allouées à un groupe de processus. `--memory=512m` ou `--cpus=2` dans Docker correspondent à des règles cgroup.

**Union Filesystem — Layers**
Les images Docker sont composées de **couches immuables** (layers) empilées via un **union filesystem** (OverlayFS par défaut). Chaque instruction `RUN`, `COPY`, `ADD` dans un Dockerfile crée un nouveau layer. Le **container layer** (R/W) est ajouté au-dessus à l'exécution. Le partage de layers entre images réduit l'espace disque.

**Dockerfile best practices (mécanisme)**
- Ordonner les instructions du moins fréquemment changé au plus fréquemment changé → **cache hit** maximal.
- Multi-stage builds : séparer l'environnement de build de l'image finale → images de production minimales.

---

### 9.2 Kubernetes — Concepts Fondamentaux

**Architecture Kubernetes**

**Control Plane** (cerveau du cluster) :
- **API Server** : point d'entrée unique — toutes les requêtes passent par lui. Stockage dans etcd.
- **etcd** : store clé-valeur distribué — source de vérité de l'état du cluster.
- **Scheduler** : décide sur quel **node** placer un nouveau Pod (selon ressources, affinités, taints).
- **Controller Manager** : boucle de contrôle qui reconcilie l'état désiré avec l'état réel (ReplicaSet controller, Deployment controller, etc.).

**Worker Nodes** :
- **kubelet** : agent sur chaque node — s'assure que les containers définis dans les PodSpecs s'exécutent.
- **kube-proxy** : implémente les règles de réseau (iptables/IPVS) pour les Services.
- **Container Runtime** : Docker, containerd, CRI-O.

**Primitives clés**

| Ressource | Rôle |
|---|---|
| **Pod** | Unité atomique de déploiement — un ou plusieurs containers partageant un namespace réseau et de stockage |
| **Deployment** | Déclare l'état désiré (N replicas d'un Pod) — gère les rolling updates |
| **Service** | Abstraction réseau stable vers un groupe de Pods (ClusterIP, NodePort, LoadBalancer) |
| **ConfigMap / Secret** | Découplage configuration/secrets du code |
| **Ingress** | Règles HTTP/HTTPS routing vers les Services (L7 load balancing) |
| **PersistentVolume (PV/PVC)** | Abstraction du stockage persistant |
| **HorizontalPodAutoscaler** | Auto-scaling basé sur métriques (CPU, RAM, custom) |

**Boucle de réconciliation** : le principe fondamental de Kubernetes est le **desired state vs observed state**. Chaque controller surveille en permanence la différence et agit pour la résorber.

---

## PILIER X — Structures de Données & Algorithmes

---

### 10.1 Notations de Complexité

**Big O — Définition formelle**
$f(n) = O(g(n))$ si $\exists c > 0, n_0 > 0$ tels que $\forall n \geq n_0 : f(n) \leq c \cdot g(n)$. Big O décrit la **borne supérieure asymptotique** (worst case).

**Notations associées**
- $\Omega(g(n))$ : borne inférieure (best case).
- $\Theta(g(n))$ : borne exacte (si O et Ω coïncident).

**Hiérarchie des complexités** (du plus lent au plus rapide) :
$$O(1) < O(\log n) < O(n) < O(n \log n) < O(n^2) < O(2^n) < O(n!)$$

**Complexité amortie** : coût moyen sur une séquence d'opérations. L'insertion dans un `list` Python est O(1) amorti bien qu'occasionnellement O(n) (lors du resize).

---

### 10.2 Structures de Données Fondamentales

**Tableau (Array)**
- Accès O(1), insertion/suppression O(n).
- Données contiguës en mémoire → cache-friendly.

**Liste Chaînée (Linked List)**
- Accès O(n), insertion/suppression O(1) (si pointeur sur nœud).
- Doublement chaînée : navigation bidirectionnelle. Base des deques.

**Arbre Binaire de Recherche (BST)**
- Propriété : nœud gauche < nœud < nœud droit.
- Recherche/insertion/suppression : O(h) où h = hauteur. O(log n) si équilibré, O(n) si dégénéré.
- **AVL / Red-Black Tree** : BST auto-équilibré → O(log n) garanti.

**Heap (Tas)**
- Arbre binaire complet stocké dans un tableau.
- **Min-heap** : parent ≤ enfants. `heap[0]` = minimum.
- `heappush` / `heappop` : O(log n). `heapify` : O(n).
- Usage : priority queues, Dijkstra, tri par tas.

**Table de Hachage**
- Tableau + fonction de hachage. Résolution des collisions : **chaînage séparé** (liste) ou **adressage ouvert** (probing linéaire/quadratique/double hashing).
- O(1) amorti pour insert/search/delete.

**Graphe**
- **Représentations** : matrice d'adjacence O(V²) espace, liste d'adjacence O(V+E) espace.
- **DFS** (Depth-First Search) : O(V+E) — détection de cycles, tri topologique.
- **BFS** (Breadth-First Search) : O(V+E) — plus court chemin en graphe non-pondéré.
- **Dijkstra** : O((V+E) log V) avec heap — plus court chemin en graphe pondéré (poids positifs).

---

### 10.3 Algorithmes de Tri

| Algorithme | Best | Average | Worst | Space | Stable ? |
|---|---|---|---|---|---|
| **Bubble Sort** | O(n) | O(n²) | O(n²) | O(1) | Oui |
| **Insertion Sort** | O(n) | O(n²) | O(n²) | O(1) | Oui |
| **Merge Sort** | O(n log n) | O(n log n) | O(n log n) | O(n) | Oui |
| **Quick Sort** | O(n log n) | O(n log n) | O(n²) | O(log n) | Non |
| **Heap Sort** | O(n log n) | O(n log n) | O(n log n) | O(1) | Non |
| **Tim Sort** (Python) | O(n) | O(n log n) | O(n log n) | O(n) | Oui |

**Tim Sort** : algorithme hybride (Merge Sort + Insertion Sort) utilisé dans Python (`list.sort()`, `sorted()`). Exploite les **runs** (sous-séquences déjà triées) naturelles des données réelles.

**Borne inférieure** : tout algorithme de tri par comparaison est en $\Omega(n \log n)$ — preuve par le modèle de l'arbre de décision (une feuille par permutation possible).

---

### 10.4 Paradigmes Algorithmiques

**Diviser pour Régner (Divide & Conquer)**
Divise le problème en sous-problèmes indépendants, résout récursivement, combine. Analyse via le **théorème maître** :
$$T(n) = aT(n/b) + f(n)$$

**Programmation Dynamique (DP)**
Décomposition en sous-problèmes **chevauchants** avec mémoïsation. Deux approches :
- **Top-down (memoization)** : récursion + cache.
- **Bottom-up (tabulation)** : remplissage itératif d'une table.
Exemples classiques : Fibonacci, Longest Common Subsequence, Knapsack, Edit Distance.

**Greedy (Glouton)**
Choisit à chaque étape l'option localement optimale sans revenir en arrière. Correct uniquement si la structure du problème exhibe la **propriété du choix glouton** et la **sous-structure optimale** (conditions nécessaires). Exemples : Dijkstra, Huffman coding, algorithme de Kruskal (MST).

---

## PILIER XI — Réseau & Sécurité (Bases)

---

### 11.1 HTTP/HTTPS & TLS

**HTTP — Protocole**
Protocol applicatif **sans état** (stateless) en texte clair. HTTP/1.1 : connexions persistantes (keep-alive), pipelining. HTTP/2 : multiplexage de flux sur une seule connexion TCP, compression des headers (HPACK), server push. HTTP/3 : basé sur QUIC (UDP) — élimine le head-of-line blocking de TCP.

**TLS (Transport Layer Security)**
Protocole cryptographique qui sécurise la communication au-dessus de TCP. TLS 1.3 (standard actuel) :

1. **Handshake** (1-RTT ou 0-RTT en reprise de session) :
   - Client envoie `ClientHello` : versions TLS supportées, **cipher suites**, aléatoire client.
   - Serveur envoie `ServerHello` : cipher suite choisie, aléatoire serveur, **certificat** (clé publique + identité signée par une CA).
   - **Échange de clés** via **ECDHE** (Elliptic Curve Diffie-Hellman Ephemeral) — génère un **shared secret** sans transmettre de clé privée.
   - Dérivation de clés symétriques via HKDF.
2. **Transport** : chiffrement symétrique (AES-GCM, ChaCha20-Poly1305) avec les clés dérivées.

**Propriétés TLS** : confidentialité (chiffrement), intégrité (MAC), authenticité (certificat), **forward secrecy** (ECDHE — compromission de la clé privée à long terme ne compromet pas les sessions passées).

---

### 11.2 Authentification — JWT & OAuth2

**JWT (JSON Web Token)**
Token auto-porteur composé de 3 parties encodées en Base64URL, séparées par `.` :
- **Header** : algorithme de signature (`RS256`, `HS256`) + type.
- **Payload** : claims (informations : `sub`, `iat`, `exp`, `role`, etc.).
- **Signature** : `HMAC(header.payload, secret)` ou `RSA(header.payload, private_key)`.

**Vérification** : le serveur recalcule la signature et la compare — aucune base de données consultée (stateless). L'expiration (`exp`) est vérifiée localement.

**Différence HS256 vs RS256** :
- **HS256** : secret symétrique partagé — tous les vérificateurs connaissent le secret. Risque si multi-service.
- **RS256** : clé privée pour signer (serveur d'auth uniquement), clé publique pour vérifier (tout service) — plus sécurisé en architecture microservices.

**OAuth2 — Flows principaux**
- **Authorization Code + PKCE** : flux pour les applications web/mobile — l'accès aux ressources protégées passe par un `authorization code` à courte durée de vie, jamais directement par le token.
- **Client Credentials** : communication M2M (machine-to-machine) — pas d'utilisateur impliqué.

---

### 11.3 Sécurité — OWASP Top 10 (Concepts clés)

**SQL Injection**
Injection de code SQL dans une entrée utilisateur. Prévention : **requêtes paramétrées** (prepared statements) — jamais de concaténation de chaînes pour construire des requêtes SQL.

**XSS (Cross-Site Scripting)**
Injection de code JavaScript dans une page vue par d'autres utilisateurs. Prévention : **échappement** des données en sortie (HTML encoding), Content Security Policy (CSP header).

**CSRF (Cross-Site Request Forgery)**
Forcer un utilisateur authentifié à effectuer une action non souhaitée. Prévention : **CSRF tokens** (nonce unique par formulaire), vérification de l'Origin/Referer header, SameSite cookie attribute.

**IDOR (Insecure Direct Object Reference)**
Accès à des ressources en manipulant les IDs (ex: `GET /users/42` → modifier en `GET /users/43`). Prévention : vérification d'autorisation systématique côté serveur, usage d'UUIDs non prédictibles.

---

---

## PILIER XII — Python Fondamentaux Manquants

---

### 12.1 `*args` / `**kwargs` — Mécanisme Interne

**Définition**
- `*args` : collecte les arguments positionnels **en surplus** dans un **tuple**.
- `**kwargs` : collecte les arguments nommés **en surplus** dans un **dict**.

**Mécanisme de résolution des arguments (ordre strict)**
Python résout les paramètres dans cet ordre exact :
1. Paramètres positionnels standard (`def f(a, b)`)
2. `*args` — capture le surplus positionnel
3. Paramètres keyword-only (après `*` ou `*args`)
4. `**kwargs` — capture le surplus nommé

```python
def f(a, b, *args, kw_only, **kwargs):
    pass
f(1, 2, 3, 4, kw_only=5, x=6, y=7)
# a=1, b=2, args=(3,4), kw_only=5, kwargs={'x':6,'y':7}
```

**Unpacking lors de l'appel**
- `f(*[1, 2, 3])` → décompacte la liste en arguments positionnels.
- `f(**{'a': 1, 'b': 2})` → décompacte le dict en arguments nommés.

**Paramètres positional-only (PEP 570, Python 3.8+)**
```python
def f(a, b, /, c, d):  # a et b sont positional-only (avant le /)
```

---

### 12.2 `@classmethod` vs `@staticmethod` vs Méthode d'Instance

| | Méthode d'instance | `@classmethod` | `@staticmethod` |
|---|---|---|---|
| **Premier paramètre** | `self` (instance) | `cls` (classe) | Aucun |
| **Accès à l'instance** | Oui | Non | Non |
| **Accès à la classe** | Via `self.__class__` | Oui, directement | Non |
| **Héritage** | Polymorphe | `cls` = sous-classe si appelé sur sous-classe | Lié à la classe de définition |
| **Cas d'usage** | Comportement d'objet | Constructeurs alternatifs, factory | Fonctions utilitaires liées conceptuellement à la classe |

**Mécanisme interne**
- `@classmethod` : le décorateur enveloppe la fonction dans un **descriptor** qui injecte automatiquement `cls` (la classe sur laquelle la méthode est appelée, pas nécessairement celle de définition — polymorphisme préservé).
- `@staticmethod` : supprime le mécanisme de descriptor — la fonction n'est pas liée, appelée exactement comme une fonction ordinaire.

**Pattern fondamental — Constructeur alternatif** :
```python
class Date:
    def __init__(self, year, month, day): ...

    @classmethod
    def from_string(cls, s):      # cls = Date (ou sous-classe)
        y, m, d = s.split('-')
        return cls(int(y), int(m), int(d))
```

---

### 12.3 `@property` — Getter / Setter / Deleter

**Définition**
`@property` transforme une méthode en **attribut calculé** — l'accès à l'attribut déclenche l'appel de la méthode. Implémenté via le **protocole descripteur** (`__get__`, `__set__`, `__delete__`).

**Mécanisme**
```python
class Circle:
    def __init__(self, radius):
        self._radius = radius

    @property
    def radius(self):           # getter — déclenché par obj.radius
        return self._radius

    @radius.setter
    def radius(self, value):    # setter — déclenché par obj.radius = x
        if value < 0:
            raise ValueError("Radius must be non-negative")
        self._radius = value

    @radius.deleter
    def radius(self):           # deleter — déclenché par del obj.radius
        del self._radius
```

**Différence fondamentale avec un attribut public** : le `@property` permet d'**encapsuler de la logique de validation** tout en conservant l'interface d'accès direct (`obj.radius`), sans casser le code client si l'implémentation interne change.

---

### 12.4 Mutable vs Immutable — `is` vs `==` — Integer Caching

**Types immuables** : `int`, `float`, `bool`, `str`, `tuple`, `frozenset`, `bytes`
**Types mutables** : `list`, `dict`, `set`, `bytearray`, objets utilisateur

**Conséquence fondamentale de l'immuabilité**
- Les types immuables peuvent être **hashables** (si leur valeur ne peut pas changer, le hash reste stable).
- Les types mutables ne peuvent pas être hashables → non utilisables comme clés de `dict` ou éléments de `set`.

**`is` vs `==`**
- `==` : teste l'**égalité de valeur** (appelle `__eq__`).
- `is` : teste l'**identité** (même objet en mémoire, même `id()`).

**Integer Caching (Small Integer Optimization)**
CPython pré-alloue et met en cache les entiers de **-5 à 256** inclus. Ces objets sont des singletons :
```python
a = 256; b = 256; a is b  # True  — même objet caché
a = 257; b = 257; a is b  # False — deux objets distincts créés
```
**String Interning** : CPython interne automatiquement les chaînes qui ressemblent à des identifiants Python (alphanumériques, sans espaces). `sys.intern()` force l'internement.

**Passage d'arguments — "Pass by Object Reference"**
Python n'est ni pass-by-value ni pass-by-reference. C'est du **pass-by-object-reference** (alias pass-by-assignment) :
- La fonction reçoit une référence vers l'objet.
- Pour un type **mutable** : la mutation de l'objet est visible dans le scope appelant.
- Pour un type **immutable** : la réassignation de la variable locale ne modifie pas l'objet original.

```python
def f(lst, x):
    lst.append(1)   # mutable → modifie l'objet original
    x = 99          # immutable → nouvelle liaison locale, original inchangé
```

---

### 12.5 `copy` vs `deepcopy`

**Shallow Copy (`copy.copy()`)**
Crée un **nouvel objet** contenant des **références** vers les mêmes objets enfants que l'original. La copie est indépendante au niveau du conteneur, mais partage les objets imbriqués.
```python
original = [[1, 2], [3, 4]]
shallow = copy.copy(original)
shallow[0].append(99)   # modifie aussi original[0] → [1, 2, 99]
shallow.append([5])     # n'affecte pas original
```

**Deep Copy (`copy.deepcopy()`)**
Crée récursivement des copies de **tous les objets imbriqués**. La copie est totalement indépendante. Utilise un dictionnaire mémo (`memo`) pour gérer les références circulaires.

**Cas d'usage pratique**
- Shallow copy : suffisant pour les structures plates, ou quand les objets imbriqués sont immuables.
- Deep copy : nécessaire pour les structures imbriquées mutables quand l'isolation complète est requise.

---

### 12.6 Exceptions — Hiérarchie & Sémantique de `try/except/else/finally`

**Hiérarchie des exceptions (partielle)**
```
BaseException
├── SystemExit           ← levée par sys.exit() — ne pas attraper avec Exception
├── KeyboardInterrupt    ← Ctrl+C — ne pas attraper avec Exception
├── GeneratorExit        ← fermeture d'un générateur
└── Exception            ← toutes les exceptions "normales"
    ├── ValueError
    ├── TypeError
    ├── AttributeError
    ├── KeyError / IndexError / LookupError
    ├── IOError / OSError
    ├── RuntimeError
    │   └── RecursionError
    ├── StopIteration
    └── ...
```

**Règle critique** : attraper `except Exception` ne capture pas `SystemExit`, `KeyboardInterrupt`, `GeneratorExit`. Attraper `except BaseException` les capture tous — presque jamais souhaité.

**Sémantique complète de `try/except/else/finally`**
```python
try:
    risky_op()
except ValueError as e:
    # Exécuté si ValueError levée dans try
except (TypeError, KeyError):
    # Plusieurs types dans un même except
else:
    # Exécuté si et seulement si aucune exception n'a été levée dans try
    # Utile pour séparer "le code qui peut échouer" de "le code qui suit le succès"
finally:
    # Toujours exécuté — avec ou sans exception, avant return/break/continue
    cleanup()
```

**`else` est crucial** : code dans `else` n'est pas protégé par le `try` — une exception dans `else` n'est pas capturée par les `except` du même bloc.

**Exception chaining**
```python
raise RuntimeError("contexte") from original_exc  # __cause__ = original_exc
raise RuntimeError("contexte")                     # __context__ = original_exc (implicite)
raise RuntimeError("contexte") from None           # supprime le chaining
```

---

### 12.7 Type Hints & Typing Module

**Définition**
Les type hints sont des **annotations** ajoutées au code Python pour déclarer le type attendu des variables, paramètres et valeurs de retour. Ignorées au runtime par l'interpréteur, elles sont utilisées par les outils d'analyse statique (mypy, pyright, Pylance).

**Primitives fondamentales**
```python
from typing import Optional, Union, List, Dict, Tuple, Any, Callable
from typing import TypeVar, Generic, Protocol

# Optional[X] ≡ Union[X, None]
def f(x: Optional[int] = None) -> str: ...

# Union (Python 3.10+ : X | Y)
def g(x: int | str) -> None: ...

# TypeVar — pour le polymorphisme générique
T = TypeVar('T')
def identity(x: T) -> T: return x

# Generic — classe paramétrée
class Stack(Generic[T]):
    def push(self, item: T) -> None: ...
    def pop(self) -> T: ...

# Protocol — structural subtyping (duck typing statique)
class Drawable(Protocol):
    def draw(self) -> None: ...
```

**`Literal`, `Final`, `TypedDict`**
- `Literal['GET', 'POST']` : restreint aux valeurs exactes.
- `Final[int]` : constante non réassignable.
- `TypedDict` : dict avec un schéma de clés/types défini.

**PEP 484, 526, 544, 585, 604** : évolution progressive du système de types Python.

---

### 12.8 List Comprehension vs Generator Expression vs `map/filter`

**List Comprehension**
```python
result = [x**2 for x in range(1000) if x % 2 == 0]
```
- **Évaluation eager** : construit immédiatement la liste entière en mémoire — O(n) espace.
- Syntaxe lisible, idiomatique Python.

**Generator Expression**
```python
result = (x**2 for x in range(1000) if x % 2 == 0)
```
- **Évaluation lazy** : génère les valeurs à la demande — O(1) espace.
- Retourne un objet générateur (iterator).
- Préféré quand : le résultat est consommé une seule fois, la séquence est très large, ou passé directement à une fonction (`sum()`, `any()`, `max()`).

**`map()` / `filter()`**
- Retournent des **iterators lazy** (Python 3).
- Moins lisibles que les comprehensions pour les cas simples.
- Utiles avec des fonctions déjà définies : `map(str, numbers)`.

**Dict / Set Comprehension**
```python
{k: v for k, v in items}   # dict comprehension
{x**2 for x in range(10)}  # set comprehension
```

---

### 12.9 `functools.lru_cache` & Mémoïsation

**Définition**
`@functools.lru_cache(maxsize=128)` est un décorateur qui mémoïse les résultats d'une fonction dans un **cache LRU** (Least Recently Used). Quand un appel est effectué avec les mêmes arguments, le résultat est retourné depuis le cache sans réexécuter la fonction.

**Mécanisme**
- Les arguments doivent être **hashables** (utilisés comme clés du cache).
- `maxsize=None` → cache illimité (`@functools.cache` alias en Python 3.9+).
- `maxsize=128` → cache LRU avec éviction des entrées les moins récemment utilisées quand plein.
- `func.cache_info()` → statistiques (hits, misses, currsize).
- `func.cache_clear()` → vide le cache.

**Complexité** : transforme une récursion exponentielle (ex : Fibonacci naïf O(2^n)) en O(n) avec O(n) espace.

---

## PILIER XIII — GenAI Avancé

---

### 13.1 Fine-tuning LLMs — LoRA / QLoRA / Full Fine-tuning

**Full Fine-tuning**
Mise à jour de **tous** les paramètres du modèle pré-entraîné sur un dataset spécialisé. Coûteux en mémoire (paramètres + gradients + optimiseur = ~16-20 bytes/paramètre en fp32). Risque d'**oubli catastrophique** (catastrophic forgetting) des connaissances générales.

**LoRA — Low-Rank Adaptation (Hu et al., 2021)**
**Hypothèse** : les mises à jour de poids lors du fine-tuning ont une **faible rang intrinsèque** — elles peuvent être approximées par des matrices de rang inférieur.

**Mécanisme** : pour une couche de poids $W_0 \in \mathbb{R}^{d \times k}$, on gèle $W_0$ et on ajoute une décomposition basse-rang apprise :
$$W = W_0 + \Delta W = W_0 + BA$$
où $B \in \mathbb{R}^{d \times r}$, $A \in \mathbb{R}^{r \times k}$, avec $r \ll \min(d, k)$.

- Seuls $A$ et $B$ sont entraînés — réduction drastique des paramètres (ex : $r=8$, $d=k=4096$ → 65536 params au lieu de 16M).
- À l'inférence : $W = W_0 + BA$ (merge possible sans overhead).
- Appliqué typiquement aux matrices Q, K, V, et de projection de l'attention.

**QLoRA (Dettmers et al., 2023)**
LoRA + quantification du modèle de base en **4-bit NF4** (NormalFloat4 — distribution adaptée aux poids Gaussiens des LLMs). Permet de fine-tuner des modèles de 70B sur une seule GPU A100 80GB.
- Modèle de base : 4-bit (gelé, déquantifié à la volée pour le calcul).
- Adaptateurs LoRA : bf16 (entraînables).

**Termes associés** : PEFT (Parameter-Efficient Fine-Tuning), adapter layers, prefix tuning, prompt tuning, instruction tuning, RLHF.

---

### 13.2 RLHF — Reinforcement Learning from Human Feedback

**Pipeline en 3 étapes**

1. **Supervised Fine-Tuning (SFT)** : fine-tuning du LLM pré-entraîné sur des démonstrations humaines de haute qualité → modèle SFT.

2. **Reward Model Training** : collecte de **comparaisons** humaines entre paires de réponses du modèle SFT. Entraînement d'un **modèle de récompense** (RM) qui prédit le score de préférence humaine pour une réponse.

3. **PPO (Proximal Policy Optimization)** : fine-tuning du modèle SFT via RL en maximisant la récompense donnée par le RM, avec une contrainte KL pour éviter de trop s'éloigner du modèle SFT :
$$\text{objective} = \mathbb{E}[r_\theta(x, y) - \beta \cdot \text{KL}(\pi_\theta || \pi_{\text{SFT}})]$$

**DPO (Direct Preference Optimization)** — alternative moderne à PPO : reformule le problème de RL comme une classification binaire sur des paires de préférences, sans modèle de récompense explicite. Plus stable et plus simple à entraîner.

---

### 13.3 Hallucination — Définition & Mitigation

**Définition formelle**
Une **hallucination** est une génération d'un LLM qui est factuellement incorrecte, non fondée sur le contexte fourni, ou incohérente — mais présentée avec confiance. Elle résulte de la nature probabiliste du processus de génération : le modèle génère des tokens vraisemblables, pas nécessairement vrais.

**Taxonomie**
- **Hallucination intrinsèque** : la génération contredit le contexte fourni.
- **Hallucination extrinsèque** : la génération ne peut être vérifiée par le contexte fourni (information inventée).
- **Hallucination factuelle** : faits incorrects sur le monde réel.

**Causes**
- Biais dans les données d'entraînement.
- Architecture du Transformer : génération token par token sans mécanisme explicite de vérification.
- Maximisation de la vraisemblance ≠ maximisation de la factualité.

**Stratégies de mitigation**
- **RAG** : ancrer les réponses dans des documents récupérés → réduction des hallucinations extrinsèques.
- **Température faible** : réduction de l'entropie de la distribution de sortie → réponses plus conservatrices.
- **Self-consistency** : génération de N réponses, sélection par vote majoritaire.
- **Chain-of-Thought** : raisonnement étape par étape → détection des incohérences logiques.
- **Métriques d'évaluation** : Faithfulness (RAGas) — vérifie si la réponse est supportée par le contexte récupéré.

---

### 13.4 Tokenisation — Algorithme BPE

**Définition**
**BPE** (Byte-Pair Encoding) est un algorithme de tokenisation subword qui apprend un vocabulaire de **sous-mots** fréquents. Évite les problèmes du vocabulaire mot-par-mot (OOV — Out Of Vocabulary) et caractère-par-caractère (séquences trop longues).

**Algorithme (entraînement)**
1. Initialiser le vocabulaire avec tous les caractères du corpus (+ token de fin de mot).
2. Compter toutes les **paires de symboles adjacents** dans le corpus.
3. Fusionner la paire la plus fréquente → nouveau symbole.
4. Répéter jusqu'à atteindre la taille de vocabulaire cible.

**À l'inférence** : appliquer les fusions dans l'ordre appris. Un mot inconnu est décomposé en sous-mots connus (ou en caractères en dernier recours).

**Variants**
- **WordPiece** (BERT) : similaire à BPE mais maximise la vraisemblance du corpus (maximisation du log-likelihood).
- **SentencePiece** : opère directement sur les bytes Unicode — language-agnostic, pas de pré-tokenisation.
- **Tiktoken** (OpenAI / GPT-4) : BPE optimisé pour la vitesse.

---

### 13.5 Prompt Engineering — Patterns Formels

**Zero-shot** : la tâche est décrite sans exemple. Le modèle s'appuie entièrement sur ses connaissances pré-entraînées.

**Few-shot** : N exemples (input → output) sont fournis dans le prompt avant la requête cible. Améliore significativement les performances sur les tâches spécialisées sans fine-tuning.

**Chain-of-Thought (CoT)** : demander au modèle de raisonner étape par étape avant de donner la réponse finale. `"Explique ton raisonnement étape par étape."` Améliore les performances sur les tâches de raisonnement (arithmétique, logique). Émergent au-delà de ~100B paramètres.

**ReAct (Reason + Act)** : alternance de pensées (raisonnement interne) et d'actions (appel d'outils). Base des architectures **agentiques**.

**System Prompt** : instruction de niveau supérieur définissant le rôle, le comportement et les contraintes du modèle — prioritaire sur les instructions utilisateur dans la plupart des implémentations.

---

## PILIER XIV — Métriques & Évaluation ML

---

### 14.1 Matrice de Confusion & Métriques Dérivées

**Matrice de confusion (classification binaire)**

```
                  Prédit Positif    Prédit Négatif
Réel Positif    |  TP (Vrai +)   |  FN (Faux -)  |
Réel Négatif    |  FP (Faux +)   |  TN (Vrai -)  |
```

**Métriques fondamentales**

$$\text{Accuracy} = \frac{TP + TN}{TP + TN + FP + FN}$$
→ Trompeuse en cas de **déséquilibre de classes**.

$$\text{Precision} = \frac{TP}{TP + FP}$$
→ "Parmi ce que j'ai prédit positif, combien l'est vraiment ?" — Minimise les faux positifs.

$$\text{Recall (Sensitivity)} = \frac{TP}{TP + FN}$$
→ "Parmi les vrais positifs, combien ai-je détectés ?" — Minimise les faux négatifs.

$$\text{F1-Score} = \frac{2 \cdot \text{Precision} \cdot \text{Recall}}{\text{Precision} + \text{Recall}}$$
→ Moyenne harmonique — équilibre Precision et Recall.

$$\text{Specificity} = \frac{TN}{TN + FP}$$
→ Vrai Negative Rate.

**Quand privilégier Recall vs Precision ?**
- **Recall prioritaire** : détection de cancer, fraude (le coût d'un FN est élevé — manquer un cas grave).
- **Precision prioritaire** : spam filter, recommandations (le coût d'un FP est élevé — bombarder d'irrélevant).

---

### 14.2 ROC Curve & AUC

**Courbe ROC** (Receiver Operating Characteristic) : trace le **Recall (TPR)** en fonction du **FPR (False Positive Rate = FP/(FP+TN))** pour tous les seuils de classification possibles.

**AUC** (Area Under the Curve) : aire sous la courbe ROC ∈ [0, 1].
- AUC = 1 : classificateur parfait.
- AUC = 0.5 : classificateur aléatoire (diagonale).
- AUC = 0 : classificateur parfaitement inversé.

**Interprétation probabiliste** : AUC = probabilité qu'un exemple positif choisi au hasard ait un score supérieur à un exemple négatif choisi au hasard.

**PR-AUC (Precision-Recall AUC)** : préféré à ROC-AUC en cas de **fort déséquilibre de classes** — la courbe ROC peut être trompeusement optimiste car elle utilise TN dans le FPR (et les TN sont abondants en cas de déséquilibre).

**Termes associés** : threshold, calibration, Brier Score, Log Loss (Cross-Entropy comme métrique d'évaluation).

---

### 14.3 Métriques Multi-classes & NLP

**Multi-class**
- **Macro-average** : calcule la métrique par classe, puis moyenne non-pondérée. Traite toutes les classes également.
- **Micro-average** : agrège les TP/FP/FN de toutes les classes avant de calculer. Pondérée implicitement par la fréquence.
- **Weighted-average** : moyenne pondérée par le nombre d'instances par classe.

**Métriques NLP**
- **BLEU** (Bilingual Evaluation Understudy) : compare les n-grammes de la traduction générée avec des références. Range [0,1]. Limité car insensible à la sémantique.
- **ROUGE** (Recall-Oriented Understudy for Gisting Evaluation) : mesure de recall de n-grammes. Standard pour le résumé automatique.
- **BERTScore** : similarité cosinus des embeddings BERT entre génération et référence — sensible à la sémantique.
- **Perplexité** : $\text{PP}(W) = P(w_1, ..., w_N)^{-1/N}$ — mesure à quel point le modèle est "surpris" par un corpus. Plus basse = meilleur modèle de langue.

---

## PILIER XV — Architecture Logicielle Complémentaire

---

### 15.1 Patterns GoF Supplémentaires

#### Observer (Comportemental)

**Définition** : définit une relation **un-à-plusieurs** entre objets : quand un objet change d'état, tous ses **observateurs** sont notifiés et mis à jour automatiquement.

**Structure** :
- `Subject` (Observable) : maintient une liste d'observateurs, notifie via `notify()`.
- `Observer` (interface) : déclare `update()`.
- `ConcreteObserver` : réagit au changement de l'état du sujet.

**Cas d'usage** : event systems, pub/sub, MVC (le modèle notifie la vue), reactive programming.

**Différence avec pub/sub** : dans Observer, le sujet connaît ses observateurs (couplage direct). Dans pub/sub, un **message broker** découple producteur et consommateur.

#### Factory Method (Créationnel)

**Définition** : définit une interface pour créer un objet, mais laisse les sous-classes décider quelle classe instancier. Délègue l'instanciation aux sous-classes.

```python
class Serializer(ABC):
    @abstractmethod
    def serialize(self, data) -> str: ...   # Factory Method

class JSONSerializer(Serializer):
    def serialize(self, data) -> str:
        return json.dumps(data)
```

**Abstract Factory** : crée des **familles** d'objets liés sans spécifier leurs classes concrètes.

#### Decorator Pattern (GoF — ≠ décorateur Python)

**Définition** : attache dynamiquement des responsabilités supplémentaires à un objet en l'enveloppant. Alternative flexible à l'héritage pour étendre les fonctionnalités.

**Note** : le décorateur Python (`@`) est une implémentation possible du pattern Decorator, mais le pattern GoF s'applique généralement aux instances d'objets, pas aux fonctions.

---

### 15.2 Microservices vs Monolithe

| Critère | Monolithe | Microservices |
|---|---|---|
| **Déploiement** | Un seul artefact | Services indépendants |
| **Scalabilité** | Scalabilité globale | Scalabilité par service |
| **Couplage** | Couplage interne fort (appels directs) | Couplage faible (API / messages) |
| **Complexité opérationnelle** | Faible (un seul processus) | Élevée (orchestration, discovery, réseau) |
| **Latence** | In-process (ns) | Inter-process via réseau (ms) |
| **Cohérence** | Transactions ACID locales | Eventual consistency (SAGA pattern) |
| **Débogage** | Simple (stack trace local) | Complexe (distributed tracing, Jaeger) |

**Quand choisir microservices ?** : équipes multiples indépendantes, contraintes de scalabilité différentes par domaine, nécessité de déploiements indépendants. **Anti-pattern** : "microservices prématurés" — commencer par un monolithe bien structuré (modulaire), décomposer si nécessaire.

**SAGA Pattern** : gestion des transactions distribuées. Séquence de transactions locales avec **compensating transactions** en cas d'échec.

---

### 15.3 gRPC vs REST

| Critère | REST | gRPC |
|---|---|---|
| **Protocole** | HTTP/1.1 ou HTTP/2 | HTTP/2 (obligatoire) |
| **Format** | JSON (texte) | Protocol Buffers (binaire) |
| **Contrat** | OpenAPI (optionnel) | `.proto` (obligatoire, généré) |
| **Performance** | Moyen (JSON parsing overhead) | Élevée (binaire, compression, multiplexage) |
| **Streaming** | Limité (SSE, WebSockets séparés) | Natif (unary, server/client/bidirectional streaming) |
| **Browser support** | Natif | Limité (gRPC-Web requis) |
| **Typage** | Faible (JSON) | Fort (Protobuf, génération de code) |
| **Cas d'usage** | APIs publiques, web, mobile | Communication inter-microservices, temps-réel |

**Protocol Buffers** : format de sérialisation binaire. Le schéma `.proto` définit les messages et services. Génère du code client/serveur dans de nombreux langages. Environ 3-10x plus compact et plus rapide que JSON.

---

### 15.4 Message Queues — Kafka

**Définition**
**Apache Kafka** est une plateforme de **streaming d'événements distribués** basée sur un modèle **log append-only**. Publie/consomme des messages via un système de **topics**, **partitions**, et **consumer groups**.

**Concepts fondamentaux**

- **Topic** : catégorie logique de messages. Partitionné pour la parallélisation.
- **Partition** : séquence ordonnée et immuable de messages, identifiés par un **offset**.
- **Producer** : publie des messages dans un topic. Le **partitionnement** est déterminé par une clé de partition (hash) ou round-robin.
- **Consumer Group** : chaque message d'une partition est livré à **exactement un consumer** du groupe. Plusieurs groupes peuvent consommer le même topic indépendamment.
- **Broker** : serveur Kafka. Réplication des partitions entre brokers pour la tolérance aux pannes.
- **Retention** : les messages sont conservés selon une durée ou une taille configurée — les consumers peuvent **relire** les messages.

**Garanties de livraison**
- **At most once** : possible perte de messages, jamais de doublon.
- **At least once** : jamais de perte, possibles doublons.
- **Exactly once** : via transactions Kafka + idempotent producers.

**Avantages vs RabbitMQ** : Kafka est optimisé pour le **débit très élevé**, la persistence longue durée, et la relecture. RabbitMQ est optimisé pour le **routage complexe** et les queues traditionnelles.

---

## PILIER XVI — Tests & Qualité Logicielle

---

### 16.1 Pyramide des Tests

**Définition** : modèle décrivant la distribution recommandée des types de tests selon leur coût et leur rapidité.

```
        /\
       /E2E\        ← Peu, lents, coûteux
      /------\
     /Intégra-\     ← Quelques-uns
    /  tion    \
   /------------\
  /  Unit Tests  \  ← Nombreux, rapides, isolés
 /________________\
```

**Tests Unitaires**
- Testent une **unité de code isolée** (fonction, méthode, classe) sans dépendances externes.
- Rapides (ms), déterministes, nombreux.
- Les dépendances sont remplacées par des **doublures de test** (test doubles).

**Tests d'Intégration**
- Testent l'**interaction entre plusieurs composants** (ex : service + base de données réelle).
- Plus lents, détectent les problèmes d'intégration (contrats d'API, requêtes SQL).
- Recommandation (SITA) : tester avec la vraie BDD — les mocks peuvent masquer les incompatibilités.

**Tests End-to-End (E2E)**
- Testent le **flux complet** de l'application du point de vue utilisateur (navigateur, API publique).
- Très lents, fragiles, maintenables avec parcimonie.

---

### 16.2 Test Doubles — Mocks, Stubs, Fakes, Spies

| Type | Définition | Exemple |
|---|---|---|
| **Dummy** | Objet passé mais jamais utilisé (remplit une signature) | `None` ou objet vide |
| **Stub** | Retourne des réponses prédéfinies sans logique | `return {"status": "ok"}` |
| **Fake** | Implémentation simplifiée fonctionnelle | Base de données en mémoire |
| **Spy** | Stub qui enregistre comment il a été appelé | Vérifie les arguments passés |
| **Mock** | Spy avec des **assertions** sur le comportement attendu | `mock.assert_called_once_with(42)` |

**Distinction Mock vs Stub** : un stub fournit des données, un mock **vérifie des comportements** (interactions). Terme utilisé abusivement pour désigner tous les test doubles.

**`pytest` — Fixtures**
Les fixtures pytest injectent des dépendances dans les tests via des paramètres de fonction :
```python
@pytest.fixture
def db_session():
    session = create_test_session()
    yield session        # analogie avec context manager
    session.rollback()   # cleanup garanti

def test_user_creation(db_session):
    user = User(name="Alice")
    db_session.add(user)
    assert db_session.query(User).count() == 1
```
Scopes : `function` (défaut), `class`, `module`, `session`.

---

### 16.3 TDD — Test-Driven Development

**Cycle Red-Green-Refactor**
1. **Red** : écrire un test qui échoue (pour une fonctionnalité non encore implémentée).
2. **Green** : écrire le minimum de code pour faire passer le test.
3. **Refactor** : améliorer le code sans casser les tests.

**Avantages fondamentaux**
- Force à réfléchir à l'interface avant l'implémentation (design emergent).
- Garantit une couverture de test complète par construction.
- Les tests constituent une **spécification exécutable** du comportement attendu.

**TDD vs BDD** : le BDD (Behavior-Driven Development) est une extension du TDD qui décrit les comportements en langage naturel (Given/When/Then — Gherkin, pytest-bdd).

---

## PILIER XVII — Statistiques & Probabilités pour le ML

---

### 17.1 Distributions Fondamentales

**Distribution Normale (Gaussienne)**
$$\mathcal{N}(\mu, \sigma^2) : f(x) = \frac{1}{\sigma\sqrt{2\pi}} e^{-\frac{(x-\mu)^2}{2\sigma^2}}$$
- Caractérisée par sa moyenne $\mu$ et sa variance $\sigma^2$.
- **Théorème Central Limite** : la somme de $n$ variables aléatoires i.i.d. (de variance finie) converge vers une loi normale quand $n \to \infty$.
- Symétrique, unimodale, queue légère.

**Distribution de Bernoulli**
$X \sim \text{Bern}(p)$ : variable binaire, $P(X=1) = p$, $P(X=0) = 1-p$.
Modèle la probabilité de succès d'un unique essai.

**Distribution Binomiale**
$X \sim B(n, p)$ : nombre de succès en $n$ essais Bernoulli indépendants.
$$P(X=k) = \binom{n}{k} p^k (1-p)^{n-k}$$

**Distribution de Poisson**
$X \sim \text{Poisson}(\lambda)$ : nombre d'événements rares dans un intervalle de temps/espace fixe.
$$P(X=k) = \frac{\lambda^k e^{-\lambda}}{k!}, \quad \mathbb{E}[X] = \text{Var}(X) = \lambda$$

**Distributions continues importantes** : Uniforme, Exponentielle (temps entre événements Poisson), Beta (prior sur probabilités), Dirichlet (généralisation multi-dimensionnelle de Beta).

---

### 17.2 Théorème de Bayes & Inférence

**Théorème de Bayes**
$$P(\theta | X) = \frac{P(X | \theta) \cdot P(\theta)}{P(X)}$$

- $P(\theta)$ : **prior** — croyance sur $\theta$ avant d'observer les données.
- $P(X | \theta)$ : **vraisemblance** (likelihood) — probabilité des données étant donné les paramètres.
- $P(\theta | X)$ : **posterior** — croyance sur $\theta$ après observation des données.
- $P(X)$ : **évidence** (marginale) — constante de normalisation.

**MLE vs MAP**
- **MLE** (Maximum Likelihood Estimation) : $\hat{\theta}_{MLE} = \arg\max_\theta P(X|\theta)$ — maximise la vraisemblance, ignore le prior. Équivalent à MAP avec un prior uniforme.
- **MAP** (Maximum A Posteriori) : $\hat{\theta}_{MAP} = \arg\max_\theta P(\theta|X) = \arg\max_\theta [P(X|\theta) \cdot P(\theta)]$ — inclut le prior → **régularisation implicite**. L2 régularisation ↔ prior Gaussien. L1 régularisation ↔ prior Laplacien.

---

### 17.3 Entropie & Information

**Entropie de Shannon**
$$H(X) = -\sum_i p_i \log_2 p_i$$
Mesure l'**incertitude** (ou l'information) d'une distribution. Maximale pour une distribution uniforme, nulle pour une distribution déterministe.

**Entropie Croisée (Cross-Entropy)**
$$H(p, q) = -\sum_i p_i \log q_i$$
Mesure combien d'information est nécessaire pour encoder les données réelles $p$ en utilisant le modèle $q$. En ML : $p$ = distribution réelle (labels), $q$ = distribution prédite.

$$H(p, q) = H(p) + D_{KL}(p || q)$$

**Divergence KL (Kullback-Leibler)**
$$D_{KL}(p || q) = \sum_i p_i \log \frac{p_i}{q_i}$$
Mesure la distance asymétrique entre deux distributions. Non-symétrique : $D_{KL}(p||q) \neq D_{KL}(q||p)$.

**Information Gain** (pour les arbres de décision) :
$$IG(S, A) = H(S) - \sum_{v \in \text{values}(A)} \frac{|S_v|}{|S|} H(S_v)$$
Réduction d'entropie obtenue en splitant sur l'attribut $A$.

---

### 17.4 Corrélation & Régression — Pièges

**Corrélation de Pearson**
$$r = \frac{\text{Cov}(X, Y)}{\sigma_X \sigma_Y} \in [-1, 1]$$
Mesure la **corrélation linéaire**. $r = 0$ n'implique pas l'indépendance (corrélation non-linéaire possible).

**Corrélation ≠ Causalité**
- Corrélation : association statistique.
- Causalité : la modification de X entraîne une modification de Y.
- **Variable confondante** : une variable $Z$ peut causer à la fois $X$ et $Y$, créant une corrélation spurious entre X et Y.

**Biais de sélection** : si les données d'entraînement ne sont pas représentatives de la population cible, le modèle apprendra une distribution biaisée.

**Data Leakage** : information du futur ou de la cible qui se retrouve dans les features d'entraînement → métriques optimistes en train, effondrement en production.

---

## PILIER XVIII — Normalisation SQL & Déploiement

---

### 18.1 Formes Normales SQL

**Pourquoi normaliser ?** Éliminer la **redondance de données** et les **anomalies de mise à jour** (update, insertion, suppression).

**1NF (Première Forme Normale)**
- Chaque colonne contient des **valeurs atomiques** (pas de listes, pas de valeurs multiples).
- Chaque ligne est unique (clé primaire).

**2NF (Deuxième Forme Normale)**
- Satisfait 1NF.
- Chaque attribut non-clé dépend de la **clé primaire entière** (pas d'une partie de la clé — élimine les dépendances partielles dans les clés composées).

**3NF (Troisième Forme Normale)**
- Satisfait 2NF.
- Aucun attribut non-clé ne dépend d'un autre attribut non-clé (**dépendance transitive** éliminée).

**BCNF (Boyce-Codd Normal Form)**
- Version renforcée de 3NF : pour toute dépendance fonctionnelle $X \to Y$, $X$ doit être une **superclé**.

**Dénormalisation** : compromis délibéré — accepter la redondance pour améliorer les performances de lecture (OLAP, data warehouses). Les colonnes calculées stockées (colonnes matérialisées) sont un exemple.

---

### 18.2 Vues & Vues Matérialisées

**Vue (View)**
- **Table virtuelle** définie par une requête SELECT stockée. Pas de données stockées.
- Chaque accès exécute la requête sous-jacente.
- Utilités : simplification de requêtes complexes, contrôle d'accès (exposer un sous-ensemble de colonnes), abstraction du schéma.

**Vue Matérialisée (Materialized View)**
- Vue dont le **résultat est physiquement stocké** et indexable.
- Doit être **rafraîchie** (manuellement ou automatiquement) quand les données sources changent.
- Trade-off : performances de lecture élevées vs overhead de maintenance et données potentiellement stale.
- Cas d'usage : agrégations coûteuses pour dashboards, data warehouses (PostgreSQL, Snowflake, BigQuery).

---

### 18.3 Stratégies de Déploiement

**Blue/Green Deployment**
- Deux environnements identiques : **Blue** (production actuelle) et **Green** (nouvelle version).
- Le trafic est basculé instantanément de Blue vers Green.
- Rollback immédiat : rebascule vers Blue.
- Coût : double la capacité d'infrastructure.

**Canary Release**
- Déploiement progressif : la nouvelle version reçoit un **faible pourcentage** du trafic (1%, 5%, 20%...).
- Surveillance des métriques sur la fraction canary avant rollout complet.
- Réduction du blast radius en cas de bug.

**Rolling Deployment**
- Remplacement progressif des instances de l'ancienne version par la nouvelle.
- Pas de downtime, mais les deux versions coexistent pendant la transition → risque de problèmes de compatibilité.

**Feature Flags (Feature Toggles)**
- Activation/désactivation de fonctionnalités **sans redéploiement** via la configuration.
- Permettent de livrer du code en production avant de l'activer (dark launch), de tester en A/B, de rollback sans rollout.

---

---

## PILIER XIX — Python Senior : Métaclasses & Concurrence

---

### 19.1 Métaclasses

**Définition formelle**
Une **métaclasse** est une classe dont les instances sont des classes. En Python, toute classe est une instance d'une métaclasse. La métaclasse par défaut est `type`.

**`type` — Le créateur de classes**
`type` est simultanément :
- Une fonction built-in : `type(obj)` retourne la classe de `obj`.
- Une métaclasse : `type(name, bases, namespace)` **crée dynamiquement une classe**.

```python
# Ces deux écritures sont strictement équivalentes :
class MyClass(Base):
    x = 42

MyClass = type('MyClass', (Base,), {'x': 42})
```

**Mécanisme de création d'une classe**
Quand Python rencontre `class MyClass(Base): ...` :
1. Collecte le corps de la classe dans un dictionnaire `namespace`.
2. Identifie la métaclasse : cherche `metaclass=` dans les arguments, sinon hérite de la métaclasse de la première base.
3. Appelle `metaclass(name, bases, namespace)` → retourne la classe.

**Création d'une métaclasse personnalisée**
```python
class Meta(type):
    def __new__(mcs, name, bases, namespace):
        # Intercepte la CRÉATION de la classe
        cls = super().__new__(mcs, name, bases, namespace)
        return cls

    def __init__(cls, name, bases, namespace):
        # Intercepte l'INITIALISATION de la classe
        super().__init__(name, bases, namespace)

    def __call__(cls, *args, **kwargs):
        # Intercepte l'INSTANCIATION (MyClass())
        instance = super().__call__(*args, **kwargs)
        return instance

class MyClass(metaclass=Meta):
    pass
```

**Cas d'usage légitimes**
- **Singleton via métaclasse** : `__call__` retourne toujours la même instance.
- **ORM** (Django, SQLAlchemy) : enregistrement automatique des modèles, création des colonnes SQL à partir des attributs de classe.
- **Validation de classe** : vérifier à la création qu'une classe implémente une interface.
- **API déclarative** (ex : ProtocolBuffers) : transformer des déclarations de champs en objets complexes.

**`__init_subclass__` — Alternative moderne aux métaclasses**
Depuis Python 3.6, `__init_subclass__` est appelé sur la classe parente quand une sous-classe est créée — couvre 80% des cas d'usage des métaclasses sans leur complexité.

```python
class Base:
    def __init_subclass__(cls, required_attr=None, **kwargs):
        super().__init_subclass__(**kwargs)
        if required_attr and not hasattr(cls, required_attr):
            raise TypeError(f"{cls.__name__} must define '{required_attr}'")

class Child(Base, required_attr='process'):
    def process(self): ...
```

**`__class_getitem__`** — Permet la syntaxe `MyClass[T]` (support des génériques).

**Termes académiques associés** : metaclass, `type`, `__new__`, `__init__`, `__call__` (sur métaclasse), `__prepare__` (namespace factory), `__init_subclass__`, descriptor, class decorator (alternative légère).

---

### 19.2 `concurrent.futures` — Exécution Parallèle Abstraite

**Définition formelle**
`concurrent.futures` fournit une interface de haut niveau pour l'exécution asynchrone de callables via des **Executors**. Abstrait `threading` et `multiprocessing` derrière une API uniforme basée sur les **Future objects**.

**`ThreadPoolExecutor` vs `ProcessPoolExecutor`**

| | `ThreadPoolExecutor` | `ProcessPoolExecutor` |
|---|---|---|
| **Mécanisme** | Pool de threads OS | Pool de sous-processus |
| **GIL** | Limité par le GIL | Contourne le GIL |
| **Cas d'usage** | I/O-bound | CPU-bound |
| **Overhead** | Faible (mémoire partagée) | Élevé (pickle/IPC) |
| **Partage d'état** | Mémoire partagée (risque de race condition) | Mémoire isolée (sécurisé) |

**Patterns fondamentaux**
```python
from concurrent.futures import ThreadPoolExecutor, as_completed

# Pattern 1 — map() : résultats dans l'ordre de soumission
with ThreadPoolExecutor(max_workers=4) as executor:
    results = list(executor.map(fetch_url, urls))

# Pattern 2 — submit() + as_completed() : résultats dans l'ordre de complétion
futures = {executor.submit(fetch_url, url): url for url in urls}
for future in as_completed(futures):
    url = futures[future]
    try:
        result = future.result()
    except Exception as exc:
        print(f"{url} raised {exc}")
```

**`Future` object** : représente un calcul potentiellement pas encore terminé. Méthodes : `result()` (bloquant), `done()`, `cancel()`, `add_done_callback()`.

---

### 19.3 `dataclasses` — Définition & Mécanisme

**Définition**
`@dataclass` est un décorateur qui **génère automatiquement** les méthodes boilerplate (`__init__`, `__repr__`, `__eq__`) à partir des annotations de classe.

```python
from dataclasses import dataclass, field
from typing import List

@dataclass(order=True, frozen=True)
class Point:
    x: float
    y: float
    tags: List[str] = field(default_factory=list)  # évite le mutable default
    _cache: dict = field(default_factory=dict, repr=False, compare=False)
```

**Paramètres du décorateur**
- `eq=True` (défaut) : génère `__eq__` (comparaison par valeur).
- `order=True` : génère `__lt__`, `__le__`, `__gt__`, `__ge__` (nécessite `eq=True`).
- `frozen=True` : génère `__hash__`, rend l'instance immuable (lève `FrozenInstanceError` sur assignation).
- `slots=True` (Python 3.10+) : génère `__slots__` → économie mémoire, accès plus rapide.

**`__post_init__`** : méthode spéciale appelée après `__init__` généré — pour validation ou transformation.

```python
@dataclass
class PositiveInt:
    value: int

    def __post_init__(self):
        if self.value <= 0:
            raise ValueError(f"value must be positive, got {self.value}")
```

**Comparaison des alternatives**

| | `dataclass` | `NamedTuple` | `TypedDict` |
|---|---|---|---|
| **Mutabilité** | Mutable (ou `frozen`) | Immutable | Dict mutable |
| **Héritage** | Oui | Limité | Non |
| **Runtime** | Classe normale | Sous-classe de tuple | Dict au runtime |
| **Cas d'usage** | Data objects classiques | Données immuables légères | Typing de dicts |

---

### 19.4 Walrus Operator `:=` (PEP 572)

**Définition**
L'opérateur **walrus** (`:=`) est un **opérateur d'assignation dans une expression** — il assigne une valeur à une variable ET retourne cette valeur dans le même statement.

```python
# Sans walrus — double appel
data = fetch_data()
if data:
    process(data)

# Avec walrus — un seul appel
if data := fetch_data():
    process(data)

# Cas d'usage classique — while loop
while chunk := file.read(8192):
    process(chunk)

# Dans une comprehension — évite de recalculer
results = [y for x in data if (y := expensive(x)) > 0]
```

**Portée** : la variable assignée par `:=` dans une comprehension **échappe** au scope de la comprehension (contrairement aux variables de boucle classiques depuis Python 3). C'est la principale différence sémantique.

---

### 19.5 `weakref` — Références Faibles

**Définition**
Une **référence faible** (`weakref.ref`) pointe vers un objet sans incrémenter son `ob_refcnt`. L'objet peut donc être collecté par le GC même si des références faibles existent — la référence faible retourne alors `None`.

**Cas d'usage fondamental — Caches sans fuite mémoire**
```python
import weakref

cache = weakref.WeakValueDictionary()
# Les valeurs sont collectées si plus aucune référence forte n'existe
cache['key'] = MyObject()    # Si MyObject non référencé ailleurs → collecté
```

**`weakref.WeakValueDictionary`** : dict dont les valeurs sont des références faibles.
**`weakref.WeakKeyDictionary`** : dict dont les clés sont des références faibles.
**`weakref.WeakSet`** : set avec références faibles.

**Problème résolu** : un cache dict normal maintient en vie tous ses objets (référence forte). Avec un `WeakValueDictionary`, les objets non utilisés ailleurs sont collectés automatiquement.

---

## PILIER XX — Algèbre Linéaire pour le ML

---

### 20.1 Vecteurs, Matrices & Opérations Fondamentales

**Définitions**
- **Vecteur** $\mathbf{v} \in \mathbb{R}^n$ : liste ordonnée de $n$ scalaires.
- **Matrice** $A \in \mathbb{R}^{m \times n}$ : tableau rectangulaire de scalaires, $m$ lignes, $n$ colonnes.
- **Transposée** $A^T$ : échange lignes et colonnes. $(AB)^T = B^T A^T$.
- **Produit matriciel** $C = AB$ où $C_{ij} = \sum_k A_{ik} B_{kj}$ — nécessite que les dimensions intérieures soient compatibles ($m \times k) \cdot (k \times n) = m \times n$.

**Propriétés essentielles**
- Produit matriciel : **non commutatif** ($AB \neq BA$ en général), associatif, distributif.
- **Matrice identité** $I$ : $AI = IA = A$.
- **Matrice inverse** $A^{-1}$ : $AA^{-1} = I$. Existe si et seulement si $\det(A) \neq 0$ (matrice inversible / non singulière).

---

### 20.2 Valeurs Propres & Vecteurs Propres (Eigenvalues / Eigenvectors)

**Définition formelle**
Pour une matrice carrée $A \in \mathbb{R}^{n \times n}$, un **vecteur propre** $\mathbf{v} \neq \mathbf{0}$ et sa **valeur propre** $\lambda$ vérifient :

$$A\mathbf{v} = \lambda \mathbf{v}$$

L'application de $A$ au vecteur propre ne change que son **échelle**, pas sa **direction**.

**Interprétation géométrique**
Les vecteurs propres sont les directions invariantes de la transformation linéaire représentée par $A$. Les valeurs propres mesurent l'amplification (ou compression) dans ces directions.

**Calcul**
$(A - \lambda I)\mathbf{v} = \mathbf{0}$ admet une solution non-nulle $\Leftrightarrow$ $\det(A - \lambda I) = 0$ (**équation caractéristique**).

**Applications en ML**
- **PCA** : les composantes principales sont les **vecteurs propres** de la matrice de covariance $\Sigma$. Les **valeurs propres** mesurent la variance expliquée dans chaque direction.
- **PageRank** : le vecteur de rang est le vecteur propre dominant de la matrice de transition du graphe web.
- **Spectral Clustering** : décomposition spectrale de la matrice Laplacienne du graphe.

---

### 20.3 SVD — Singular Value Decomposition

**Définition formelle**
Toute matrice $A \in \mathbb{R}^{m \times n}$ (pas nécessairement carrée) admet une décomposition :

$$A = U \Sigma V^T$$

où :
- $U \in \mathbb{R}^{m \times m}$ : matrice **orthogonale** — vecteurs singuliers gauches (colonnes = base de l'espace image).
- $\Sigma \in \mathbb{R}^{m \times n}$ : matrice **diagonale** — valeurs singulières $\sigma_1 \geq \sigma_2 \geq ... \geq 0$.
- $V^T \in \mathbb{R}^{n \times n}$ : matrice **orthogonale** — vecteurs singuliers droits (colonnes de $V$ = base de l'espace entrée).

**Lien avec les valeurs propres**
- Les valeurs singulières $\sigma_i = \sqrt{\lambda_i(A^T A)}$ — racines carrées des valeurs propres de $A^T A$.
- Les colonnes de $V$ sont les vecteurs propres de $A^T A$.
- Les colonnes de $U$ sont les vecteurs propres de $AA^T$.

**SVD Tronquée (Truncated SVD)**
En gardant uniquement les $k$ plus grandes valeurs singulières :
$$A \approx U_k \Sigma_k V_k^T$$
Cette approximation de rang $k$ est **optimale** au sens de la norme de Frobenius (théorème de Eckart–Young).

**Applications**
- **PCA** : $\text{PCA}(X) = U_k$ (directement depuis SVD de la matrice centrée $X$).
- **Recommandation (SVD matricielle)** : factorisation de la matrice utilisateur×item.
- **LSA** (Latent Semantic Analysis) : SVD de la matrice TF-IDF.
- **Compression d'images** : approximation de rang $k$ d'une matrice de pixels.
- **LoRA** : hypothèse que $\Delta W$ est de bas rang → approximé par $BA$ (analogue SVD tronquée).

**Rang d'une matrice**
- **Rang** : nombre de valeurs singulières non-nulles = dimension de l'espace image.
- **Espace nul (null space / kernel)** : ensemble des $\mathbf{x}$ tels que $A\mathbf{x} = \mathbf{0}$.
- **Espace colonne (column space / range)** : espace engendré par les colonnes de $A$.
- Théorème du rang : $\text{rang}(A) + \dim(\ker(A)) = n$ (nombre de colonnes).

---

### 20.4 Normes de Vecteurs & Matrices

**Normes vectorielles**
- **L1 (Manhattan)** : $\|\mathbf{v}\|_1 = \sum_i |v_i|$ — induit la sparsité (L1 regularization).
- **L2 (Euclidienne)** : $\|\mathbf{v}\|_2 = \sqrt{\sum_i v_i^2}$ — pénalise les grands poids (L2 regularization).
- **L∞ (Chebyshev)** : $\|\mathbf{v}\|_\infty = \max_i |v_i|$ — valeur maximale absolue.

**Norme de Frobenius (matrices)**
$$\|A\|_F = \sqrt{\sum_{i,j} a_{ij}^2} = \sqrt{\text{tr}(A^T A)} = \sqrt{\sum_i \sigma_i^2}$$

---

## PILIER XXI — Réseaux de Neurones Classiques

---

### 21.1 CNN — Convolutional Neural Networks

**Motivation**
Les CNN exploitent trois propriétés des données visuelles : **localité** (les pixels proches sont liés), **stationnarité** (un même motif peut apparaître n'importe où), et **hiérarchie** (les motifs complexes se construisent à partir de primitives simples).

**Opération de convolution 2D**
```
Output[i,j] = Σ_u Σ_v Input[i+u, j+v] × Kernel[u,v] + bias
```
- **Kernel (filtre)** : matrice de poids appris $k \times k$ (typiquement 3×3 ou 5×5).
- **Stride** $s$ : déplacement du kernel entre chaque application. $s=1$ → sortie de même taille, $s=2$ → downsampling.
- **Padding** : ajout de zéros autour de l'entrée. `same` → sortie même taille que l'entrée. `valid` → pas de padding.
- **Dimension de sortie** : $\lfloor \frac{n + 2p - k}{s} \rfloor + 1$ où $n$ = taille entrée, $p$ = padding, $k$ = kernel size.

**Paramètres partagés** : le même kernel est appliqué à toute la carte d'entrée — c'est l'**invariance par translation**. Un réseau FC équivalent aurait beaucoup plus de paramètres.

**Pooling**
- **Max Pooling** : retient la valeur maximale dans chaque région → invariance aux petites translations, réduction dimensionnelle.
- **Average Pooling** : moyenne de la région.
- **Global Average Pooling (GAP)** : moyenne globale de chaque feature map → vecteur de taille = nombre de filtres. Remplace les couches FC dans les architectures modernes.

**Champ réceptif (Receptive Field)**
Région de l'entrée originale qui influence un neurone donné dans une couche profonde. Augmente avec la profondeur. L'empilement de convolutions 3×3 est préféré aux grandes convolutions car même réceptive field, moins de paramètres, plus de non-linéarités.

**Architectures historiques clés**
- **LeNet-5** (LeCun, 1998) : premier CNN pratique — chiffres manuscrits.
- **AlexNet** (2012) : Deep CNN + ReLU + Dropout + GPU — révolution ImageNet.
- **VGG** (2014) : empilage de conv 3×3 — rapport profondeur/performance.
- **ResNet** (2015) : **skip connections** (connexions résiduelles) — résout le vanishing gradient dans les réseaux très profonds. $H(x) = F(x) + x$.
- **Inception** : blocs avec convolutions de tailles différentes en parallèle.

**Invariance vs Équivariance**
- **Équivariance à la translation** : si l'entrée est translatée, la sortie est translatée de la même façon (propriété des couches de convolution).
- **Invariance à la translation** : la sortie ne change pas si l'entrée est translatée (propriété du global pooling / classification finale).

---

### 21.2 RNN, LSTM & GRU

**RNN (Recurrent Neural Network) — Problème fondamental**

L'état caché $h_t = f(W_h h_{t-1} + W_x x_t + b)$ traite les séquences en maintenant un état recurrent. Mais lors de la backpropagation à travers le temps (**BPTT**), le gradient est multiplié par $W_h$ à chaque step :

$$\frac{\partial h_T}{\partial h_t} = \prod_{k=t}^{T-1} W_h$$

Si $\|W_h\| < 1$ : **vanishing gradient** (les dépendances longue portée disparaissent).
Si $\|W_h\| > 1$ : **exploding gradient** (instabilité).

**LSTM (Long Short-Term Memory) — Hochreiter & Schmidhuber, 1997**

Introduit une **cell state** $c_t$ (mémoire à long terme) et 3 **gates** (portes) qui contrôlent le flux d'information :

$$f_t = \sigma(W_f [h_{t-1}, x_t] + b_f) \quad \text{(Forget gate — oublier l'ancienne mémoire)}$$
$$i_t = \sigma(W_i [h_{t-1}, x_t] + b_i) \quad \text{(Input gate — écrire dans la mémoire)}$$
$$\tilde{c}_t = \tanh(W_c [h_{t-1}, x_t] + b_c) \quad \text{(Candidate memory)}$$
$$c_t = f_t \odot c_{t-1} + i_t \odot \tilde{c}_t \quad \text{(Mise à jour cell state)}$$
$$o_t = \sigma(W_o [h_{t-1}, x_t] + b_o) \quad \text{(Output gate)}$$
$$h_t = o_t \odot \tanh(c_t) \quad \text{(Hidden state)}$$

**Pourquoi ça résout le vanishing gradient ?**
La cell state $c_t$ est mise à jour par une **addition** (pas une multiplication répétée par $W$). Le gradient de la perte par rapport à $c_{t-1}$ inclut $f_t$ (valeur entre 0 et 1) — mais pas de multiplication répétée par $W$. Les gradients peuvent circuler sur de longues distances via la cell state.

**GRU (Gated Recurrent Unit) — Cho et al., 2014**
Simplification du LSTM : 2 gates au lieu de 3, fusionne cell state et hidden state. Moins de paramètres, performances comparables sur beaucoup de tâches.

$$z_t = \sigma(W_z [h_{t-1}, x_t]) \quad \text{(Update gate)}$$
$$r_t = \sigma(W_r [h_{t-1}, x_t]) \quad \text{(Reset gate)}$$
$$h_t = (1 - z_t) \odot h_{t-1} + z_t \odot \tanh(W[r_t \odot h_{t-1}, x_t])$$

**Contexte historique** : LSTM/GRU dominaient le NLP jusqu'en 2017 (Transformer "Attention Is All You Need"). Toujours utilisés pour les séquences où la longueur est variable et modérée, ou sur des ressources limitées.

---

### 21.3 Word2Vec — Embeddings de Mots

**Définition**
Word2Vec (Mikolov et al., 2013) est un modèle qui apprend des **représentations vectorielles denses** des mots (word embeddings) telles que les mots sémantiquement proches ont des vecteurs géométriquement proches.

**Hypothèse distributionnelle** : *"Un mot est caractérisé par la compagnie qu'il tient"* (Firth, 1957). Les mots qui apparaissent dans des contextes similaires ont des significations proches.

**Deux architectures**

**CBOW (Continuous Bag of Words)**
Prédit le mot central à partir des mots du contexte (fenêtre de taille $w$ autour du mot cible). Entraînement rapide, bon pour les petits datasets.

**Skip-gram**
Prédit les mots du contexte à partir du mot central. Plus lent, meilleur pour les mots rares et les grands datasets.

**Negative Sampling**
Pour éviter de calculer le softmax sur tout le vocabulaire (coûteux), le Skip-gram avec negative sampling transforme le problème en **classification binaire** : distinguer les vrais mots de contexte de $k$ mots tirés au hasard ("négatifs").

**Propriétés arithmétiques des embeddings**
$$\text{vec}(\text{"roi"}) - \text{vec}(\text{"homme"}) + \text{vec}(\text{"femme"}) \approx \text{vec}(\text{"reine"})$$
Les relations sémantiques et syntaxiques sont encodées comme des **translations dans l'espace vectoriel**.

**Évolution** : Word2Vec → GloVe (Global Vectors, co-occurrence globale) → FastText (embeddings de sous-mots, gère les OOV) → BERT (contextual embeddings — un même mot a des vecteurs différents selon le contexte).

**Différence fondamentale avec BERT** : Word2Vec produit des embeddings **statiques** (un vecteur par mot, indépendant du contexte). BERT produit des embeddings **contextuels** (le vecteur de "banque" diffère selon qu'on parle de finance ou de rivière).

---

## PILIER XXII — ML/Évaluation Complémentaire

---

### 22.1 Cross-Validation

**Définition formelle**
La **validation croisée** est une technique d'évaluation qui estime la performance de généralisation d'un modèle en le testant sur différentes partitions des données, réduisant le biais induit par un unique split train/test.

**K-Fold Cross-Validation**
1. Diviser le dataset en $k$ plis (folds) de taille égale.
2. Pour chaque fold $i$ : entraîner sur les $k-1$ autres folds, évaluer sur le fold $i$.
3. La métrique finale est la **moyenne** sur les $k$ évaluations (± écart-type).

**Variantes**

| Méthode | Description | Quand l'utiliser |
|---|---|---|
| **k-Fold** (standard) | $k$ splits aléatoires | Cas général |
| **Stratified k-Fold** | Preserve la proportion des classes dans chaque fold | Classification avec déséquilibre de classes |
| **Leave-One-Out (LOO)** | $k = n$ — chaque exemple est un fold | Très petit dataset (coûteux) |
| **Time Series Split** | Splits temporels croissants — jamais de données futures en train | Séries temporelles |
| **Group k-Fold** | Les exemples d'un même groupe sont dans le même fold | Données groupées (patients, sujets) |

**Train / Validation / Test — rôles distincts**
- **Train** : apprentissage des paramètres.
- **Validation** : sélection des hyperparamètres et de l'architecture (model selection).
- **Test** : estimation finale non biaisée de la performance — **utilisé une seule fois**.

**Data leakage dans la CV** : les transformations (StandardScaler, PCA, feature engineering) doivent être **fit uniquement sur le train fold** et appliquées au val fold. Les fitter sur l'ensemble complet avant la CV introduit du leakage.

---

### 22.2 Feature Engineering & Preprocessing

**StandardScaler vs MinMaxScaler**

| | `StandardScaler` | `MinMaxScaler` |
|---|---|---|
| **Formule** | $x' = \frac{x - \mu}{\sigma}$ | $x' = \frac{x - x_{min}}{x_{max} - x_{min}}$ |
| **Range** | $\mu=0$, $\sigma=1$ (non borné) | $[0, 1]$ |
| **Sensible aux outliers** | Modérément | Très sensible |
| **Quand l'utiliser** | Algorithmes basés sur la distance (SVM, KNN), gradient descent | Réseaux de neurones, algorithmes nécessitant un range défini |

**RobustScaler** : utilise la médiane et l'IQR → robuste aux outliers. Recommandé si le dataset contient des outliers significatifs.

**Encodage des variables catégorielles**

- **One-Hot Encoding** : crée $n-1$ colonnes binaires. Risque de **multicolinéarité** (dummy variable trap) si $n$ colonnes retenues. Problème avec les variables à haute cardinalité (explosion dimensionnelle).
- **Label Encoding** : mappe les catégories à des entiers $0, 1, ..., n-1$. **Introduit un ordre artificiel** — uniquement valide pour les variables ordinales.
- **Target Encoding** : remplace chaque catégorie par la moyenne de la cible dans cette catégorie. Risque de **data leakage** si calculé sur tout le train (utiliser la CV interne). Régularisation nécessaire.
- **Frequency Encoding** : remplace par la fréquence d'apparition — préserve l'information de rareté.
- **Embedding** (pour DL) : couche apprise qui mappe chaque catégorie à un vecteur dense.

---

### 22.3 Déséquilibre de Classes & SMOTE

**SMOTE (Synthetic Minority Over-sampling Technique)**

**Mécanisme interne** :
1. Pour chaque exemple de la classe minoritaire $x_i$, trouver ses $k$ plus proches voisins (dans la classe minoritaire).
2. Choisir aléatoirement un voisin $x_j$.
3. Générer un point synthétique : $x_{new} = x_i + \lambda \cdot (x_j - x_i)$ avec $\lambda \sim \text{Uniform}(0, 1)$.

SMOTE génère des points **entre** les exemples existants (interpolation dans l'espace des features), pas simplement des duplications.

**Variantes** : SMOTE-NC (features mixtes numériques/catégorielles), ADASYN (adaptatif — plus de synth. là où le classifieur fait des erreurs), SMOTE-Tomek (SMOTE + suppression des exemples ambigus à la frontière).

**Alternatives à l'oversampling**
- **Undersampling** : Random undersampling, NearMiss, Tomek Links.
- **`class_weight='balanced'`** dans scikit-learn : pondère les exemples de chaque classe inversement à leur fréquence dans la fonction de perte.
- **Focal Loss** (Lin et al., 2017) : down-weight les exemples faciles (bien classifiés), focus sur les difficiles.
- **Threshold adjustment** : changer le seuil de décision (pas toujours 0.5) pour équilibrer Precision/Recall.

---

### 22.4 Hyperparameter Tuning

**Grid Search**
Exhaustif — évalue toutes les combinaisons d'hyperparamètres dans une grille. Complexité : $O(\prod_i |H_i|)$ — exponentiel en nombre d'hyperparamètres. Garanti de trouver le meilleur dans la grille.

**Random Search (Bergstra & Bengio, 2012)**
Échantillonne aléatoirement dans l'espace des hyperparamètres. Théoriquement : si $p$ dimensions sont importantes parmi $n$, Random Search en explore $p$ fois plus que Grid Search à budget égal. Empiriquement supérieur à Grid Search sur les espaces de haute dimension.

**Bayesian Optimization**
Modélise la **fonction objectif** (performance du modèle selon les hyperparamètres) par un **surrogate model** (typiquement un Gaussian Process — GPR) et utilise une **acquisition function** pour décider quel point évaluer ensuite.

**Mécanisme en 3 étapes** :
1. Évaluer la performance sur quelques points initiaux.
2. Ajuster le GPR sur les observations.
3. Maximiser l'acquisition function (ex : Expected Improvement) → point le plus prometteur → évaluer → retour à 2.

**Acquisition functions** :
- **Expected Improvement (EI)** : espérance du gain par rapport au meilleur résultat actuel.
- **UCB (Upper Confidence Bound)** : exploration vs exploitation.
- **PI (Probability of Improvement)**.

**Outils** : Optuna, Ray Tune, Hyperopt, KerasTuner.

**Early Stopping**
Arrêt de l'entraînement quand la métrique de validation cesse de s'améliorer pendant $n$ epochs (**patience**). Régularisation implicite — évite l'overfitting sans modifier l'architecture.

---

## PILIER XXIII — Architecture Logicielle Complémentaire II

---

### 23.1 Principes DRY, KISS, YAGNI

**DRY — Don't Repeat Yourself (Hunt & Thomas, "The Pragmatic Programmer")**
*"Every piece of knowledge must have a single, unambiguous, authoritative representation within a system."*
- La duplication de **logique** (pas de code) est le problème. Deux morceaux de code peuvent être syntaxiquement similaires sans être du DRY — si les raisons de changer sont différentes, les dupliquer est correct.
- **Violation DRY** → toute modification nécessite des mises à jour en plusieurs endroits → source de bugs.

**KISS — Keep It Simple, Stupid**
La simplicité doit être un objectif de conception. Un design plus simple est plus facile à comprendre, tester, modifier et déboguer. **La complexité accidentelle** (celle qu'on introduit, pas celle inhérente au problème) doit être minimisée.

**YAGNI — You Aren't Gonna Need It (XP — Extreme Programming)**
Ne pas implémenter une fonctionnalité tant qu'elle n'est pas nécessaire. Les prédictions sur les besoins futurs sont rarement exactes. Le code inutile coûte à écrire, tester, maintenir et comprendre.

**Tension entre principes** : DRY pousse à abstraire ; YAGNI pousse à ne pas abstraire prématurément. La règle de trois est utile : **duplicer deux fois, abstraire à la troisième occurrence**.

---

### 23.2 Couplage & Cohésion

**Cohésion** : degré auquel les éléments d'un module **appartiennent ensemble**. Une haute cohésion signifie qu'un module fait une seule chose bien définie.

**Types de cohésion (du plus faible au plus fort)**
- **Accidentelle** : éléments groupés arbitrairement (classe "Utils" fourre-tout).
- **Logique** : éléments réalisant des fonctions similaires mais sans lien fonctionnel.
- **Temporelle** : éléments exécutés au même moment (initialisation).
- **Fonctionnelle** : tous les éléments contribuent à une seule tâche bien définie. **Idéal**.

**Couplage** : degré de **dépendance entre modules**. Un faible couplage est souhaitable — les modules peuvent évoluer indépendamment.

**Types de couplage (du plus fort au plus faible)**
- **Contenu** : un module modifie directement les données internes d'un autre. Pire forme.
- **Commun** : partage de données globales (variables globales).
- **Contrôle** : un module passe un flag qui contrôle le comportement d'un autre.
- **Stamp** : partage d'une structure de données mais utilise seulement une partie.
- **Data** : échange uniquement des données primitives nécessaires. **Idéal**.

**Couplage afférent/efférent (Robert C. Martin)**
- **Ca (afférent)** : nombre de classes qui dépendent du module. Mesure la "responsabilité".
- **Ce (efférent)** : nombre de classes dont le module dépend. Mesure la "dépendance".
- **Instabilité** $I = Ce / (Ca + Ce)$ ∈ [0,1]. $I=0$ = module stable (beaucoup de dépendants, peu de dépendances). $I=1$ = module instable.
- **Principe des dépendances stables (SDP)** : les modules instables doivent dépendre des modules stables.

---

### 23.3 Patterns GoF Manquants

#### Adapter (Structurel)

**Définition** : convertit l'interface d'une classe en une autre interface attendue par les clients. Permet à des classes incompatibles de collaborer.

```python
class EuropeanSocket:          # interface existante (incompatible)
    def voltage(self): return 220

class AmericanDevice:          # interface attendue
    def connect(self, socket):
        assert socket.volt() == 110   # attend volt(), pas voltage()

class SocketAdapter:           # Adapter
    def __init__(self, socket):
        self._socket = socket
    def volt(self):            # traduit volt() → voltage()
        return self._socket.voltage() // 2
```

**Cas d'usage** : intégration de bibliothèques tierces, migration progressive d'une API vers une autre.

#### Builder (Créationnel)

**Définition** : sépare la construction d'un objet complexe de sa représentation finale. Permet de construire le même objet avec différentes configurations via un processus étape par étape.

```python
class QueryBuilder:
    def __init__(self): self._query = {}
    def table(self, t): self._query['table'] = t; return self
    def where(self, cond): self._query['where'] = cond; return self
    def limit(self, n): self._query['limit'] = n; return self
    def build(self): return self._query

query = QueryBuilder().table('users').where('age > 18').limit(10).build()
```

**Avantage** : évite les constructeurs avec de nombreux paramètres optionnels (telescoping constructor anti-pattern).

#### Command (Comportemental)

**Définition** : encapsule une requête comme un objet, permettant de paramétrer des actions, de les mettre en queue, de les logger, et de supporter le **undo/redo**.

**Structure** :
- `Command` (interface) : `execute()`, `undo()`.
- `ConcreteCommand` : encapsule l'action et le receiver.
- `Invoker` : déclenche la commande, peut maintenir l'historique.
- `Receiver` : l'objet qui effectue le travail réel.

**Cas d'usage** : queues de tâches (celery), undo/redo (éditeurs), transactions, macro-commandes.

---

### 23.4 Event Sourcing & CQRS

**Event Sourcing**

**Définition** : au lieu de stocker l'**état courant** des entités, on stocke la **séquence complète des événements** qui ont conduit à cet état. L'état courant est reconstruit en "rejouant" les événements.

```
État traditionnel :  users.balance = 150
Event Sourcing   :   [Deposited(200), Withdrew(50)] → balance = 150
```

**Avantages** : audit trail complet, possibilité de remonter dans le temps (time-travel debugging), projections multiples du même stream d'événements, event replay pour corriger des bugs.

**CQRS — Command Query Responsibility Segregation**

**Définition** : séparation stricte entre les opérations qui **modifient l'état** (Commands) et celles qui le **lisent** (Queries). Les deux peuvent utiliser des modèles de données différents et des bases de données différentes.

```
Write side (Commands) → Events → Read side (Queries)
      [Modèle normalisé]             [Modèle dénormalisé / optimisé lecture]
```

**Avantages** : scalabilité indépendante en lecture et en écriture, modèles de lecture optimisés (vues matérialisées, Elasticsearch).

**Risque** : **eventual consistency** entre write et read side.

---

### 23.5 Circuit Breaker

**Définition** : pattern de résilience qui protège un service contre les défaillances en cascade d'un service dépendant.

**États du Circuit Breaker**

```
CLOSED ──(failures > threshold)──→ OPEN ──(timeout)──→ HALF-OPEN
  ↑                                                          │
  └─────────────(success)───────────────────────────────────┘
```

- **CLOSED** : les requêtes passent normalement. Comptabilise les échecs.
- **OPEN** : les requêtes sont bloquées immédiatement (fail fast) sans appeler le service défaillant. Après un timeout, transition vers HALF-OPEN.
- **HALF-OPEN** : laisse passer quelques requêtes de test. Si elles réussissent → CLOSED. Sinon → OPEN.

**Différence avec Retry** : Retry répète les appels jusqu'à succès — amplifie la charge sur un service déjà surchargé. Circuit Breaker coupe le flux pour laisser le service se rétablir.

---

### 23.6 12-Factor App

**Définition** : méthodologie de 12 principes pour construire des applications **cloud-native**, portables et scalables (Heroku, 2012).

| Factor | Principe |
|---|---|
| **1. Codebase** | Un dépôt, plusieurs déploiements (dev/staging/prod) |
| **2. Dependencies** | Déclarer et isoler les dépendances explicitement (`requirements.txt`, `pyproject.toml`) |
| **3. Config** | Stocker la config dans l'environnement (variables d'env), jamais dans le code |
| **4. Backing services** | Traiter les services externes (DB, cache, queue) comme des ressources attachées |
| **5. Build/Release/Run** | Séparer strictement les trois étapes |
| **6. Processes** | Exécuter l'app comme des processus **stateless** — pas de sticky session |
| **7. Port binding** | Le service s'expose lui-même via un port (pas de serveur web externe injecté) |
| **8. Concurrency** | Scaler via des processus (process model) |
| **9. Disposability** | Démarrage rapide, arrêt gracieux (SIGTERM → finish current request) |
| **10. Dev/Prod Parity** | Minimiser les différences entre les environnements |
| **11. Logs** | Traiter les logs comme des streams d'événements (stdout) — pas de fichiers |
| **12. Admin processes** | Exécuter les tâches d'admin (migrations) comme des one-off processes |

---

## PILIER XXIV — Réseau Complémentaire

---

### 24.1 TCP vs UDP

**TCP — Transmission Control Protocol**

Protocol **orienté connexion**, fiable, ordonné.

**Three-way handshake** :
```
Client → SYN         → Server
Client ← SYN-ACK     ← Server
Client → ACK         → Server
[Connexion établie]
```

**Four-way termination** :
```
Client → FIN → Server
Client ← ACK ← Server
Client ← FIN ← Server
Client → ACK → Server
[Connexion terminée]
```

**Garanties TCP** :
- **Livraison garantie** : retransmission des segments perdus (via ACK + timeout).
- **Ordre garanti** : les segments sont réordonnés côté récepteur (numéros de séquence).
- **Contrôle de flux** : sliding window — le récepteur contrôle la vitesse d'émission.
- **Contrôle de congestion** : slow start, congestion avoidance (AIMD — Additive Increase Multiplicative Decrease).

**UDP — User Datagram Protocol**

Protocol **sans connexion**, non fiable, non ordonné. Envoie des **datagrams** indépendants sans garantie de livraison, d'ordre, ou de déduplication.

**Avantages** : faible latence (pas de handshake, pas d'ACK), pas d'overhead de contrôle de congestion. Adapté quand la perte est acceptable ou gérée applicativement.

**Cas d'usage UDP** : DNS (requête/réponse courte, retransmission gérée par l'application), VoIP/streaming vidéo (mieux vaut perdre un frame que geler), jeux en ligne, QUIC (HTTP/3 réimplémente la fiabilité sélective au-dessus d'UDP).

---

### 24.2 Modèle OSI — 7 Couches

| Couche | Nom | Protocoles | Unité |
|---|---|---|---|
| **7** | Application | HTTP, HTTPS, FTP, SMTP, DNS, WebSocket | Message |
| **6** | Présentation | TLS/SSL, JPEG, ASCII | — |
| **5** | Session | TLS (sessions), RPC | — |
| **4** | Transport | TCP, UDP | Segment / Datagram |
| **3** | Réseau | IP, ICMP, OSPF, BGP | Paquet |
| **2** | Liaison | Ethernet, WiFi (802.11), ARP | Frame |
| **1** | Physique | Câbles, fibre, radio | Bit |

**Modèle TCP/IP (4 couches)** — utilisé en pratique :
- Application (L7) → Transport (L4) → Internet (L3) → Network Access (L1-L2).

**À quel niveau opèrent les équipements ?**
- **Hub** : L1 (broadcast).
- **Switch** : L2 (adresses MAC).
- **Router** : L3 (adresses IP).
- **Load Balancer L4** : TCP/UDP — based on IP/port.
- **Load Balancer L7** : HTTP — based on URL, headers, cookies.

---

### 24.3 WebSockets vs SSE vs Long Polling vs Webhooks

| Méthode | Direction | Protocole | Cas d'usage |
|---|---|---|---|
| **Short Polling** | Client → Serveur (répété) | HTTP | Simple, inefficace (overhead constant) |
| **Long Polling** | Client → Serveur (tenu ouvert) | HTTP | Requête maintenue ouverte jusqu'à événement |
| **SSE (Server-Sent Events)** | Serveur → Client (unidirectionnel) | HTTP/1.1+ | Notifications serveur → client (dashboards, feeds) |
| **WebSocket** | Bidirectionnel (full-duplex) | WS / WSS | Chat, collaboration temps-réel, jeux |
| **Webhook** | Serveur → Serveur (HTTP push) | HTTP | Notification d'événements entre services (CI/CD, Stripe) |

**WebSocket — mécanisme**
1. Handshake HTTP initial avec `Upgrade: websocket` et `Connection: Upgrade`.
2. Serveur répond `101 Switching Protocols`.
3. Connexion TCP maintenue ouverte — messages bidirectionnels via des **frames** WS (pas des requêtes HTTP).

**SSE vs WebSocket** : SSE est plus simple (HTTP standard, reconnexion automatique, EventSource API) mais unidirectionnel. WebSocket nécessite une bibliothèque mais supporte la communication bidirectionnelle et les formats binaires.

---

### 24.4 CORS — Cross-Origin Resource Sharing

**Définition**
CORS est un mécanisme de sécurité implémenté dans les navigateurs qui restreint les requêtes HTTP vers une **origine différente** de celle de la page (same-origin policy).

**Origine** = protocole + domaine + port. `https://api.example.com` ≠ `https://app.example.com`.

**Mécanisme — Preflight Request**
Pour les requêtes "non-simples" (POST/PUT avec JSON, headers personnalisés), le navigateur envoie d'abord une requête `OPTIONS` **preflight** :

```
OPTIONS /api/data HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type
```

Le serveur répond avec les permissions :
```
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST
Access-Control-Allow-Headers: Content-Type
Access-Control-Max-Age: 86400
```

**`Access-Control-Allow-Origin: *`** : autorise toutes les origines mais **interdit l'envoi de credentials** (cookies, Authorization header).

**Pourquoi CORS existe** : protège les utilisateurs contre les attaques CSRF où un site malveillant ferait des requêtes authentifiées vers un service tiers (banque, API) en utilisant les cookies de session de l'utilisateur.

---

### 24.5 DNS — Domain Name System

**Définition**
DNS est le système qui traduit les **noms de domaine** humains (google.com) en **adresses IP** (142.250.74.46). C'est un système distribué hiérarchique.

**Résolution récursive (mécanisme)**
```
Client → Resolver (ISP/8.8.8.8)
Resolver → Root Server (.) → adresse du serveur .com
Resolver → TLD Server (.com) → adresse du NS de example.com
Resolver → Authoritative NS (example.com) → 93.184.216.34
Resolver → Client [résultat mis en cache selon TTL]
```

**Types d'enregistrements DNS**

| Type | Description |
|---|---|
| **A** | Nom → adresse IPv4 |
| **AAAA** | Nom → adresse IPv6 |
| **CNAME** | Alias → autre nom (ne peut pas être sur le apex/root domain) |
| **MX** | Mail eXchange — serveurs de messagerie du domaine |
| **TXT** | Texte arbitraire (SPF, DKIM, vérification de domaine) |
| **NS** | Nameservers autoritaires du domaine |
| **SOA** | Start of Authority — métadonnées de la zone |

**TTL (Time To Live)** : durée en secondes pendant laquelle un résolveur peut mettre en cache la réponse. TTL bas = changements propagés rapidement mais plus de charge sur les NS.

---

## PILIER XXV — Sécurité Complémentaire & Cloud

---

### 25.1 Hashing vs Encryption vs Encoding

**Distinction fondamentale**

| | **Encoding** | **Encryption** | **Hashing** |
|---|---|---|---|
| **Réversible ?** | Oui (pas de clé) | Oui (avec clé) | **Non** (one-way) |
| **But** | Représentation (transport) | Confidentialité | Intégrité, authentification |
| **Clé ?** | Non | Oui | Non (ou HMAC) |
| **Exemples** | Base64, URL encoding | AES, RSA | SHA-256, bcrypt |

**Encoding** : transforme les données dans un format différent pour le transport ou la représentation. **Pas de sécurité** — n'importe qui peut décoder. (Base64 n'est **pas** du chiffrement.)

**Encryption** : transforme les données en ciphertext illisible sans la clé.
- **Symétrique** (AES) : même clé pour chiffrer et déchiffrer. Rapide, problème de distribution de clé.
- **Asymétrique** (RSA) : clé publique pour chiffrer, clé privée pour déchiffrer. Lente, résout la distribution de clé.

**Hashing** : fonction one-way déterministe qui produit un digest de taille fixe.
- Propriétés : déterministe, rapide, **effet avalanche** (un bit changé → digest complètement différent), résistance aux collisions, résistance à la pré-image.

**Pourquoi ne pas stocker les mots de passe avec SHA-256 ?**
SHA-256 est **trop rapide** → un attaquant peut calculer des milliards de hashes/seconde (rainbow tables, brute force). 

**bcrypt / Argon2** pour les mots de passe :
- **Salting** : ajout d'un sel aléatoire unique par utilisateur avant le hash → neutralise les rainbow tables.
- **Work factor (cost factor)** : paramètre ajustable qui rend le hash **intentionnellement lent** (bcrypt : 100ms par hash typique). Si le hardware devient plus rapide, augmenter le cost factor.
- **bcrypt** : basé sur Blowfish, work factor $2^{cost}$ itérations. Standard de l'industrie.
- **Argon2** (gagnant de PHC — Password Hashing Competition, 2015) : paramétrable en temps, mémoire ET parallélisme. Recommandé pour les nouveaux systèmes.

**HMAC** (Hash-based Message Authentication Code) : $HMAC(K, m) = H((K \oplus opad) || H((K \oplus ipad) || m))$. Combine un hash avec une clé secrète pour garantir l'**intégrité ET l'authenticité** d'un message.

---

### 25.2 HTTPS Certificate Chain & Chain of Trust

**Composants**

- **CA (Certificate Authority)** : entité de confiance qui signe des certificats. Les **Root CAs** (DigiCert, Let's Encrypt, etc.) sont pré-installés dans les OS/navigateurs.
- **Certificat X.509** : document signé numériquement contenant : clé publique du serveur, identité (domaine), CA signataire, dates de validité.
- **Chain of Trust** : Root CA → Intermediate CA → Leaf Certificate (serveur).

**Pourquoi des CAs intermédiaires ?**
La clé privée du Root CA est stockée **hors ligne** (air-gapped) pour maximiser la sécurité. Les CAs intermédiaires, connectées à Internet, signent les certificats de serveur. Si un intermédiaire est compromis, seule sa chaîne est révoquée — pas le Root CA.

**Validation du certificat par le navigateur**
1. Vérifier que la signature du leaf cert est valide avec la clé publique du CA intermédiaire.
2. Vérifier que la signature du CA intermédiaire est valide avec la clé publique du Root CA.
3. Vérifier que le Root CA est dans le trust store du navigateur.
4. Vérifier que la date d'expiration n'est pas dépassée.
5. Vérifier que le certificat n'est pas révoqué (CRL ou OCSP).

**Certificate Pinning** : l'application accepte uniquement un certificat ou une clé publique spécifique — protection contre les attaques MitM même avec un CA compromis.

---

### 25.3 AWS Services Fondamentaux

**Compute**
- **EC2** (Elastic Compute Cloud) : VMs provisionnées. Contrôle total sur le système. Facturation à l'heure/seconde.
- **Lambda** : **Serverless** — exécute du code en réponse à des événements sans gérer de serveur. Facturation à l'invocation (100ms de granularité). **Cold start** : ~100ms-1s si le container est inactif. Limite : 15 min d'exécution max, 10GB RAM max.
- **ECS** (Elastic Container Service) : orchestration de containers Docker (sur EC2 ou Fargate). Fargate = serverless containers.
- **EKS** (Elastic Kubernetes Service) : Kubernetes managé.

**Storage**
- **S3** (Simple Storage Service) : object storage — fichiers de taille quelconque, haute durabilité (11 9s), scalabilité illimitée. Pas un filesystem (pas d'opérations de liste efficaces dans un répertoire). Classes : Standard, Intelligent-Tiering, Glacier (archive).
- **EBS** (Elastic Block Store) : volumes de blocs attachés aux instances EC2 (analogue à un disque dur).
- **EFS** (Elastic File System) : filesystem partagé NFS pour plusieurs instances EC2.

**Database**
- **RDS** : bases relationnelles managées (PostgreSQL, MySQL, Aurora). Multi-AZ, backup automatique, failover.
- **DynamoDB** : NoSQL clé-valeur/document managé, scalabilité automatique, faible latence garantie. Modèle de consistance : eventually consistent par défaut, strongly consistent en option.
- **ElastiCache** : Redis ou Memcached managé.
- **Redshift** : data warehouse colonne pour l'analytique OLAP.

**Messaging**
- **SQS** (Simple Queue Service) : queue de messages. Découplage producteur/consommateur. Garantie at-least-once. Standard (ordre non garanti) vs FIFO (ordre garanti, débit limité).
- **SNS** (Simple Notification Service) : pub/sub. Un message est livré à tous les abonnés (email, SQS, Lambda, HTTP).
- **EventBridge** : event bus managé, routage d'événements par patterns.

---

### 25.4 Serverless — Définition Formelle & Trade-offs

**Définition**
Serverless est un modèle d'exécution où le **fournisseur cloud gère l'infrastructure** (provisionment, scaling, haute disponibilité) et l'utilisateur ne déploie que du **code fonctionnel** (fonctions). La facturation est à l'usage réel (invocations + durée).

**Avantages**
- Pas de gestion de serveur.
- Scaling automatique de 0 à des millions d'invocations.
- Coût nul si pas d'usage.

**Limitations & Trade-offs**

| Limitation | Description |
|---|---|
| **Cold Start** | Le container doit être initialisé si inactif (~100ms à 1s selon le runtime). Problématique pour les APIs à faible latence. |
| **Durée max** | 15 min pour Lambda — interdit les traitements longs. |
| **État** | Stateless par nature — l'état doit être externalisé (S3, DynamoDB, Redis). |
| **Vendor lock-in** | Les abstractions spécifiques au provider (Lambda triggers) couplent le code au cloud. |
| **Debugging complexe** | Distributed tracing nécessaire (AWS X-Ray). |
| **Coût à fort débit** | EC2 peut être moins cher à très haute utilisation continue. |

---

## PILIER XXVI — Réglementation IA

---

### 26.1 AI Act — Règlement Européen sur l'IA

**Définition**
L'**AI Act** (Règlement UE 2024/1689, entré en vigueur août 2024) est le premier cadre réglementaire complet au monde sur l'intelligence artificielle. Il adopte une approche **basée sur le risque** : les obligations sont proportionnelles au niveau de risque de l'application IA.

**4 Catégories de Risque**

**1. Risque Inacceptable (Interdit)**
Applications interdites dès l'entrée en vigueur :
- Systèmes de **notation sociale** par les gouvernements.
- Manipulation psychologique subliminale.
- Exploitation des vulnérabilités (âge, handicap).
- Identification biométrique à distance en temps réel dans les espaces publics (avec exceptions étroites).
- Systèmes d'inférence d'émotions sur le lieu de travail et dans l'éducation.

**2. Risque Élevé (Obligations strictes)**
Exemples : IA pour le recrutement, le crédit, les services essentiels, la sécurité des produits (véhicules autonomes), les infrastructures critiques, l'éducation, la justice, la biométrie.

Obligations :
- **Système de gestion des risques** tout au long du cycle de vie.
- **Gouvernance des données** : données d'entraînement représentatives, documentation complète.
- **Logging et traçabilité** : enregistrement automatique des opérations.
- **Transparence** : information claire pour les utilisateurs et autorités de surveillance.
- **Supervision humaine** : capacité à arrêter/superviser le système.
- **Robustesse, précision et cybersécurité**.
- **Conformité évaluée** avant mise sur le marché (auto-évaluation ou tiers).

**3. Risque Limité (Obligations de transparence)**
Chatbots, deepfakes, IA générative de contenu : obligation d'informer l'utilisateur qu'il interagit avec une IA.

**4. Risque Minimal (Pas d'obligation spécifique)**
Filtres anti-spam, jeux vidéo avec IA, etc. — vaste majorité des applications IA.

**GPAI Models (General Purpose AI)**
Règles spécifiques pour les grands modèles de fondation (GPT-4, Gemini, Claude...) :
- Documentation technique et résumé des données d'entraînement.
- Politique de droits d'auteur.
- Pour les **GPAI à risque systémique** (>$10^{25}$ FLOPS d'entraînement) : red teaming, incidents graves à reporter à la Commission.

**Calendrier d'application**
- Août 2024 : entrée en vigueur.
- Février 2025 : interdictions (risque inacceptable) applicables.
- Août 2026 : règles complètes applicables (risque élevé).

---

### 26.2 RGPD & Impact sur le ML

**Principes clés (Art. 5)**
- **Licéité, loyauté, transparence** : base légale requise pour tout traitement.
- **Limitation des finalités** : données collectées pour des finalités déterminées, ne peuvent être réutilisées autrement.
- **Minimisation des données** : collecter uniquement les données nécessaires.
- **Exactitude** : données tenues à jour.
- **Limitation de la conservation** : pas de conservation indéfinie.
- **Intégrité et confidentialité** : sécurité appropriée.
- **Responsabilité** (Accountability) : le responsable du traitement doit pouvoir démontrer la conformité.

**Privacy by Design & by Default (Art. 25)**
La protection des données doit être intégrée **dès la conception** du système, pas ajoutée après. Par défaut, seules les données strictement nécessaires doivent être traitées.

**Impact direct sur le ML**

- **Droit à l'explication (Art. 22)** : interdiction des décisions entièrement automatisées à impact significatif (crédit, emploi) sans possibilité pour l'individu d'obtenir une **explication humaine**. L'**explainability** (SHAP, LIME) devient une obligation légale.
- **Droit à l'oubli (Art. 17)** : une personne peut demander la suppression de ses données. En ML : si les données d'un individu ont servi à l'entraînement, **machine unlearning** ou réentraînement sans ces données.
- **Droit à la portabilité (Art. 20)** : fournir les données dans un format machine-readable.
- **DPIA** (Data Protection Impact Assessment) : évaluation obligatoire pour les traitements à risque élevé (profilage, biométrie).
- **DPO** (Data Protection Officer) : obligatoire pour les organismes publics et les traitements à grande échelle de données sensibles.

**Bases légales du traitement (Art. 6)**
Consentement, contrat, obligation légale, intérêts vitaux, mission d'intérêt public, **intérêts légitimes** (LIA — Legitimate Interest Assessment requis).

---

---

## PILIER XXVII — Python : Fondamentaux Manquants Critiques

---

### 27.1 LEGB Rule — Résolution des Scopes

**Définition formelle**
Lorsque Python résout un nom de variable, il cherche dans les scopes dans cet ordre exact :

```
L → Local      : le corps de la fonction courante
E → Enclosing  : les fonctions englobantes (closures) — de l'intérieur vers l'extérieur
G → Global     : le module (niveau du fichier .py)
B → Built-in   : le module builtins (len, print, range, etc.)
```

La résolution s'arrête au **premier scope où le nom est trouvé**. Si aucun scope ne contient le nom → `NameError`.

**`global` vs `nonlocal`**

- **`global x`** : déclare que `x` dans la fonction courante fait référence au `x` du scope **global** (module). Permet de modifier une variable globale depuis une fonction.
- **`nonlocal x`** : déclare que `x` fait référence à la variable `x` du scope **englobant le plus proche** (Enclosing). Permet à une closure de modifier la variable capturée.

```python
x = 'global'

def outer():
    x = 'enclosing'

    def inner():
        nonlocal x          # modifie le x de outer()
        x = 'modified'
        print(x)            # 'modified'

    inner()
    print(x)                # 'modified'

outer()
print(x)                    # 'global' — inchangé
```

**Piège classique — Late Binding dans les closures**
```python
fns = [lambda: i for i in range(3)]
fns[0]()   # retourne 2, pas 0 !
# Raison : i est résolu au moment de l'APPEL (E scope), pas de la création
# À ce moment, i == 2 (dernière valeur de la boucle)

# Fix : early binding via argument par défaut
fns = [lambda i=i: i for i in range(3)]
fns[0]()   # retourne 0
```

---

### 27.2 `super()` — Mécanisme Interne

**Définition**
`super()` retourne un **objet proxy** qui délègue les appels de méthode à la **prochaine classe dans le MRO** de la classe courante.

**Mécanisme interne**
`super()` sans arguments (Python 3) utilise deux variables implicites de la cellule de fermeture :
- `__class__` : la classe dans laquelle la méthode est définie (statique, déterminé à la compilation).
- `self` (ou `cls`) : l'instance ou la classe sur laquelle la méthode est appelée.

`super()` consulte le MRO de `type(self)` et retourne un proxy qui commence la recherche **après `__class__`** dans ce MRO.

```python
class A:
    def method(self): print("A")

class B(A):
    def method(self):
        super().method()    # cherche après B dans le MRO de self
        print("B")

class C(A):
    def method(self):
        super().method()    # cherche après C dans le MRO de self
        print("C")

class D(B, C):              # MRO : D → B → C → A → object
    def method(self):
        super().method()    # cherche après D → appelle B.method
        print("D")

D().method()    # A, C, B, D  (ordre MRO inverse, chaque super() avance d'un cran)
```

**Pourquoi `super()` est essentiel dans l'héritage multiple**
Dans le Diamond Problem, `super()` garantit que chaque classe dans le MRO est appelée **exactement une fois**, dans l'ordre C3. Sans `super()`, appeler `B.__init__(self)` directement court-circuite le MRO.

---

### 27.3 `isinstance()` vs `type()`

**Différence fondamentale**

- `type(obj) == SomeClass` : teste l'**identité de type exacte** — `True` uniquement si `obj` est une instance directe de `SomeClass`, pas d'une sous-classe.
- `isinstance(obj, SomeClass)` : teste si `obj` est une instance de `SomeClass` **ou d'une de ses sous-classes** — respecte l'héritage.

```python
class Animal: pass
class Dog(Animal): pass

d = Dog()
type(d) == Animal        # False — type exact est Dog
isinstance(d, Animal)    # True  — Dog est une sous-classe de Animal
isinstance(d, Dog)       # True
```

**Cas surprenant — `bool` est une sous-classe de `int`**
```python
isinstance(True, int)    # True  — bool hérite de int
type(True) == int        # False — type exact est bool
True + 1                 # 2     — les opérations int fonctionnent sur bool
```

**`isinstance()` avec un tuple de types**
```python
isinstance(x, (int, float, complex))  # True si x est de l'un de ces types
```

**Recommandation** : préférer `isinstance()` dans le code applicatif (respecte le polymorphisme). Utiliser `type() ==` uniquement quand on veut explicitement exclure les sous-classes.

---

### 27.4 Truthy / Falsy — Protocole `__bool__` & `__len__`

**Valeurs Falsy exhaustives en Python**
```
False, None, 0, 0.0, 0j,        # constantes et numériques nuls
"", b"", [],  (), {}, set(),     # conteneurs vides
range(0)                         # range vide
```
Tout le reste est **Truthy**.

**Protocole de résolution de la valeur de vérité**
Quand Python évalue `bool(obj)` ou `if obj:`, il cherche dans cet ordre :
1. `obj.__bool__()` → retourne `True` ou `False`.
2. Si absent : `obj.__len__()` → l'objet est Falsy si `len == 0`.
3. Si absent : l'objet est **toujours Truthy** (toute instance d'une classe sans `__bool__` ni `__len__` est truthy).

```python
class MyContainer:
    def __init__(self, items):
        self.items = items
    def __len__(self):
        return len(self.items)

c = MyContainer([])
bool(c)     # False — __len__() retourne 0
if c:       # branche non prise
    ...
```

**Pièges classiques**
```python
# numpy array — ne pas tester avec if arr:
# raise ValueError: truth value of array is ambiguous
# Utiliser : arr.size > 0 ou arr.any() / arr.all()

# 0 est falsy, mais pas None — distinction importante
if result is not None:   # correct pour vérifier l'absence de résultat
if result:               # incorrect si result peut légitimement valoir 0 ou []
```

---

### 27.5 `__slots__` — Mécanisme Interne

**Définition**
`__slots__` est un attribut de classe qui remplace le `__dict__` d'instance par des **descripteurs statiques**. Chaque attribut déclaré dans `__slots__` obtient un slot — un emplacement mémoire fixe dans la structure C de l'instance.

```python
class Point:
    __slots__ = ('x', 'y')    # seuls x et y sont autorisés
    def __init__(self, x, y):
        self.x = x
        self.y = y

p = Point(1, 2)
p.z = 3        # AttributeError — z n'est pas dans __slots__
p.__dict__     # AttributeError — __dict__ n'existe pas
```

**Avantages**
- **Mémoire** : une instance avec `__dict__` coûte ~232 bytes (Python 3.11). Avec `__slots__` : ~56 bytes pour 2 attributs. Gain de ~60-80% pour les classes avec peu d'attributs instanciées massivement.
- **Vitesse d'accès** : accès direct à l'offset mémoire plutôt que lookup dans un dict.

**Limitations**
- Impossible d'ajouter des attributs dynamiques non déclarés.
- L'héritage multiple avec plusieurs classes ayant chacune `__slots__` est problématique (les slots des parents sont hérités, mais les slots définis dans plusieurs bases peuvent conflictuels).
- Si une classe hérite d'une classe sans `__slots__`, elle aura quand même un `__dict__` (le `__slots__` de la sous-classe n'a plus d'effet mémoire complet).
- `pickle` et `copy` nécessitent une attention particulière.

**Quand l'utiliser** : classes de données simples instanciées en très grand nombre (pixels, points, événements de logs).

---

### 27.6 `__getattr__` vs `__getattribute__`

**`__getattribute__(self, name)`**
- Appelé **sur chaque accès d'attribut**, sans exception : `obj.attr` → `obj.__getattribute__('attr')`.
- Mécanisme de base du lookup d'attributs Python.
- **Très dangereux à surcharger** : une erreur crée une récursion infinie (`self.x` dans `__getattribute__` rappelle `__getattribute__`). Utiliser `object.__getattribute__(self, name)` pour l'implémentation de base.

**`__getattr__(self, name)`**
- Appelé **uniquement si l'attribut n'a pas été trouvé** par les mécanismes normaux (ni dans `__dict__`, ni dans la classe, ni dans les bases).
- **Sûr à surcharger** — fallback de dernier recours.
- Utilisation : attributs virtuels, proxies, lazy loading.

```python
class LazyProxy:
    def __init__(self, target):
        object.__setattr__(self, '_target', target)    # évite __setattr__ récursif

    def __getattr__(self, name):
        # Appelé uniquement si name non trouvé normalement
        return getattr(self._target, name)

    def __getattribute__(self, name):
        # Appelé sur CHAQUE accès — override avec précaution
        print(f"Accessing: {name}")
        return object.__getattribute__(self, name)
```

**Ordre de résolution complet d'un accès `obj.attr`**
1. `type(obj).__getattribute__(obj, 'attr')` est appelé.
2. Data descriptors de la classe (et bases) — `property`, `__slots__`.
3. `obj.__dict__['attr']` — dictionnaire de l'instance.
4. Non-data descriptors et autres attributs de la classe.
5. Si toujours pas trouvé : `type(obj).__getattr__(obj, 'attr')`.
6. Sinon : `AttributeError`.

---

### 27.7 `collections` — Module Essentiel

**`Counter`**
Dict spécialisé pour compter des éléments hashables.
```python
from collections import Counter

c = Counter("abracadabra")
c.most_common(3)          # [('a', 5), ('b', 2), ('r', 2)]
c + Counter("abc")        # addition des comptages
c - Counter("abc")        # soustraction (supprime les négatifs)
c['z']                    # 0 — pas de KeyError pour les clés absentes
```

**`defaultdict`**
Dict qui appelle une **factory function** pour créer une valeur par défaut quand une clé absente est accédée.
```python
from collections import defaultdict

graph = defaultdict(list)
graph['A'].append('B')     # pas de KeyError — graph['A'] créé avec []

word_count = defaultdict(int)
for word in words:
    word_count[word] += 1  # int() = 0 par défaut
```
**Différence avec `dict.get(key, default)`** : `defaultdict` **insère** la valeur par défaut dans le dict ; `get()` retourne la valeur sans l'insérer.

**`OrderedDict`**
Dict qui preserve l'ordre d'insertion (redondant en Python 3.7+ où tous les dicts sont ordonnés, mais conserve des méthodes utiles).
```python
from collections import OrderedDict
od = OrderedDict()
od.move_to_end('key')          # déplace la clé en fin (ou début avec last=False)
od.popitem(last=True)          # supprime et retourne le dernier item (LIFO)
```
**Toujours pertinent** : `OrderedDict.__eq__` prend en compte l'ordre (deux `dict` égaux en contenu sont égaux même si ordonnés différemment, pas deux `OrderedDict`).

**`namedtuple`**
Sous-classe de `tuple` avec des champs accessibles par nom.
```python
from collections import namedtuple

Point = namedtuple('Point', ['x', 'y'])
p = Point(1, 2)
p.x         # 1
p[0]        # 1 — accès par index toujours possible
p._asdict() # OrderedDict([('x', 1), ('y', 2)])
p._replace(x=10)  # retourne un nouveau namedtuple avec x=10
```
**Avantage vs tuple** : auto-documentation, accès par nom. **Avantage vs dataclass** : immuable, léger (pas de `__dict__`), interopérable avec le protocole tuple (unpacking, indexation).

**`deque`**
Double-ended queue — O(1) aux deux extrémités.
```python
from collections import deque

d = deque(maxlen=3)    # buffer circulaire de taille fixe
d.append(1)            # droite
d.appendleft(0)        # gauche
d.rotate(1)            # rotation : le dernier devient le premier
```
`maxlen` : quand plein, les nouveaux éléments évincent les anciens du côté opposé. Idéal pour les sliding windows.

---

### 27.8 `itertools` — Combinatoire & Itération Avancée

**Itérateurs infinis**
```python
import itertools

itertools.count(start=0, step=1)      # 0, 1, 2, 3, ...
itertools.cycle([1, 2, 3])            # 1, 2, 3, 1, 2, 3, ...
itertools.repeat(x, n)               # x, x, x, ... (n fois ou infini)
```

**Sélection et découpage**
```python
itertools.islice(iterable, stop)      # lazy slicing sans index
itertools.takewhile(pred, iterable)   # s'arrête dès que pred est False
itertools.dropwhile(pred, iterable)   # skip jusqu'à ce que pred soit False
itertools.filterfalse(pred, iterable) # inverse de filter()
itertools.compress(data, selectors)   # garde data[i] si selectors[i] est True
```

**Combinatoire**
```python
itertools.product('AB', repeat=2)        # AA AB BA BB  (produit cartésien)
itertools.permutations('ABC', r=2)       # toutes les permutations de longueur 2
itertools.combinations('ABC', r=2)       # AB AC BC     (sans répétition, ordonnées)
itertools.combinations_with_replacement  # AA AB AC BB BC CC
```

**Regroupement & combinaison**
```python
itertools.chain([1,2], [3,4])            # 1 2 3 4      (concatène des iterables)
itertools.chain.from_iterable([[1,2],[3,4]])  # identique pour itérable d'iterables
itertools.groupby(data, key=func)        # regroupe les éléments consécutifs par clé
itertools.accumulate([1,2,3,4], func=operator.add)  # 1, 3, 6, 10  (scan)
itertools.starmap(func, iterable)        # map() avec unpacking : func(*args)
itertools.zip_longest(*iterables, fillvalue=None)   # zip qui continue jusqu'au plus long
```

---

### 27.9 `functools.partial` & `functools.reduce`

**`functools.partial`**
Crée une nouvelle fonction en **fixant** certains arguments d'une fonction existante (application partielle).
```python
from functools import partial

def power(base, exp):
    return base ** exp

square = partial(power, exp=2)    # fixe exp=2
cube   = partial(power, exp=3)

square(5)   # 25
cube(3)     # 27

# Différence avec lambda
square_lambda = lambda x: power(x, 2)  # équivalent mais moins explicite
# partial est préféré : conserve __name__, __doc__, __wrapped__, introspectable
```

**`functools.reduce`**
Applique une fonction binaire cumulativement à une séquence, de gauche à droite, pour réduire à une seule valeur.
```python
from functools import reduce
import operator

reduce(operator.add, [1, 2, 3, 4])     # ((1+2)+3)+4 = 10
reduce(operator.add, [], initializer=0) # initializer requis si séquence vide
```

---

### 27.10 `enum` — Énumérations

**Définition**
`enum.Enum` crée des **constantes nommées typées** — alternative aux constantes magiques (strings/entiers arbitraires).

```python
from enum import Enum, IntEnum, Flag, auto

class Color(Enum):
    RED   = 1
    GREEN = 2
    BLUE  = 3

Color.RED            # <Color.RED: 1>
Color.RED.name       # 'RED'
Color.RED.value      # 1
Color(1)             # <Color.RED: 1>   — lookup par valeur
Color['RED']         # <Color.RED: 1>   — lookup par nom

class Permission(Flag):
    READ    = auto()   # 1
    WRITE   = auto()   # 2
    EXECUTE = auto()   # 4
    # Flag supporte les combinaisons bit à bit
    ALL = READ | WRITE | EXECUTE

Permission.READ | Permission.WRITE    # <Permission.READ|WRITE: 3>
```

**`IntEnum`** : sous-classe à la fois de `Enum` et `int` → utilisable partout où un int est attendu. Utile pour l'interopérabilité avec du code legacy.

**`auto()`** : génère automatiquement des valeurs (entiers croissants par défaut pour `Enum`, puissances de 2 pour `Flag`).

---

### 27.11 `match/case` — Structural Pattern Matching (Python 3.10+)

**Définition**
`match/case` n'est **pas** un simple switch/case — c'est du **pattern matching structurel** qui déstructure et filtre simultanément.

```python
# Matching sur type et structure
def process(command):
    match command:
        case {"action": "quit"}:
            return "Quitting"
        case {"action": "move", "direction": d}:
            return f"Moving {d}"       # d est capturé
        case {"action": action}:
            return f"Unknown action: {action}"
        case _:                        # wildcard — cas par défaut
            return "Invalid command"

# Matching sur types
match point:
    case Point(x=0, y=0):
        print("Origin")
    case Point(x=0, y=y):             # capture y
        print(f"Y-axis at y={y}")
    case Point(x=x, y=y):
        print(f"({x}, {y})")

# Guard clause
match value:
    case x if x > 100:                # guard
        print(f"Large: {x}")
    case x:
        print(f"Small: {x}")
```

**Différence avec `if/elif`**
- **Déstructuration** : décompose automatiquement les structures (dicts, classes, tuples).
- **Capture de variables** : lie des parties de la structure à des variables locales.
- **Pattern guards** : conditions supplémentaires via `if`.
- **Pas d'évaluation d'expressions** : les patterns sont des structures, pas des expressions booléennes.

---

### 27.12 `threading` — Primitives de Synchronisation

**`threading.Lock`**
Verrou d'exclusion mutuelle — seul le thread qui l'a acquis peut le libérer.
```python
lock = threading.Lock()
with lock:                  # acquire() / release() automatiques
    shared_resource += 1    # section critique
```

**`threading.RLock` (Reentrant Lock)**
Verrou **réentrant** — le **même thread** peut l'acquérir plusieurs fois sans se bloquer. Maintient un compteur d'acquisitions. Utilisé quand une méthode synchronisée appelle une autre méthode synchronisée du même objet.
```python
rlock = threading.RLock()
with rlock:
    with rlock:   # Lock ordinaire se bloquerait ici — RLock ne se bloque pas
        ...
```

**`threading.Semaphore`**
Compteur atomique. Permet à **N threads simultanément** d'accéder à une ressource.
```python
sem = threading.Semaphore(3)   # 3 threads max simultanément
with sem:
    access_limited_resource()
```

**`threading.Event`**
Flag de signalisation entre threads — un thread attend qu'un autre signale.
```python
event = threading.Event()

# Thread A — attend
event.wait()                   # bloque jusqu'à event.set()

# Thread B — signale
event.set()                    # débloque tous les threads en wait
event.clear()                  # remet à l'état non-signalé
```

**`threading.Condition`**
Combine un verrou et la capacité d'attendre/signaler des conditions complexes.
```python
cond = threading.Condition()
with cond:
    while not condition_satisfied():
        cond.wait()            # libère le lock et attend notify
    do_work()

# Dans un autre thread :
with cond:
    set_condition()
    cond.notify_all()          # réveille tous les threads en wait
```

**`threading.Barrier`**
Tous les threads doivent atteindre la barrière avant qu'aucun ne continue — synchronisation de point de rendez-vous.

**`queue.Queue`** (thread-safe) : file FIFO (aussi LIFO et PriorityQueue) pour communiquer entre threads sans Lock explicite. `put()`, `get()`, `task_done()`, `join()`.

---

### 27.13 `pickle` — Sérialisation Python

**Définition**
`pickle` est le module de **sérialisation native Python** — convertit des objets Python en flux d'octets (sérialisation) et inversement (désérialisation).

```python
import pickle

data = {"key": [1, 2, 3], "model": my_sklearn_model}

# Sérialisation
with open("data.pkl", "wb") as f:
    pickle.dump(data, f, protocol=pickle.HIGHEST_PROTOCOL)

# Désérialisation
with open("data.pkl", "rb") as f:
    data = pickle.load(f)
```

**Protocoles** : 0 (ASCII), 1-4 (binaires progressivement plus efficaces), 5 (Python 3.8+, supporte les buffers out-of-band). `HIGHEST_PROTOCOL` = toujours le plus récent disponible.

**Ce qui est picklable** : fonctions et classes définies au niveau module (par nom), instances, types built-in.
**Ce qui n'est pas picklable** : lambdas, closures, générateurs, connexions réseau, fichiers ouverts, threads.

**`__reduce__` et `__getstate__`/`__setstate__`** : protocoles pour personnaliser la sérialisation.

**AVERTISSEMENT DE SÉCURITÉ CRITIQUE**
`pickle.load()` exécute du code arbitraire pendant la désérialisation — **ne jamais désérialiser des données provenant d'une source non fiable**. Vecteur d'attaque Remote Code Execution.

---

## PILIER XXVIII — ML Complémentaire Critique

---

### 28.1 BatchNorm vs LayerNorm

**Batch Normalization (Ioffe & Szegedy, 2015)**

Normalise les activations **par feature, à travers le batch** :

$$\hat{x}^{(k)} = \frac{x^{(k)} - \mu_B^{(k)}}{\sqrt{\sigma_B^{(k)2} + \epsilon}}$$

où $\mu_B$ et $\sigma_B^2$ sont calculés sur le **batch courant**, pour la feature $k$.

- Deux paramètres apprenables par feature : $\gamma^{(k)}$ (scale) et $\beta^{(k)}$ (shift).
- **Problème** : dépend de la taille du batch (instable avec des petits batchs). Comportement différent en training (stats du batch) vs inference (moving average des stats d'entraînement). **Impossible à appliquer sur les séquences de longueur variable**.

**Layer Normalization (Ba et al., 2016)**

Normalise les activations **par exemple, à travers toutes les features** de la couche :

$$\hat{x}_i = \frac{x_i - \mu_L}{\sqrt{\sigma_L^2 + \epsilon}}$$

où $\mu_L$ et $\sigma_L^2$ sont calculés sur **toutes les features de l'exemple courant**.

- **Indépendant du batch** → comportement identique en training et inference.
- **Fonctionne sur les séquences** de longueur variable.
- Standard dans les **Transformers** (BERT, GPT, T5) — appliqué après l'attention et le FFN (Pre-LN dans les architectures modernes).

**Pourquoi BatchNorm ne fonctionne pas dans les Transformers ?**
Les séquences ont des longueurs variables → les stats de batch seraient calculées sur des positions de tokens différentes → statistiques incohérentes. LayerNorm normalise par exemple indépendamment, ce qui est naturel pour les séquences.

**Instance Norm** : normalise par feature et par exemple (Images). **Group Norm** : normalise sur des groupes de features — compromis entre BatchNorm et LayerNorm.

---

### 28.2 Learning Rate Scheduling

**Pourquoi le LR Scheduling ?**
Un LR fixe est rarement optimal : trop grand → instabilité, pas de convergence ; trop petit → convergence lente, possible blocage dans des minima peu profonds. Le scheduling adapte le LR au cours de l'entraînement.

**Strategies courantes**

**Warmup Linéaire + Cosine Decay** (standard Transformers) :
- **Phase warmup** (ex : 4% des steps) : LR augmente linéairement de 0 → LR max.
- **Phase decay** : LR suit une décroissance en cosinus → LR min.

**Pourquoi le warmup pour Adam ?**
Au début de l'entraînement, les estimations des moments ($m_t$, $v_t$) sont initialisées à 0 et très bruitées. Un LR élevé dès le début avec des gradients bruités → mises à jour instables des poids. Le warmup donne le temps aux estimations de moments de se stabiliser.

**Step Decay** : réduction du LR par un facteur (ex: /10) à des epochs prédéfinis.

**Reduce on Plateau** : réduit le LR quand la métrique de validation stagne (patience = N epochs).

**Cyclical Learning Rate (CLR)** : alterne entre LR min et LR max en cycles. Avantage : peut échapper aux saddle points.

**One Cycle Policy** : warmup jusqu'au LR max, puis décroissance. Très efficace avec `max_lr` bien calibré.

---

### 28.3 Transfer Learning — Feature Extraction vs Fine-tuning

**Transfer Learning**
Réutiliser un modèle pré-entraîné sur une grande tâche source (ImageNet, texte web) pour une tâche cible avec moins de données.

**Feature Extraction**
- **Geler** tous les poids du modèle pré-entraîné.
- Ajouter une nouvelle tête (classification head) entraînable.
- Le backbone pré-entraîné est utilisé comme extracteur de features fixes.
- **Quand** : petit dataset cible, similarité forte avec la tâche source.

**Fine-tuning**
- **Dégeler** tout ou partie du modèle pré-entraîné.
- Entraîner avec un LR faible (typiquement 10-100x plus petit que l'entraînement original).
- **Quand** : dataset cible plus grand, ou tâche suffisamment différente de la source.

**Stratégie progressive (Discriminative Fine-tuning)**
Différents LR par couche : couches profondes (bas-niveau) → LR très faible ; couches finales (haut-niveau) → LR plus élevé. Les représentations bas-niveau (bords, textures) sont universelles ; les représentations haut-niveau sont plus spécifiques à la tâche.

**Catastrophic Forgetting**
En fine-tunant trop agressivement, le modèle "oublie" ses connaissances générales. Solutions : LoRA (geler les poids de base), EWC (Elastic Weight Consolidation — pénalise les modifications des poids importants pour la tâche source).

---

### 28.4 SHAP & LIME — Explicabilité (XAI)

**Contexte légal** : Art. 22 RGPD et AI Act risque élevé imposent une capacité d'explication des décisions automatisées.

#### SHAP (SHapley Additive exPlanations — Lundberg & Lee, 2017)

**Fondement théorique — Valeurs de Shapley (théorie des jeux)**
La valeur de Shapley $\phi_i$ d'un joueur $i$ dans un jeu coopératif est sa **contribution marginale moyenne**, calculée sur toutes les coalitions possibles de joueurs :

$$\phi_i = \sum_{S \subseteq N \setminus \{i\}} \frac{|S|!(|N|-|S|-1)!}{|N|!} [v(S \cup \{i\}) - v(S)]$$

En ML : les "joueurs" sont les features, le "jeu" est la prédiction du modèle. $\phi_i$ mesure la contribution de la feature $i$ à la prédiction, en moyennant sur toutes les coalitions de features.

**Propriétés garanties**
- **Efficacité** : $\sum_i \phi_i = f(x) - \mathbb{E}[f(X)]$ (la somme des contributions = écart à la prédiction moyenne).
- **Symétrie** : deux features identiques ont la même valeur de Shapley.
- **Nullité** : une feature qui ne change aucune prédiction a $\phi_i = 0$.
- **Additivité** : SHAP respecte la décomposition additive.

**Variantes** : TreeSHAP (O(n) exact pour arbres), KernelSHAP (model-agnostic), DeepSHAP (réseaux de neurones).

**Interprétation** : `shap.summary_plot()` montre l'importance globale ; `shap.waterfall_plot()` explique une prédiction individuelle.

#### LIME (Local Interpretable Model-agnostic Explanations — Ribeiro et al., 2016)

**Principe** : approximer localement le comportement du modèle autour d'un exemple par un **modèle interprétable** (régression linéaire).

**Algorithme**
1. Pour l'exemple à expliquer $x$, générer $N$ perturbations $z'$ (désactiver aléatoirement des features/tokens).
2. Obtenir la prédiction du modèle sur ces perturbations.
3. Pondérer les perturbations par leur proximité à $x$ (kernel de proximité).
4. Entraîner une régression linéaire sur les perturbations pondérées → coefficients = importance locale des features.

**Différence SHAP vs LIME**
- SHAP : fondement théorique solide (Shapley values), **global ET local**, consistant, mais plus coûteux.
- LIME : heuristique locale, **uniquement local**, plus rapide, instable (résultats peuvent varier entre exécutions car perturbations aléatoires).

---

### 28.5 Curse of Dimensionality

**Définition**
Ensemble de phénomènes contre-intuitifs qui apparaissent quand la dimension de l'espace des features augmente, rendant les algorithmes basés sur la distance ou la densité inefficaces.

**Phénomène 1 — Explosion du volume**
Le volume d'une boule de rayon $r$ en dimension $d$ est $V_d(r) \propto r^d / \Gamma(d/2 + 1)$. La fraction du volume d'un hypercube contenu dans une boule inscrite tend vers 0 quand $d \to \infty$. La **plupart du volume est aux coins**, pas au centre.

**Phénomène 2 — Concentration des distances**
En haute dimension, les distances entre points tendent à se concentrer :
$$\frac{\text{dist}_{max} - \text{dist}_{min}}{\text{dist}_{min}} \to 0 \text{ quand } d \to \infty$$
Tous les points semblent équidistants → le KNN dégénère car "proche" et "loin" perdent leur sens.

**Phénomène 3 — Sparsité des données**
Pour couvrir 10% de l'espace en dimension 1 avec des données uniformes → 10% des données. En dimension 10 → 80% des données. Les données deviennent exponentiellement rares.

**Impact sur la similarité cosinus en haute dimension**
La similarité cosinus est moins affectée que la distance L2 (elle mesure l'angle, pas la magnitude). C'est pour cela qu'elle est préférée pour les embeddings de haute dimension en NLP. Mais même cosinus peut souffrir en très haute dimension si les vecteurs ne sont pas bien distribués.

**Solutions**
- Réduction de dimensionnalité : PCA, UMAP, t-SNE.
- Feature selection : retirer les features non informatives.
- Régularisation : L1 induit la sparsité (sélection implicite de features).
- Modèles adaptés : arbres de décision moins sensibles à la curse.

---

### 28.6 Bagging vs Boosting vs Stacking

**Bagging (Bootstrap AGGregating)**
- Entraîne $T$ modèles **en parallèle**, chacun sur un **bootstrap sample** (tirage avec remise) du dataset.
- Prédiction par **vote majoritaire** (classification) ou **moyenne** (régression).
- **Réduction de la variance** — les erreurs non-corrélées des modèles s'annulent en moyenne.
- Exemple : **Random Forest** (Bagging + feature subsampling).

**Boosting**
- Entraîne $T$ modèles **séquentiellement**, chaque modèle se concentrant sur les erreurs du précédent.
- À chaque étape, les exemples mal classés reçoivent un poids plus élevé.
- **Réduction du biais** — chaque modèle corrige les insuffisances du précédent.
- Exemples : **AdaBoost** (pondération des exemples), **Gradient Boosting** (optimisation du résidu), **XGBoost**.

**Stacking (Stacked Generalization)**
- Entraîne plusieurs modèles de **niveau 0** (base learners) avec des algorithmes différents.
- Les prédictions des base learners sur un held-out set constituent les features d'un **méta-learner** (niveau 1).
- Le méta-learner apprend à combiner les prédictions des base learners.
- **Exploite les forces complémentaires** des différents algorithmes.
- Risque de data leakage si le méta-learner est entraîné sur les mêmes données que les base learners (→ utiliser la cross-validation pour générer les features de niveau 1).

| | Bagging | Boosting | Stacking |
|---|---|---|---|
| **Ordre d'entraînement** | Parallèle | Séquentiel | Parallèle (L0) + Séquentiel (L1) |
| **Objectif** | Réduire la variance | Réduire le biais | Combiner les forces |
| **Modèles** | Homogènes | Homogènes | Hétérogènes |
| **Risque** | — | Overfitting si trop d'itérations | Data leakage |

---

## PILIER XXIX — Architecture & BDD Complémentaire

---

### 29.1 Deadlock, Livelock & Starvation

**Deadlock — Définition**
Situation où deux ou plusieurs threads/processus s'attendent mutuellement indéfiniment, chacun détenant une ressource dont l'autre a besoin.

**4 Conditions de Coffman (nécessaires ET suffisantes)**
1. **Mutual Exclusion** : au moins une ressource est non-partageble — seul un thread peut l'utiliser à la fois.
2. **Hold and Wait** : un thread détient au moins une ressource et attend d'en acquérir d'autres.
3. **No Preemption** : les ressources ne peuvent pas être forcément retirées à un thread — libération volontaire uniquement.
4. **Circular Wait** : il existe un cycle de threads $T_1 \to T_2 \to ... \to T_n \to T_1$ où chaque thread attend une ressource détenue par le suivant.

**Prévention** : briser une des 4 conditions.
- Briser Circular Wait : imposer un **ordre total sur les ressources** — toujours acquérir dans le même ordre (lock ordering).
- Briser Hold and Wait : acquérir toutes les ressources d'un coup ou ne pas en tenir si impossible.
- Timeout : `Lock.acquire(timeout=5)` → si non acquis en 5s, libérer et réessayer.

**Détection BDD** : les SGBD (PostgreSQL, MySQL) détectent les deadlocks via un graphe de dépendance d'attente (wait-for graph) et tuent automatiquement une transaction victime.

**Livelock**
Les threads sont actifs (pas bloqués) mais changent d'état en réponse aux autres, sans avancer. Analogue : deux personnes dans un couloir qui se déplacent chacune pour laisser passer l'autre mais se retrouvent toujours face à face.

**Starvation**
Un thread est indéfiniment retardé car d'autres threads monopolisent la ressource. Solution : **fair scheduling** (file FIFO), `threading.Semaphore` avec ordre d'acquisition garanti.

**Race Condition**
Le comportement d'un programme dépend de l'**ordre d'exécution relatif des threads** — résultat imprévisible et non-déterministe. Différent du deadlock : les threads progressent, mais produisent des résultats incorrects.

---

### 29.2 OLAP vs OLTP

**OLTP — Online Transaction Processing**
Optimisé pour les **transactions courtes** : insertions, mises à jour, suppressions d'enregistrements individuels. Charge de travail typique des applications opérationnelles (e-commerce, banque, CRM).

**OLAP — Online Analytical Processing**
Optimisé pour les **requêtes analytiques complexes** sur de grands volumes de données historiques. Agrégations, GROUP BY, JOINs multi-tables, analyses temporelles. Charge typique des data warehouses et BI.

| Critère | OLTP | OLAP |
|---|---|---|
| **Opérations** | INSERT, UPDATE, DELETE (courtes) | SELECT avec agrégations complexes |
| **Volume de données** | Faible à moyen par requête | Très large (scan de millions de lignes) |
| **Modèle de données** | Normalisé (3NF) — minimise la redondance | Dénormalisé (schéma en étoile/flocon) — minimise les JOINs |
| **Stockage** | **Row-store** : une ligne = une unité | **Column-store** : une colonne = une unité |
| **Indexing** | B-Tree sur clés primaires/étrangères | Bitmap indexes, columnar compression |
| **ACID** | Fort | Eventual consistency acceptable |
| **Exemples** | PostgreSQL, MySQL | Snowflake, BigQuery, Redshift, ClickHouse |

**Pourquoi le column-store pour OLAP ?**
Une requête analytique lit typiquement 3-5 colonnes sur des millions de lignes. Un row-store charge toute la ligne en mémoire pour accéder à une colonne. Un column-store charge uniquement les colonnes nécessaires → réduction drastique des I/O + compression très efficace (valeurs similaires adjacentes).

**Schéma en étoile (Star Schema)**
Table de faits centrale (métriques mesurables : ventes, clicks) entourée de tables de dimensions (date, produit, client). Dénormalisé intentionnellement pour éviter les JOINs coûteux.

---

### 29.3 Sharding vs Replication

**Replication**
**Copie** des mêmes données sur plusieurs nœuds.

- **Master-Slave (Primary-Replica)** : toutes les écritures vont au master, les lectures peuvent aller aux replicas. Le master réplique vers les replicas de manière synchrone ou asynchrone.
- **Multi-Master** : plusieurs nœuds acceptent les écritures → risques de conflits de mise à jour.
- **Avantages** : haute disponibilité (failover si le master tombe), scalabilité des lectures.
- **Limitations** : ne résout pas la scalabilité des **écritures** (toujours bornée par le master).

**Sharding (Partitionnement horizontal)**
**Division** des données entre plusieurs nœuds — chaque nœud contient un **sous-ensemble** des données.

- **Range-based sharding** : par plage de valeurs (users A-M → shard 1, N-Z → shard 2). Risque de hot spot si une plage est très sollicitée.
- **Hash-based sharding** : `shard = hash(key) % N`. Distribution uniforme mais les range queries nécessitent d'interroger tous les shards.
- **Directory-based** : une table de correspondance (lookup service) mappe chaque clé à son shard. Flexible mais point de défaillance.

**Consistent Hashing**
Algorithme de partitionnement qui minimise le **remapping** quand des nœuds sont ajoutés/supprimés. Les nœuds et les clés sont placés sur un **anneau circulaire** (modulo $2^{32}$). Une clé est assignée au premier nœud rencontré dans le sens horaire. Ajout/suppression d'un nœud → uniquement les clés entre deux nœuds adjacents sont remappées (environ $K/N$ clés, pas $K$). Utilisé par Cassandra, DynamoDB, Redis Cluster.

---

### 29.4 Patterns GoF Restants

#### Facade (Structurel)

**Définition** : fournit une **interface simplifiée et unifiée** à un ensemble de sous-systèmes complexes.

```python
class VideoEncoder: ...
class AudioDecoder: ...
class SubtitleParser: ...
class FileWriter: ...

class VideoConversionFacade:
    """Interface simple masquant la complexité des sous-systèmes"""
    def convert(self, input_file, output_format):
        video = VideoEncoder().encode(input_file)
        audio = AudioDecoder().decode(input_file)
        subs = SubtitleParser().parse(input_file)
        return FileWriter().write(video, audio, subs, output_format)
```

**Différence avec Adapter** : l'Adapter adapte une **interface existante** pour la rendre compatible ; la Facade **simplifie** un ensemble de sous-systèmes derrière une nouvelle interface.

#### Template Method (Comportemental)

**Définition** : définit le **squelette d'un algorithme** dans une méthode de la superclasse, en délégant certaines étapes aux sous-classes. Les sous-classes peuvent redéfinir certaines étapes sans changer la structure globale.

```python
class DataPipeline(ABC):
    def run(self):          # Template Method — squelette fixe
        data = self.extract()
        data = self.transform(data)
        self.load(data)

    @abstractmethod
    def extract(self): ...

    def transform(self, data):  # étape avec implémentation par défaut
        return data

    @abstractmethod
    def load(self, data): ...

class CSVPipeline(DataPipeline):
    def extract(self): return read_csv(...)
    def load(self, data): write_db(data)
```

**Différence avec Strategy** : Template Method utilise l'**héritage** — la structure est dans la superclasse, les variations dans les sous-classes. Strategy utilise la **composition** — l'algorithme entier est encapsulé et interchangeable.

---

---

## PILIER XXX — JIT & Optimisation de Performance Python

---

### 30.1 JIT — Just-In-Time Compilation

**Définition formelle**
Le **JIT** (Just-In-Time compilation) est une technique d'exécution hybride entre interprétation et compilation statique : le code est compilé en **code machine natif** au moment de l'exécution (runtime), et non à l'avance (ahead-of-time). La compilation est déclenchée dynamiquement, typiquement après détection du **code chaud** (hot path — code exécuté fréquemment).

**Pourquoi Python pur est lent**
CPython est un interpréteur de **bytecode** : le code source est compilé en bytecode Python (`.pyc`), puis le bytecode est interprété instruction par instruction par la machine virtuelle CPython. Chaque instruction bytecode correspond à plusieurs instructions machine. Overhead : dispatch de l'interpréteur, boxing/unboxing des objets, lookup dynamique des types à chaque opération.

**AOT vs JIT vs Interprétation**

| | Interprétation (CPython) | AOT (C, Rust) | JIT (PyPy, Numba) |
|---|---|---|---|
| **Moment de compilation** | Jamais (bytecode → VM) | Avant l'exécution | Pendant l'exécution |
| **Démarrage** | Rapide | Lent (compilation) | Moyen |
| **Performance runtime** | Lente | Maximale | Proche du natif sur les hot paths |
| **Flexibilité** | Totale (dynamisme Python) | Statique | Partielle (contraintes de typage) |

---

### 30.2 PyPy — JIT pour Python

**Définition**
PyPy est une **implémentation alternative de Python** (compatible CPython) avec un compilateur JIT intégré. Écrit en RPython (sous-ensemble restreint de Python).

**Mécanisme interne — Tracing JIT**
1. PyPy interprète le bytecode normalement au départ.
2. Il **profile** l'exécution et détecte les **traces chaudes** (boucles exécutées > seuil, typiquement 1039 fois).
3. Il enregistre la **trace** d'exécution (séquence d'opérations concrètes avec types réels).
4. Il **compile** cette trace en code machine natif optimisé (avec les types inlinés).
5. Les exécutions suivantes utilisent le code machine compilé directement.
6. Si les **guards** (hypothèses sur les types) échouent → retour à l'interprétation (**deoptimization**).

**Optimisations clés de PyPy**
- **Type specialization** : si une variable est toujours `int`, PyPy génère du code machine sans boxing.
- **Inlining** : les fonctions courtes appelées fréquemment sont inlinées dans la trace.
- **Escape analysis** : objets qui ne s'échappent pas de la trace → alloués sur la stack (pas le heap).
- **Loop unrolling** : déroulage des boucles courtes.

**Performances typiques** : 4-10x plus rapide que CPython sur du code Python pur CPU-bound. Moins efficace sur du code qui appelle massivement des C-extensions (numpy, etc.) — le JIT ne peut pas optimiser à travers les barrières FFI.

**Limitations**
- Compatibilité : pas toujours 100% compatible CPython (certaines C-extensions ne fonctionnent pas).
- Temps de warmup : la compilation JIT prend du temps — inefficace pour les scripts courts.
- GIL : PyPy a aussi un GIL (STM — Software Transactional Memory — en développement pour l'éliminer).

---

### 30.3 Numba — JIT pour le Code Numérique

**Définition**
Numba est un compilateur JIT **AOT/JIT hybride** pour Python, spécialisé dans le code numérique (tableaux NumPy, boucles mathématiques). Il utilise **LLVM** comme backend pour générer du code machine optimisé.

**Mécanisme interne**
```
@numba.jit → Python bytecode → Numba IR → LLVM IR → Code machine natif
```
1. À la première invocation de la fonction décorée, Numba **inspecte les types des arguments**.
2. Il génère une **représentation intermédiaire** spécialisée pour ces types.
3. LLVM optimise et compile en code machine natif.
4. Le code compilé est **mis en cache** — les appels suivants avec les mêmes types vont directement au code machine.
5. Si les types changent → **recompilation** pour la nouvelle signature de types.

**Modes de compilation**

```python
import numba

# nopython=True (recommandé) : tout doit être compilable sans retour à l'interpréteur Python
# Si un type non supporté est rencontré → TypingError au lieu d'une dégradation silencieuse
@numba.jit(nopython=True)    # alias : @numba.njit
def fast_sum(arr):
    total = 0.0
    for x in arr:
        total += x
    return total

# cache=True : persist le code compilé sur disque → pas de warmup au prochain lancement
@numba.njit(cache=True)
def compute(x): ...

# parallel=True : parallélise automatiquement les boucles (OpenMP-like)
@numba.njit(parallel=True)
def parallel_compute(arr):
    return numba.prange(len(arr))   # prange = parallel range
```

**GPU avec Numba CUDA**
```python
@numba.cuda.jit
def kernel(arr, result):
    i = numba.cuda.grid(1)
    if i < arr.shape[0]:
        result[i] = arr[i] ** 2
```

**Quand Numba est efficace**
- Boucles Python sur des tableaux numériques (ce que NumPy ne peut pas vectoriser facilement).
- Algorithmes avec accès mémoire complexes (non-contiguous, conditions dans les boucles).
- Code mathématique avec beaucoup d'opérations scalaires.

**Quand Numba n'aide pas**
- Code déjà vectorisé avec NumPy (NumPy appelle déjà du C optimisé en interne).
- Code I/O-bound.
- Manipulation de chaînes, dicts Python complexes (types non supportés par nopython).

**Lien SITA** : sur le module In-Flight Connectivity, Numba a réduit le temps de calcul des algorithmes de traitement de données de vol en compilant les boucles numériques en code machine natif — critique pour traiter 20 000 vols en production.

---

### 30.4 Cython — AOT Compilation

**Définition**
Cython est un **surensemble de Python** qui se compile en C, puis en code machine. Il permet d'annoter le code Python avec des **types statiques** pour générer du code C performant.

```python
# fichier .pyx
def fast_function(double[:] arr):   # typed memoryview
    cdef int i
    cdef double total = 0.0
    for i in range(arr.shape[0]):
        total += arr[i]
    return total
```

**Différence Cython vs Numba**
- **Cython** : AOT, nécessite une étape de compilation (`.pyx` → `.c` → `.so`), intégration facile dans des packages Python distribués, supporte toutes les fonctionnalités Python.
- **Numba** : JIT, aucune étape de build, fonctionne directement avec `@decorator`, limité aux types numériques.

---

### 30.5 Python 3.13 — JIT Expérimental (PEP 744)

Python 3.13 introduit un **compilateur JIT copy-and-patch** expérimental (désactivé par défaut, `--enable-experimental-jit` à la compilation de l'interpréteur).

**Mécanisme copy-and-patch** : différent du tracing JIT de PyPy. Copie des templates de code machine pré-compilés et les "patche" avec les valeurs concrètes (adresses, constantes). Plus simple qu'un vrai JIT, mais déjà mesurable sur certains benchmarks.

**Adaptive Interpreter (Python 3.11+)** : précurseur du JIT — le bytecode se **spécialise** à chaud en fonction des types observés (`LOAD_FAST` → `LOAD_FAST_AND_CLEAR`, `BINARY_OP` → `BINARY_OP_ADD_INT`). Chaque opcode peut se transformer en une version optimisée pour les types qu'il rencontre le plus souvent.

---

### 30.6 Autres Techniques d'Optimisation Python

**Vectorisation NumPy**
Remplacer les boucles Python par des opérations NumPy qui appellent du C optimisé (BLAS, LAPACK, MKL/OpenBLAS) en interne. La vectorisation élimine l'overhead de l'interpréteur pour les opérations tableau.

```python
# Lent — boucle Python
result = [x**2 + 2*x for x in arr]

# Rapide — vectorisé NumPy (C interne)
result = arr**2 + 2*arr
```

**`__slots__`** : réduction mémoire (déjà couvert — Pilier 27.5).

**Profiling — identifier les vrais bottlenecks**
- **`cProfile`** : profiler de la stdlib — mesure le temps CPU par fonction.
- **`line_profiler`** (`@profile`) : profiling ligne par ligne.
- **`memory_profiler`** : profiling mémoire ligne par ligne.
- **`py-spy`** : profiler sampling externe (pas besoin de modifier le code), faible overhead en production.

**Règle fondamentale** : **ne pas optimiser sans profiler**. La plupart du temps, 90% du temps CPU est dans 10% du code — identifier ce 10% avant d'optimiser.

**Termes académiques associés** : JIT, AOT, tracing JIT, method JIT, bytecode, LLVM IR, specializing adaptive interpreter, copy-and-patch JIT, deoptimization, guard, hot path, cold path, warmup, GIL-free (PEP 703), RPython, cProfile, BLAS/LAPACK.

---

---

## PILIER XXXI — Algorithmique & CS Fondamentaux "École"

---

### 31.1 Stack vs Heap — Gestion Mémoire

**Stack (Pile)**
- Zone mémoire à taille fixe, allouée par le système d'exploitation pour chaque thread.
- Fonctionne en **LIFO** — Last In, First Out.
- Stocke : variables locales, paramètres de fonctions, adresses de retour.
- Allocation/désallocation **automatique** et **déterministe** — quand une fonction retourne, son stack frame est libéré.
- Accès très rapide (simple décrément/incrément du stack pointer).
- Taille limitée (typiquement 1-8 MB) → **Stack Overflow** si dépassée (récursion infinie, objets locaux trop grands).

**Heap (Tas)**
- Zone mémoire dynamique, gérée par l'allocateur (malloc/free en C, GC en Python).
- Allocation **manuelle** (C) ou **automatique via GC** (Python, Java).
- Stocke : objets alloués dynamiquement, dont la durée de vie est inconnue à la compilation.
- Accès plus lent (indirection via pointeur, fragmentation possible).
- Taille limitée uniquement par la RAM disponible.

**En Python spécifiquement**
- Tous les objets Python vivent sur le **heap** — même les petits entiers (sauf le cache -5..256).
- Les variables Python sont des **références** (pointeurs) vers des objets sur le heap.
- Les variables locales (les références elles-mêmes) vivent dans le **frame d'exécution** sur le heap aussi (PyFrameObject est un objet Python alloué sur le heap).

**Stack Overflow en Python**
```python
import sys
sys.getrecursionlimit()    # 1000 par défaut
sys.setrecursionlimit(10000)
```
La profondeur de récursion est limitée par la taille du stack C sous-jacent.

---

### 31.2 Récursion — Mécanisme, Tail Recursion, Itération

**Mécanisme de la récursion**
Chaque appel récursif crée un nouveau **stack frame** contenant les variables locales et l'adresse de retour. La pile d'appels grandit jusqu'au cas de base, puis se dépile.

```
factorial(4)
  → factorial(3)
      → factorial(2)
          → factorial(1)
              → return 1
          → return 2
      → return 6
  → return 24
```

**Complexité** : $O(n)$ en espace pour une récursion de profondeur $n$ (chaque frame occupe de la mémoire).

**Tail Recursion (Récursion Terminale)**
La récursion est **tail recursive** si l'appel récursif est la **dernière opération** de la fonction — aucun calcul n'est effectué après le retour de l'appel.

```python
# NON tail recursive — multiplication après le retour
def factorial(n):
    if n == 0: return 1
    return n * factorial(n - 1)   # n * ... → doit attendre le retour

# Tail recursive — accumulation dans un paramètre
def factorial(n, acc=1):
    if n == 0: return acc
    return factorial(n - 1, n * acc)   # dernier appel, rien après
```

**Tail Call Optimization (TCO)** : les compilateurs/interpréteurs qui reconnaissent la tail recursion peuvent réutiliser le stack frame existant au lieu d'en créer un nouveau → récursion en O(1) espace.
**Python ne fait PAS de TCO** (décision de Guido van Rossum — préserve les stack traces lisibles). Convertir en itération manuellement si nécessaire.

**Récursion vs Itération**

| | Récursion | Itération |
|---|---|---|
| **Lisibilité** | Haute (pour les problèmes naturellement récursifs) | Variable |
| **Espace** | O(n) stack frames | O(1) si pas de structure auxiliaire |
| **Performance** | Overhead d'appel de fonction | Généralement plus rapide |
| **Quand** | Arbres, graphes, divide & conquer, backtracking | Boucles simples, grands n |

---

### 31.3 Parcours d'Arbres — Systématique

**Arbre Binaire — Structure**
```
        1
       / \
      2   3
     / \
    4   5
```

**Parcours en Profondeur (DFS)**

```python
class Node:
    def __init__(self, val, left=None, right=None):
        self.val = val; self.left = left; self.right = right

# Pre-order : Racine → Gauche → Droite
def preorder(node):          # 1, 2, 4, 5, 3
    if not node: return
    print(node.val)
    preorder(node.left)
    preorder(node.right)

# In-order : Gauche → Racine → Droite
def inorder(node):           # 4, 2, 5, 1, 3  ← donne les valeurs triées pour un BST
    if not node: return
    inorder(node.left)
    print(node.val)
    inorder(node.right)

# Post-order : Gauche → Droite → Racine
def postorder(node):         # 4, 5, 2, 3, 1  ← utile pour supprimer un arbre
    if not node: return
    postorder(node.left)
    postorder(node.right)
    print(node.val)
```

**Parcours en Largeur (BFS) — Level-order**
```python
from collections import deque

def bfs(root):               # 1, 2, 3, 4, 5
    if not root: return
    queue = deque([root])
    while queue:
        node = queue.popleft()
        print(node.val)
        if node.left:  queue.append(node.left)
        if node.right: queue.append(node.right)
```

**Propriété fondamentale** : l'**in-order traversal d'un BST** produit les valeurs dans l'ordre croissant.

**DFS itératif** — utilise une **stack** explicite au lieu de la pile d'appels.
**BFS** — utilise toujours une **queue**.

---

### 31.4 Théorème Maître

**Contexte** : résoudre les récurrences de la forme $T(n) = aT(n/b) + f(n)$ issues des algorithmes Divide & Conquer.

- $a \geq 1$ : nombre de sous-problèmes.
- $b > 1$ : facteur de réduction de la taille.
- $f(n)$ : coût de la division et de la combinaison.

**3 Cas**

Soit $c = \log_b a$ (exposant critique).

| Cas | Condition | Résultat |
|---|---|---|
| **Cas 1** | $f(n) = O(n^{c-\epsilon})$ pour $\epsilon > 0$ — $f$ croît plus lentement | $T(n) = \Theta(n^c)$ |
| **Cas 2** | $f(n) = \Theta(n^c \log^k n)$ — $f$ et la récurrence sont équilibrées | $T(n) = \Theta(n^c \log^{k+1} n)$ |
| **Cas 3** | $f(n) = \Omega(n^{c+\epsilon})$ — $f$ domine | $T(n) = \Theta(f(n))$ |

**Applications canoniques**

| Algorithme | Récurrence | Cas | Résultat |
|---|---|---|---|
| Merge Sort | $T(n) = 2T(n/2) + O(n)$ | Cas 2 ($c=1$, $f=n$) | $O(n \log n)$ |
| Binary Search | $T(n) = T(n/2) + O(1)$ | Cas 2 ($c=0$, $f=1$) | $O(\log n)$ |
| Strassen | $T(n) = 7T(n/2) + O(n^2)$ | Cas 1 ($c=\log_2 7 \approx 2.81$) | $O(n^{2.81})$ |
| Simple recursion | $T(n) = T(n/2) + O(n)$ | Cas 3 ($f=n$ domine) | $O(n)$ |

---

### 31.5 Paradigmes de Programmation

**Procédural**
Le programme est une séquence d'instructions qui modifient l'état. Unité de base : la **procédure** (fonction). Flux de contrôle explicite. Exemples : C, Pascal.

**Orienté Objet (OOP)**
Organisation du code autour d'**objets** qui encapsulent état (attributs) et comportement (méthodes). Principes : encapsulation, héritage, polymorphisme, abstraction. Exemples : Java, C++, Python.

**Fonctionnel**
Le programme est une composition de **fonctions pures** sans état mutable. Unité de base : la fonction mathématique. Pas de side effects, pas de mutation. Exemples : Haskell, Erlang. Python supporte le style fonctionnel.

**Déclaratif**
Décrit **quoi** faire, pas **comment**. Le moteur d'exécution détermine le comment. Exemples : SQL (`SELECT ... FROM ... WHERE`), HTML, Prolog.

**Concepts clés de la programmation fonctionnelle**

- **Fonction pure** : même entrée → même sortie, **aucun side effect** (pas de modification d'état externe, pas d'I/O, pas d'exception).
- **Immutabilité** : les données ne sont jamais modifiées — on crée de nouvelles structures.
- **Side effect** : toute interaction avec le monde extérieur à la fonction (I/O, modification de variable globale, mutation d'un argument).
- **Referential transparency** : une expression peut être remplacée par sa valeur sans changer le comportement du programme. Propriété des fonctions pures.
- **First-class functions** : les fonctions sont des valeurs — passées en argument, retournées, stockées.
- **Higher-order functions** : fonctions qui prennent ou retournent des fonctions (`map`, `filter`, `reduce`).
- **Currying** : transformation d'une fonction à $n$ arguments en une chaîne de fonctions unaires. `f(a, b, c)` → `f(a)(b)(c)`.

```python
# Pur fonctionnel en Python
from functools import reduce
import operator

# Pure function — pas de side effect
def double(x): return x * 2

# Higher-order function
result = list(map(double, [1, 2, 3]))          # [2, 4, 6]
evens  = list(filter(lambda x: x % 2 == 0, range(10)))
total  = reduce(operator.add, [1, 2, 3, 4])   # 10
```

---

### 31.6 Algorithmes de Recherche

**Recherche Linéaire (Linear Search)**
Parcourt séquentiellement — O(n). Fonctionne sur tout tableau, trié ou non.

**Recherche Binaire (Binary Search)**
Nécessite un tableau **trié**. À chaque étape, compare l'élément du milieu et élimine la moitié du tableau restant.

```python
def binary_search(arr, target):
    lo, hi = 0, len(arr) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if arr[mid] == target:   return mid
        elif arr[mid] < target:  lo = mid + 1
        else:                    hi = mid - 1
    return -1
```

Complexité : O(log n). Invariant : `arr[lo] ≤ target ≤ arr[hi]` à chaque itération.

**`bisect` module (Python)** : implémentation standard de la binary search. `bisect_left`, `bisect_right` — insertion position pour maintenir l'ordre.

---

### 31.7 Opérations Bit à Bit & Représentation des Nombres

**Opérateurs bitwise**

| Opérateur | Python | Signification |
|---|---|---|
| AND | `a & b` | 1 si les deux bits sont 1 |
| OR | `a \| b` | 1 si au moins un bit est 1 |
| XOR | `a ^ b` | 1 si les bits sont différents |
| NOT | `~a` | Inversion (complément à 2) |
| Left shift | `a << n` | Multiplie par $2^n$ |
| Right shift | `a >> n` | Divise par $2^n$ (entier) |

**Tricks classiques**
```python
x & (x-1)      # met à 0 le bit le moins significatif — teste si x est puissance de 2 (== 0)
x & (-x)       # isole le bit le moins significatif
x ^ x          # toujours 0 — trouver l'élément unique dans un tableau de doublons
```

**IEEE 754 — Représentation des flottants**
Un `float` Python (64-bit double) : 1 bit signe + 11 bits exposant + 52 bits mantisse.
$$\text{valeur} = (-1)^s \times 1.mantisse \times 2^{exposant - 1023}$$

**Conséquence** : les flottants ne sont pas exacts — `0.1 + 0.2 ≠ 0.3` en arithmétique flottante. Utiliser `math.isclose()` ou `decimal.Decimal` pour les comparaisons précises.

**Valeurs spéciales** : `inf`, `-inf`, `nan` (`not a number` — résultat d'opérations indéfinies).

**Unicode vs ASCII vs UTF-8 vs bytes**

| | ASCII | Unicode | UTF-8 | bytes |
|---|---|---|---|---|
| **Définition** | 128 caractères (7 bits) | Standard universel (1M+ points de code) | Encodage variable de Unicode | Séquence brute d'octets |
| **Taille** | 1 byte/char | Abstrait (pas de taille fixe) | 1-4 bytes/char | 1 byte |
| **Python** | — | `str` (Python 3) | `str.encode('utf-8')` | `bytes` |

En Python 3, `str` est **toujours Unicode**. `bytes` est une séquence d'entiers 0-255. L'encodage convertit `str` → `bytes`, le décodage l'inverse.

---

## PILIER XXXII — ML "Premier Cours" Manquant

---

### 32.1 Régression Linéaire — Fondements Rigoureux

**Modèle**
$$\hat{y} = \theta_0 + \theta_1 x_1 + ... + \theta_n x_n = \theta^T x$$

**OLS — Ordinary Least Squares**
Minimise la somme des carrés des résidus :
$$\mathcal{L}(\theta) = \sum_{i=1}^m (y_i - \theta^T x_i)^2 = \|y - X\theta\|^2$$

**Équation Normale (Normal Equation)**
Solution analytique exacte (pas d'itération) :
$$\hat{\theta} = (X^T X)^{-1} X^T y$$

Dérivation : $\nabla_\theta \mathcal{L} = 0$ → $X^T X \theta = X^T y$.

**Quand l'équation normale est inapplicable**
- $X^T X$ non inversible : features colinéaires (multicolinéarité) → ajouter Ridge ($X^T X + \lambda I$).
- $O(n^3)$ pour l'inversion → trop lent si $n$ (features) est grand. Utiliser la descente de gradient.

**Hypothèses de validité d'OLS (Gauss-Markov)**
1. **Linéarité** : la relation entre $X$ et $y$ est linéaire.
2. **Exogénéité** : $\mathbb{E}[\epsilon | X] = 0$ — les erreurs ne sont pas corrélées aux features.
3. **Homoscédasticité** : variance constante des erreurs $\text{Var}(\epsilon_i) = \sigma^2$.
4. **Pas d'autocorrélation** : $\text{Cov}(\epsilon_i, \epsilon_j) = 0$ pour $i \neq j$.
5. **Pas de multicolinéarité parfaite** : $X$ est de rang plein.

Si ces hypothèses sont vérifiées, OLS est le meilleur estimateur linéaire non biaisé (**BLUE — Best Linear Unbiased Estimator**, théorème de Gauss-Markov).

**Métriques d'évaluation**
$$R^2 = 1 - \frac{SS_{res}}{SS_{tot}} = 1 - \frac{\sum(y_i - \hat{y}_i)^2}{\sum(y_i - \bar{y})^2}$$

$R^2 \in [0,1]$ — proportion de variance expliquée. $R^2 = 1$ : fit parfait. $R^2 = 0$ : le modèle ne fait pas mieux que la moyenne.

**$R^2$ ajusté** : pénalise l'ajout de features non informatives (contrairement à $R^2$ qui ne peut qu'augmenter).

**Ridge vs Lasso (rappel)**
- **Ridge** ($L2$) : $\mathcal{L} + \lambda \|\theta\|_2^2$ → solution analytique : $\hat{\theta} = (X^T X + \lambda I)^{-1} X^T y$.
- **Lasso** ($L1$) : $\mathcal{L} + \lambda \|\theta\|_1$ → pas de solution analytique (non-différentiable en 0) → algorithmes itératifs (coordinate descent).

---

### 32.2 Naive Bayes

**Principe — Indépendance Conditionnelle Naïve**
Applique le théorème de Bayes en supposant que les features sont **conditionnellement indépendantes** étant donné la classe :
$$P(y | x_1, ..., x_n) \propto P(y) \prod_{i=1}^n P(x_i | y)$$

La classe prédite est :
$$\hat{y} = \arg\max_y P(y) \prod_{i=1}^n P(x_i | y)$$

**Pourquoi "naïf"** : l'hypothèse d'indépendance conditionnelle est rarement vraie en pratique, mais Naive Bayes fonctionne étonnamment bien malgré cela (surtout en classification de texte).

**Variantes**

| Variante | Distribution $P(x_i | y)$ | Cas d'usage |
|---|---|---|
| **Gaussian NB** | Gaussienne (mean, variance estimés) | Features continues |
| **Multinomial NB** | Multinomiale (comptages) | Fréquences de mots (bag-of-words) |
| **Bernoulli NB** | Bernoulli (binaire) | Présence/absence de mots |

**Lissage de Laplace (Additive Smoothing)**
Si un mot n'apparaît jamais avec une classe dans le corpus d'entraînement → $P(x_i | y) = 0$ → produit entier = 0 (zero probability problem).
Solution : ajouter $\alpha$ (typiquement 1) à tous les comptages :
$$P(x_i | y) = \frac{\text{count}(x_i, y) + \alpha}{\text{count}(y) + \alpha \cdot |V|}$$

**Underflow numérique** : le produit de nombreuses probabilités → très petit nombre → underflow. Solution : travailler en **log-espace** (log-sum au lieu du produit).

**Avantages** : très rapide à entraîner et inférer, fonctionne bien avec peu de données, pas de problème de convergence.

---

### 32.3 KNN — K-Nearest Neighbors

**Algorithme exact**
1. Calculer la distance entre le point de requête $x$ et **tous les points d'entraînement**.
2. Sélectionner les $K$ points les plus proches.
3. **Classification** : vote majoritaire parmi les $K$ voisins.
4. **Régression** : moyenne (ou moyenne pondérée par l'inverse de la distance) des $K$ voisins.

**Complexité** : O(nd) à l'entraînement (rien — lazy learner), O(nd) à l'inférence (calcul de toutes les distances), $n$ = taille dataset, $d$ = dimensions.

**Choix de K**
- $K$ trop petit : overfitting (sensible au bruit).
- $K$ trop grand : underfitting (frontière trop lisse, classes distantes influencent).
- Heuristique de départ : $K = \sqrt{n}$.
- Sélection par cross-validation.

**KNN est un learner non paramétrique et lazy** : aucun paramètre appris, tout le dataset est gardé en mémoire.

**Impact de la Curse of Dimensionality** : en haute dimension, les distances entre points se concentrent — tous les voisins semblent équidistants. KNN dégénère.

**Structures d'indexation pour accélérer** : KD-Tree (efficace en basse dimension), Ball Tree, HNSW (haute dimension).

---

### 32.4 Perceptron — Fondement des Réseaux de Neurones

**Définition (Rosenblatt, 1958)**
Le perceptron est le neurone artificiel le plus simple : combinaison linéaire des entrées + fonction d'activation seuil.

$$\hat{y} = \begin{cases} 1 & \text{si } w^T x + b \geq 0 \\ 0 & \text{sinon} \end{cases}$$

**Règle de mise à jour (Perceptron Learning Rule)**
Pour chaque exemple mal classé $(x_i, y_i)$ :
$$w \leftarrow w + \eta (y_i - \hat{y}_i) x_i$$
$$b \leftarrow b + \eta (y_i - \hat{y}_i)$$

**Théorème de convergence** : si les données sont **linéairement séparables**, le perceptron converge en un nombre fini d'itérations.

**Limite fondamentale — Problème XOR**
Le XOR n'est pas linéairement séparable — aucun hyperplan ne peut séparer les classes. Minsky & Papert (1969) le prouvent formellement → premier AI Winter.

**Solution** : le **MLP (Multi-Layer Perceptron)** ajoute des couches cachées + fonctions d'activation non-linéaires → peut apprendre des frontières non-linéaires.

```
XOR :   (0,0)→0   (0,1)→1   (1,0)→1   (1,1)→0
Pas de ligne droite qui sépare les 1 des 0
```

---

### 32.5 DBSCAN — Clustering par Densité

**Définition** : DBSCAN (Density-Based Spatial Clustering of Applications with Noise) groupe les points en clusters basés sur la **densité locale**, sans nécessiter de spécifier le nombre de clusters.

**Deux paramètres**
- **ε (eps)** : rayon de voisinage — distance maximale pour être voisin.
- **MinPts** : nombre minimum de points dans le voisinage ε pour qu'un point soit un **core point**.

**Trois types de points**
- **Core point** : a au moins MinPts points dans son ε-voisinage.
- **Border point** : dans le ε-voisinage d'un core point, mais a moins de MinPts voisins propres.
- **Noise point (outlier)** : ni core, ni border — isolé.

**Algorithme**
1. Pour chaque point non visité :
   - S'il a ≥ MinPts voisins dans ε → nouveau cluster, expansion récursive.
   - Sinon → marqué temporairement comme bruit.
2. Un cluster est étendu à tous les points atteignables par densité (core points adjacents).

**Avantages vs k-means**
- Pas de nombre de clusters à spécifier.
- Détecte des clusters de **forme arbitraire**.
- Identifie les **outliers** explicitement.
- Robuste au bruit.

**Limitations**
- Sensible au choix de ε et MinPts.
- Difficile avec des densités variables (clusters de densités très différentes).
- Complexité O(n log n) avec indexation spatiale, O(n²) sans.

---

### 32.6 Clustering Hiérarchique

**Définition** : construit une hiérarchie de clusters représentée sous forme de **dendrogramme** — arbre où les feuilles sont les points individuels et la racine est un seul cluster.

**Approche Agglomérative (bottom-up)**
1. Chaque point commence dans son propre cluster.
2. À chaque étape, fusionner les deux clusters les **plus proches**.
3. Répéter jusqu'à un seul cluster.

**Critères de linkage (mesure de distance entre clusters)**

| Linkage | Distance entre clusters A et B |
|---|---|
| **Single** | $\min_{a \in A, b \in B} d(a, b)$ — plus proches voisins |
| **Complete** | $\max_{a \in A, b \in B} d(a, b)$ — plus éloignés voisins |
| **Average (UPGMA)** | Moyenne de toutes les distances pairées |
| **Ward** | Minimise l'augmentation de la variance intra-cluster |

- **Single linkage** → clusters allongés, sensible aux outliers (chaining effect).
- **Complete linkage** → clusters compacts.
- **Ward** → clusters sphériques de taille similaire, souvent le meilleur en pratique.

**Dendrogramme** : l'axe vertical représente la distance de fusion. Couper le dendrogramme à une hauteur donnée produit un nombre K de clusters.

**Avantage vs k-means** : pas besoin de spécifier K à l'avance, visualisation de la structure hiérarchique. **Inconvénient** : O(n² log n) complexité — ne scale pas aux très grands datasets.

---

### 32.7 t-SNE & UMAP — Réduction Dimensionnelle Non-Linéaire

**PCA (rappel)** : linéaire — ne peut capturer que les structures linéaires. t-SNE et UMAP capturent les structures non-linéaires (**manifold learning**).

**t-SNE (t-Distributed Stochastic Neighbor Embedding — Van der Maaten & Hinton, 2008)**

**Objectif** : projeter des données en 2D/3D en préservant les **relations de voisinage local**.

**Mécanisme**
1. Dans l'espace de haute dimension : calculer les similarités entre paires de points via une distribution **Gaussienne** → $p_{ij}$ (probabilité que $i$ et $j$ soient voisins).
2. Dans l'espace de basse dimension : modéliser les similarités via une distribution **t de Student** (queue plus lourde) → $q_{ij}$.
3. Minimiser la **divergence KL** entre $p_{ij}$ et $q_{ij}$ par descente de gradient.

**Pourquoi la t de Student ?** : les queues lourdes évitent le problème de "crowding" — en haute dimension, les points proches sont trop nombreux pour tenir dans un espace 2D avec une Gaussienne. La t de Student étale les points distants.

**Limitations critiques** :
- **Non déterministe** (résultats variables selon l'initialisation).
- **Lent** : O(n² log n) avec Barnes-Hut approximation.
- **Ne préserve pas les distances globales** — seule la structure locale est fiable.
- **Pas de projection out-of-sample** : un nouveau point ne peut pas être projeté sans réentraîner.
- Les distances dans l'espace t-SNE ne sont **pas interprétables**.

**UMAP (Uniform Manifold Approximation and Projection — McInnes et al., 2018)**

Basé sur la **géométrie riemannienne** et la **topologie algébrique** (théorie des catégories fuzzy simplicielles).

**Avantages vs t-SNE**
- Beaucoup plus **rapide** (O(n log n)).
- **Préserve mieux la structure globale** (en plus de la locale).
- Supporte la **projection out-of-sample** (`.transform()` sur de nouveaux points).
- Paramètre `min_dist` contrôle la compacité des clusters.

**Paramètres clés** : `n_neighbors` (taille du voisinage local — faible → structure locale, élevé → structure globale), `min_dist` (compacité dans l'espace projeté), `metric`.

---

### 32.8 Initialisation des Poids des Réseaux de Neurones

**Pourquoi l'initialisation est critique ?**
- **Tous les poids à 0** : les gradients sont identiques → tous les neurones apprennent la même chose (**symmetry breaking problem**). Le réseau ne peut pas apprendre de représentations diversifiées.
- **Poids trop grands** : saturent les activations (tanh/sigmoid) → vanishing gradient.
- **Poids trop petits** : activations → 0 → gradients → 0.

**Xavier / Glorot Initialization (Glorot & Bengio, 2010)**
Pour les activations **symétriques** (tanh, sigmoid) :
$$W \sim \mathcal{U}\left[-\frac{\sqrt{6}}{\sqrt{n_{in} + n_{out}}}, \frac{\sqrt{6}}{\sqrt{n_{in} + n_{out}}}\right]$$

Maintient la variance des activations constante à travers les couches dans les deux sens (forward et backward).

**He Initialization (He et al., 2015)**
Pour les activations **ReLU** :
$$W \sim \mathcal{N}\left(0, \frac{2}{n_{in}}\right)$$

ReLU annule la moitié des neurones → variance divisée par 2 → compensation par le facteur 2 dans He init.

**Résumé**
- Tanh/Sigmoid → Xavier.
- ReLU/LeakyReLU → He.
- SELU → LeCun init.

---

## PILIER XXXIII — Python : Outils Manquants

---

### 33.1 Expressions Régulières — Module `re`

**Définition**
Une **expression régulière** (regex) est un motif formel qui décrit un ensemble de chaînes de caractères. Basée sur la théorie des **automates finis** (NFA/DFA).

**Syntaxe fondamentale**

| Pattern | Signification |
|---|---|
| `.` | N'importe quel caractère (sauf `\n`) |
| `^` | Début de chaîne (ou de ligne avec `re.MULTILINE`) |
| `$` | Fin de chaîne |
| `*` | 0 ou plus (greedy) |
| `+` | 1 ou plus (greedy) |
| `?` | 0 ou 1 (greedy) — ou rend `*`,`+` lazy : `*?`, `+?` |
| `{n,m}` | Entre n et m occurrences |
| `[abc]` | Classe de caractères — a, b ou c |
| `[^abc]` | Classe négative — tout sauf a, b, c |
| `\d` | Chiffre `[0-9]` |
| `\w` | Alphanumérique + underscore `[a-zA-Z0-9_]` |
| `\s` | Espace blanc `[ \t\n\r\f\v]` |
| `\b` | Frontière de mot |
| `(abc)` | Groupe de capture |
| `(?:abc)` | Groupe non-capturant |
| `a\|b` | Alternation — a ou b |

**Lookahead / Lookbehind (assertions de position)**
```python
# Lookahead positif — "suivi de"
r'\d+(?= euros)'    # digits suivis de " euros" (sans capturer " euros")

# Lookahead négatif — "non suivi de"
r'\d+(?! euros)'

# Lookbehind positif — "précédé de"
r'(?<=\$)\d+'       # digits précédés de $

# Lookbehind négatif — "non précédé de"
r'(?<!\$)\d+'
```

**Fonctions `re` essentielles**
```python
import re

re.match(pattern, string)      # match uniquement en début de chaîne
re.search(pattern, string)     # premier match n'importe où
re.findall(pattern, string)    # liste de tous les matches
re.finditer(pattern, string)   # itérateur de Match objects
re.sub(pattern, repl, string)  # substitution
re.split(pattern, string)      # split

# Groupes de capture
m = re.search(r'(\d{4})-(\d{2})-(\d{2})', '2026-04-10')
m.group(0)   # '2026-04-10' — match complet
m.group(1)   # '2026'
m.groups()   # ('2026', '04', '10')

# Named groups
m = re.search(r'(?P<year>\d{4})-(?P<month>\d{2})', '2026-04')
m.group('year')   # '2026'

# Precompiler pour réutilisation
pattern = re.compile(r'\d+', re.IGNORECASE)
```

**Greedy vs Lazy**
```python
re.findall(r'<.*>',  '<a>text</a>')   # ['<a>text</a>'] — greedy (max)
re.findall(r'<.*?>', '<a>text</a>')   # ['<a>', '</a>'] — lazy (min)
```

**Flags utiles** : `re.IGNORECASE`, `re.MULTILINE`, `re.DOTALL` (`.` matche `\n`), `re.VERBOSE` (regex multi-lignes commentées).

---

### 33.2 Environnements Virtuels & Gestion de Packages

**Pourquoi les environnements virtuels ?**
Isolation des dépendances par projet — évite les conflits de versions entre projets différents et la pollution du système Python global.

**`venv` (stdlib)**
```bash
python -m venv .venv           # crée l'environnement
source .venv/bin/activate      # active (Unix)
.venv\Scripts\activate         # active (Windows)
pip install requests           # installe dans l'env isolé
deactivate                     # désactive
```

**`pip` — Package Installer for Python**
```bash
pip install package==1.2.3     # version exacte
pip install "package>=1.0,<2"  # contrainte de version
pip freeze > requirements.txt  # snapshot des versions installées
pip install -r requirements.txt
```

**`poetry` — Gestion moderne**
- Remplace `setup.py`, `requirements.txt`, `pip` par un seul outil.
- **`pyproject.toml`** (PEP 518) : standard moderne pour la configuration des packages.
- Résout les dépendances via un **SAT solver** → `poetry.lock` (snapshot exact).
- Distingue les dépendances de production et de développement.

**`conda`** : gestionnaire d'environnements et de packages multi-langage (Python + C, R, etc.). Inclut des binaires pré-compilés (numpy, scipy avec MKL) → recommandé pour le ML.

**`pyproject.toml` (PEP 517/518)** : standard remplaçant `setup.py`. Déclare le build system, les dépendances, les métadonnées du package.

---

### 33.3 Python Bytecode & AST

**Bytecode Python**
Après parsing, le code Python est compilé en **bytecode** — séquence d'instructions pour la machine virtuelle CPython. Stocké dans les fichiers `.pyc` (dans `__pycache__/`).

```python
import dis

def add(a, b):
    return a + b

dis.dis(add)
# LOAD_FAST 0 (a)
# LOAD_FAST 1 (b)
# BINARY_OP 0 (ADD)
# RETURN_VALUE
```

**`compile()` → `exec()` / `eval()`** : compilation et exécution dynamique de code Python.

**AST — Abstract Syntax Tree**
Représentation structurée du code source sous forme d'arbre. Chaque nœud correspond à une construction syntaxique.

```python
import ast

tree = ast.parse("x = 1 + 2")
ast.dump(tree)
# Module(body=[Assign(targets=[Name(id='x')],
#              value=BinOp(left=Num(1), op=Add(), right=Num(2)))])
```

**Cas d'usage de l'AST** : linters (pylint, flake8), formateurs (black), type checkers (mypy, pyright), outils de refactoring, sécurité (détection de `eval()`/`exec()` dangereux).

---

### 33.4 Outils de Qualité Code

| Outil | Rôle | Catégorie |
|---|---|---|
| **black** | Formatage automatique — opinionated, pas configurable | Formatter |
| **ruff** | Linter ultra-rapide (écrit en Rust), remplace flake8+isort+pyupgrade | Linter |
| **mypy** | Vérification statique des type hints | Type checker |
| **pyright** | Type checker (Microsoft, base de Pylance) | Type checker |
| **pylint** | Linter complet — style, erreurs, complexité | Linter |
| **isort** | Tri automatique des imports | Formatter |
| **bandit** | Analyse de sécurité statique | Security |
| **pre-commit** | Hooks Git qui exécutent les outils avant chaque commit | CI |

**Différence Linter vs Formatter vs Type Checker**
- **Formatter** : reformate le code automatiquement (espaces, indentation, guillemets).
- **Linter** : détecte les erreurs de style, les anti-patterns, les bugs potentiels — suggère des corrections.
- **Type Checker** : vérifie la cohérence des type hints sans exécuter le code.

---

## PILIER XXXIV — Autres Sujets "École" Possibles

---

### 34.1 Théorie des Graphes — Algorithmes Fondamentaux

**Dijkstra — Plus Court Chemin (poids positifs)**
```
1. Initialiser dist[source] = 0, dist[autres] = ∞
2. Mettre source dans la priority queue (min-heap)
3. Tant que la queue n'est pas vide :
   a. Extraire le nœud u avec la plus petite dist
   b. Pour chaque voisin v de u :
      si dist[u] + w(u,v) < dist[v] :
          dist[v] = dist[u] + w(u,v)
          Ajouter v à la queue
```
Complexité : O((V+E) log V) avec min-heap.
**Limitation** : ne fonctionne pas avec des poids négatifs.

**Bellman-Ford — Plus Court Chemin (poids négatifs autorisés)**
Relaxe toutes les arêtes V-1 fois. Complexité O(VE). Détecte les cycles négatifs.

**Tri Topologique** : ordre linéaire des nœuds d'un DAG tel que pour toute arête (u,v), u précède v. Algorithme : DFS avec ajout en tête de liste à la fin de chaque nœud (post-order). Utilisé pour la résolution des dépendances.

**Minimum Spanning Tree (MST)**
- **Kruskal** : trie les arêtes par poids, ajoute greedy en évitant les cycles (Union-Find). O(E log E).
- **Prim** : commence par un nœud, ajoute greedy l'arête minimale vers un nouveau nœud. O(E log V) avec heap.

**Floyd-Warshall** : tous les plus courts chemins entre toutes les paires. O(V³). Accepte les poids négatifs (mais pas les cycles négatifs).

---

### 34.2 Logique & Algèbre Booléenne

**Portes logiques fondamentales** : AND, OR, NOT, NAND, NOR, XOR, XNOR.

**Lois de De Morgan**
$$\overline{A \cdot B} = \overline{A} + \overline{B}$$
$$\overline{A + B} = \overline{A} \cdot \overline{B}$$

**Table de vérité XOR** : $A \oplus B = 1$ si et seulement si $A \neq B$. XOR est son propre inverse : $A \oplus B \oplus B = A$.

**Propriétés utiles en programmation**
```python
x ^ x == 0         # XOR avec soi-même = 0
x ^ 0 == x         # XOR avec 0 = identité
x ^ y ^ y == x     # XOR est son propre inverse
```
Application : trouver l'élément unique dans un tableau où tous les autres sont en double → XOR de tous les éléments.

---

### 34.3 Complexité & Problèmes NP

**Classes de complexité**

- **P** : problèmes décidables en temps polynomial. Exemple : tri, recherche, plus court chemin.
- **NP** : problèmes dont la **solution peut être vérifiée** en temps polynomial. Exemple : SAT, Travelling Salesman, Knapsack.
- **NP-complet** : dans NP et tout problème NP peut s'y réduire. Exemple : SAT (Cook-Levin), 3-SAT, Clique, Vertex Cover.
- **NP-dur** : au moins aussi difficile que NP-complet, mais pas nécessairement dans NP.

**P = NP ?** : question ouverte fondamentale en informatique théorique — le plus grand problème non résolu en CS.

**Réductibilité polynomiale** : si A se réduit à B en temps polynomial et B ∈ P, alors A ∈ P.

---

### 34.4 Compression — Principes

**Compression sans perte (lossless)**
- **Huffman Coding** : codes de longueur variable basés sur les fréquences. Les symboles fréquents ont des codes courts. Construit un arbre binaire de Huffman (arbre optimal par l'algorithme glouton). Entropie optimale.
- **LZ77/LZ78/LZW** : dictionnaire de séquences répétées (base de ZIP, GZIP, PNG).
- **Run-Length Encoding (RLE)** : remplace les séquences répétées par (count, value).

**Compression avec perte (lossy)**
- **JPEG** : DCT (Discrete Cosine Transform) → quantification des hautes fréquences (imperceptibles à l'œil).
- **MP3** : suppression des fréquences inaudibles (masquage auditif psychoacoustique).

**Lien avec l'entropie** : l'entropie de Shannon fixe la borne inférieure de compression — impossible de compresser en dessous de l'entropie sans perte d'information.

---

### 34.5 Bases de la Cryptographie Asymétrique

**RSA — Rivest-Shamir-Adleman**

**Génération de clés**
1. Choisir deux grands nombres premiers $p$ et $q$.
2. $n = p \cdot q$ (module).
3. $\phi(n) = (p-1)(q-1)$ (indicateur d'Euler).
4. Choisir $e$ tel que $1 < e < \phi(n)$ et $\gcd(e, \phi(n)) = 1$.
5. Calculer $d = e^{-1} \mod \phi(n)$ (inverse modulaire).
6. **Clé publique** : $(n, e)$. **Clé privée** : $(n, d)$.

**Chiffrement/Déchiffrement**
$$C = M^e \mod n \quad \text{(chiffrement avec clé publique)}$$
$$M = C^d \mod n \quad \text{(déchiffrement avec clé privée)}$$

**Sécurité** : basée sur la difficulté de factoriser $n$ en $p$ et $q$ (**factorisation d'entiers**).

**Diffie-Hellman Key Exchange**
Permet à deux parties d'établir un **secret partagé** sur un canal public, sans s'échanger directement la clé secrète. Basé sur le **problème du logarithme discret**.

---

### 34.6 API Design — Versioning, Pagination & Idempotency

**Versioning d'API**

| Stratégie | Exemple | Avantages | Limitations |
|---|---|---|---|
| **URL path** | `/api/v1/users` | Explicit, cacheable | Prolifération d'URLs |
| **Header** | `Accept: application/vnd.api+json;version=1` | URLs propres | Moins visible |
| **Query param** | `/api/users?version=1` | Simple | Non standard |

**Pagination**

- **Offset/Limit** : `?offset=20&limit=10`. Simple mais problème des pages qui sautent si des données sont insérées entre deux pages.
- **Cursor-based** : `?cursor=<opaque_token>`. Stable — le cursor pointe vers une position spécifique dans l'ensemble ordonné. Recommandé pour les grands datasets.
- **Keyset pagination** : `?after_id=42`. Basé sur la dernière clé vue. Performant (index-friendly), mais nécessite un tri stable.

**Idempotency Keys**
Pour les opérations non-idempotentes (POST, paiement), envoyer un `Idempotency-Key` header unique par requête. Si la même clé est reçue → retourner le résultat de la première exécution sans réexécuter. Protège contre les retries en cas d'erreur réseau.

**N+1 Query Problem**
Pattern anti-performant dans les ORM : charger N objets parents, puis faire 1 requête par parent pour ses enfants → N+1 requêtes au lieu de 1.
```python
# N+1 — mauvais
users = User.query.all()            # 1 requête
for user in users:
    print(user.posts.all())         # N requêtes

# Eager loading — bon
users = User.query.options(joinedload(User.posts)).all()  # 1 requête avec JOIN
```

---

### 34.7 Protocoles d'Authentification Complémentaires

**Session-based vs Token-based**

| | Session | JWT |
|---|---|---|
| **État serveur** | Stateful (session en BDD/Redis) | Stateless (token auto-porteur) |
| **Scalabilité** | Sticky sessions ou session store partagé | Naturellement scalable |
| **Révocation** | Immédiate (supprimer la session) | Difficile (jusqu'à expiration du JWT) |
| **Taille** | ID court (ex: UUID) | Token plus grand (~200-500 bytes) |

**Refresh Token Pattern**
- **Access token** : courte durée (15 min), utilisé pour les requêtes API.
- **Refresh token** : longue durée (jours/semaines), stocké HttpOnly cookie, utilisé uniquement pour obtenir un nouveau access token.
- Si le refresh token est compromis → invalider en BDD (liste noire) ou rotation forcée.

**PKCE — Proof Key for Code Exchange**
Extension de OAuth2 Authorization Code pour les clients publics (mobile, SPA). Le client génère un `code_verifier` aléatoire et envoie son hash (`code_challenge`) au serveur d'auth. À l'échange du code → vérifie que le `code_verifier` correspond. Protège contre l'interception du authorization code.

---

### 34.8 Principes de Conception de Données

**Normalisation vs Dénormalisation** (rappel étendu)
- **Normaliser** : pour les systèmes OLTP, minimiser la redondance, faciliter les écritures cohérentes.
- **Dénormaliser** : pour les systèmes analytiques ou les APIs, pré-calculer les jointures coûteuses, améliorer les performances de lecture.

**Connection Pooling**
Maintenir un **pool de connexions BDD réutilisables** au lieu d'en créer/détruire une par requête.
- Créer une connexion BDD : 10-50ms (TCP handshake + auth).
- Réutiliser une connexion du pool : < 1ms.
- Libraries : `psycopg2` pool, `SQLAlchemy` pool, `asyncpg` pool.
- Paramètres clés : `pool_size`, `max_overflow`, `pool_timeout`.

**ORM vs Raw SQL**

| | ORM | Raw SQL |
|---|---|---|
| **Productivité** | Haute (abstraction du SQL) | Plus verbeux |
| **Performance** | Peut générer des requêtes sous-optimales | Contrôle total |
| **Portabilité** | Abstrait le dialecte SQL | Lié au SGBD |
| **Débogage** | Requêtes générées parfois complexes | Direct |
| **Cas d'usage** | CRUD standard, prototypage | Requêtes complexes, optimisation |

**Indexing — Quand créer un index ?**
- Colonnes fréquemment utilisées dans `WHERE`, `JOIN ON`, `ORDER BY`.
- Colonnes avec haute cardinalité (UUID, email) — meilleur sélectivité.
- **Ne pas indexer** : colonnes rarement cherchées, faible cardinalité (booléen, genre), tables très petites (full scan plus rapide).
- Coût d'un index : overhead à chaque INSERT/UPDATE/DELETE (l'index doit être maintenu).

---

*— Fin de la Fiche de Révision — Version 6.0 Finale & Exhaustive — Athoria 2026 —*
