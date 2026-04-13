# Resume — Cheat Sheet Oral : Formulations Prêtes

*Définitions courtes, précises, prêtes à l'oral.*

---

# SECTION I — Python Internals

---

## 1. GIL — Global Interpreter Lock

**Définition en une phrase** : le GIL est un **mutex CPython** qui garantit qu'un seul thread exécute du bytecode Python à un instant donné, quel que soit le nombre de cœurs disponibles.

**Pourquoi il existe** : CPython gère la mémoire via le comptage de références (`ob_refcnt`). Sans verrou, deux threads pourraient modifier ce compteur simultanément → race condition → corruption mémoire ou libération prématurée d'objet. Le GIL est la solution pragmatique : plutôt que de verrouiller chaque objet individuellement (trop coûteux), on verrouille l'interpréteur entier.

**Conséquence directe** : sur une machine à 4 cœurs avec 4 threads Python, un seul cœur est actif à la fois pour l'exécution du bytecode. Le GIL **annule le bénéfice du multi-cœur pour les tâches CPU-bound**.

**Mécanisme de commutation** : CPython libère le GIL toutes les **5 ms** (`sys.getswitchinterval()`), permettant à un autre thread de l'acquérir. Il est aussi libéré automatiquement pendant les **opérations I/O bloquantes** (appel réseau, lecture fichier) car la C-extension sous-jacente n'a pas besoin du bytecode Python.

---

## 2. Gestion Mémoire & Garbage Collector

**Comptage de références** : chaque objet Python possède un champ `ob_refcnt`. Il est incrémenté à chaque nouvelle référence, décrémenté à chaque suppression. Quand `ob_refcnt == 0` → l'objet est **immédiatement libéré**.

**Limite** : si A référence B et B référence A, leurs compteurs ne tombent jamais à 0 même si aucune variable extérieure ne les pointe. Le comptage de références est **aveugle aux cycles**.

**GC Générationnel (solution)** : basé sur l'hypothèse empiriquement prouvée que *"la majorité des objets meurent jeunes"*. Les objets sont classés en 3 générations :

| Génération | Contenu | Déclenchement |
|---|---|---|
| **Gen 0** | Nouveaux objets | Quand (allocations − désallocations) > 700 |
| **Gen 1** | Survivants de Gen 0 | Après 10 collectes de Gen 0 |
| **Gen 2** | Survivants de Gen 1 | Après 10 collectes de Gen 1 |

Le GC détecte les cycles en simulant la suppression des références internes au groupe : tout objet dont le compteur simulé tombe à 0 est inatteignable → collecté.

---

## 3. Threading vs Multiprocessing vs Asyncio

**Threading**
- Reste dans le **même processus** → partage du même espace mémoire → léger à lancer, communication facile.
- Le GIL limite l'exécution bytecode à un thread à la fois.
- **Idéal pour I/O-bound** : pendant qu'un thread attend (réseau, disque), le GIL est relâché → un autre thread peut s'exécuter.

**Multiprocessing**
- Lance des **processus indépendants**, chacun avec son propre interpréteur et son propre GIL.
- Peut exploiter **tous les cœurs** simultanément.
- **Idéal pour CPU-bound** : calcul matriciel, traitement d'image, encodage vidéo.
- Coût : overhead de fork, sérialisation des données entre processus (pickle).

**Asyncio**
- Un **seul thread**, modèle **coopératif** : c'est le développeur qui cède le contrôle via `await`.
- L'**event loop** surveille les I/O et reprend les coroutines dès qu'elles sont prêtes.
- **Idéal pour I/O-bound massivement concurrent** : gérer des milliers de connexions réseau simultanées sans créer des milliers de threads.
- Contrairement au threading : pas de préemption OS, pas de context switch, pas de race conditions involontaires.

**Résumé de choix**

| Scénario | Solution |
|---|---|
| Attendre des réponses réseau (API, BDD) | `asyncio` ou `threading` |
| Calcul intensif sur CPU | `multiprocessing` |
| Milliers de connexions simultanées | `asyncio` |
| Bibliothèques bloquantes non-async | `threading` |

---

## 4. Décorateurs

**Définition** : un décorateur est un **callable** qui prend une fonction en entrée et retourne une nouvelle fonction augmentée, sans modifier le code source original. Syntaxe `@decorator` = sucre syntaxique pour `func = decorator(func)`.

**3 piliers nécessaires à citer à l'oral**

1. **First-class functions** : en Python, les fonctions sont des objets — on peut les passer en argument, les stocker dans des variables, les retourner depuis d'autres fonctions.
2. **Closures** : la fonction interne `wrapper` **capture** la fonction originale dans son environnement lexical (via `__closure__`), même après la fin de l'exécution du décorateur.
3. **Variable Scope (LEGB)** : le `wrapper` a accès aux variables du scope englobant (le décorateur) via la règle Enclosing.

```python
import functools

def decorator(func):
    @functools.wraps(func)   # préserve __name__, __doc__ — toujours mettre
    def wrapper(*args, **kwargs):
        # avant
        result = func(*args, **kwargs)
        # après
        return result
    return wrapper
```

---

## 5. Générateurs

**Définition** : un générateur est un **itérateur paresseux (lazy)** — il produit les valeurs à la demande plutôt que de les stocker toutes en mémoire.

**Mécanisme** : `yield` suspend l'exécution de la fonction, retourne une valeur, et **préserve l'état complet** (variables locales + pointeur d'instruction dans le `PyFrameObject`) jusqu'au prochain `__next__()`.

**Trois propriétés clés à citer**

- **Performance mémoire** : O(n) → O(1) — indépendant de la taille du flux. Indispensable pour le Big Data, la lecture de fichiers volumineux.
- **État préservé** : la fonction ne "meurt" pas entre deux `yield` — elle est suspendue, pas terminée.
- **Usage unique** : un générateur s'épuise une fois parcouru. Pour le réutiliser → recréer l'objet générateur.

```python
def infinite_stream():
    n = 0
    while True:
        yield n          # suspend ici, reprend au prochain next()
        n += 1

gen = infinite_stream()
next(gen)   # 0
next(gen)   # 1  — état préservé
```

---

## 6. Module / Package / Bibliothèque / venv

**Module** : tout fichier `.py`. Importé avec `import module_name`. À la première importation, le code de niveau supérieur s'exécute une fois, puis le module est mis en cache dans `sys.modules`.

**Package** : répertoire contenant un `__init__.py`. Permet d'organiser des modules en hiérarchie. Le `__init__.py` s'exécute à l'import du package et définit son interface publique.

**Bibliothèque** : terme informel désignant une collection de packages offrant une fonctionnalité complète (ex : NumPy pour le calcul scientifique). La structure réelle est celle d'un package.

**venv (environnement virtuel)** : dossier isolé contenant sa propre installation Python et ses propres bibliothèques — une "bulle hermétique" par projet.

*Pourquoi c'est indispensable* : sans venv, toutes les bibliothèques sont installées globalement. Si Projet A nécessite Django 3.2 et Projet B Django 4.0 → conflit insoluble. Avec un venv par projet → chaque projet a ses versions exactes, sans interférence.

```bash
python -m venv .venv
source .venv/bin/activate
pip install numpy==1.26.0
```

---

## 7. `@classmethod` vs `@staticmethod`

**`@classmethod`**
- Reçoit automatiquement la **classe** (`cls`) comme premier argument.
- Accède et modifie l'état de la classe (attributs de classe).
- **Cas d'usage principal : constructeurs alternatifs** (factory methods).
- `cls` = la classe sur laquelle la méthode est appelée → polymorphisme préservé dans les sous-classes.

**`@staticmethod`**
- Ne reçoit **aucun argument automatique** (ni `self`, ni `cls`).
- Ne peut pas accéder à l'état de l'instance ou de la classe.
- **Cas d'usage : fonctions utilitaires** logiquement liées à la classe mais indépendantes de son état.

**Résumé en une phrase** : `@classmethod` est liée à la **hiérarchie de classe** et agit sur elle ; `@staticmethod` est une **fonction ordinaire** rattachée à la classe uniquement pour l'organisation.

```python
class Date:
    def __init__(self, y, m, d): ...

    @classmethod
    def from_string(cls, s):        # constructeur alternatif
        y, m, d = s.split('-')
        return cls(int(y), int(m), int(d))

    @staticmethod
    def is_valid_year(year):        # utilitaire — pas besoin de cls ou self
        return 1900 <= year <= 2100
```

---

## 8. `@property`

**Définition** : transforme une méthode en **attribut calculé** — l'accès se fait sans parenthèses, comme un attribut, mais déclenche en coulisse l'exécution de la méthode.

**Pourquoi l'utiliser** : encapsuler de la **logique de validation ou de calcul** tout en conservant une interface d'accès propre et intuitive (`obj.radius` au lieu de `obj.get_radius()`). Si l'implémentation interne change, le code client n'est pas impacté.

```python
class Circle:
    def __init__(self, radius):
        self._radius = radius

    @property
    def radius(self):               # getter — obj.radius
        return self._radius

    @radius.setter
    def radius(self, value):        # setter — obj.radius = x
        if value < 0:
            raise ValueError("Radius must be non-negative")
        self._radius = value

    @property
    def area(self):                 # attribut calculé — pas de setter
        return 3.14159 * self._radius ** 2
```

---

## 9. `copy` vs `deepcopy`

**Shallow copy (`copy.copy()`)** : crée un **nouvel objet conteneur** mais les objets internes sont les **mêmes références** que l'original. Modifier un objet imbriqué dans la copie modifie aussi l'original.

**Deep copy (`copy.deepcopy()`)** : duplique **récursivement** l'objet et tout son contenu. La copie et l'original sont **totalement indépendants** en mémoire.

```python
import copy
original = [[1, 2], [3, 4]]

shallow = copy.copy(original)
shallow[0].append(99)    # original[0] aussi modifié → [1, 2, 99]
shallow.append([5])      # original inchangé

deep = copy.deepcopy(original)
deep[0].append(99)       # original inchangé — totalement isolé
```

**Règle de choix** : utiliser `deepcopy` quand les objets imbriqués sont **mutables** et que l'isolation complète est nécessaire. `copy` suffit si les objets imbriqués sont immuables ou si le partage est intentionnel.

---

# SECTION II — POO & Génie Logiciel

---

## 10. POO — Les 4 Piliers

**Définition de la POO** : paradigme qui organise le code autour d'**objets** regroupant état (attributs) et comportement (méthodes), par opposition à la programmation procédurale centrée sur les fonctions.

---

### Encapsulation

**Définition** : regrouper les données et les méthodes au sein d'une classe, en **restreignant l'accès direct** aux attributs depuis l'extérieur.

**En Python** : convention `_attribut` (protégé) et `__attribut` (name mangling). Accès contrôlé via `@property` / `@setter`.

**Pourquoi** : garantit l'**intégrité des données** — empêche qu'un code externe mette l'objet dans un état invalide (ex : solde bancaire négatif).

---

### Abstraction

**Définition** : masquer la complexité interne pour n'exposer que les **fonctionnalités essentielles**. Définir **quoi** faire sans dire **comment**.

**En Python** : module `abc` — `ABC` + `@abstractmethod`. Une classe qui hérite sans implémenter la méthode abstraite lève `TypeError` à l'instanciation.

**Pourquoi** : réduit la charge mentale — on manipule une interface simple sans connaître la mécanique interne. Principe de **séparation interface/implémentation**.

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...   # contrat — pas d'implémentation

class Circle(Shape):
    def __init__(self, r): self.r = r
    def area(self): return 3.14159 * self.r ** 2
```

---

### Héritage

**Définition** : mécanisme par lequel une classe **enfant** acquiert tous les attributs et méthodes d'une classe **parente**.

**En Python** : `class Enfant(Parent)`. Héritage multiple possible → résolution via **MRO (C3 Linearization)**.

**Pourquoi** : favorise la **réutilisation du code** et respecte le principe DRY — la logique commune est centralisée dans la classe générique.

**Point oral — Héritage vs Composition** : préférer la composition quand la relation n'est pas "est-un" mais "a-un" (évite le couplage fort de l'héritage profond).

---

### Polymorphisme

**Définition** : capacité à traiter des objets de **classes différentes** via une **interface commune** — une même méthode produit des comportements différents selon l'objet qui l'appelle.

**Deux formes à distinguer** :

| Type | Mécanisme | Résolution |
|---|---|---|
| **Statique** (surcharge) | Même nom de méthode, signatures différentes | Compile-time (absent nativement en Python) |
| **Dynamique** (override) | Méthode fille redéfinit la méthode héritée | Runtime — via MRO |

**En Python** : le polymorphisme dynamique se manifeste par le **Duck Typing** ("si ça marche comme un canard, c'est un canard") — pas besoin d'héritage formel, il suffit que l'objet implémente la méthode attendue.

**Pourquoi** : rend le code **flexible et extensible** — on peut ajouter de nouveaux types sans modifier les fonctions qui les manipulent. C'est l'essence du principe **Open/Closed** (SOLID).

```python
class Dog:
    def speak(self): return "Woof"

class Cat:
    def speak(self): return "Meow"

def make_sound(animal):      # Duck Typing — pas d'isinstance()
    print(animal.speak())    # polymorphisme dynamique : résolu à runtime

make_sound(Dog())   # Woof
make_sound(Cat())   # Meow
```

---

## 11. Principes SOLID

**Définition** : 5 principes de conception orientée objet formulés par Robert C. Martin, dont le respect produit un code **lisible, extensible et testable**.

---

### S — Single Responsibility Principle (Responsabilité Unique)

**La règle** : une classe ne doit avoir qu'**une seule raison de changer** — une seule responsabilité.

**Le problème** : une classe `Rapport` qui génère les données ET les formate ET les envoie par email. Si le format change, tu touches la même classe que si l'adresse email change → couplage de responsabilités sans rapport.

**La solution** : trois classes distinctes — `GenerateurRapport`, `FormatteurRapport`, `EnvoyeurRapport`. Chacune évolue indépendamment.

---

### O — Open/Closed Principle (Ouvert/Fermé)

**La règle** : une classe doit être **ouverte à l'extension, fermée à la modification**. On ajoute du comportement sans toucher au code existant.

**Le problème** : une fonction `calculer_remise(type_client)` avec un `if/elif` par type. Ajouter un nouveau type de client = modifier la fonction → risque de régression.

**La solution** : une interface `Client` avec une méthode `get_remise()`. Chaque type de client est une classe qui l'implémente. Ajouter un type = créer une nouvelle classe, sans toucher à l'existant.

---

### L — Liskov Substitution Principle (Substitution de Liskov)

**La règle** : tout objet d'une classe **fille** doit pouvoir remplacer un objet de la classe **parente** sans altérer le comportement du programme.

**Le problème** : `Carré` hérite de `Rectangle`. `Rectangle` a `set_largeur()` et `set_hauteur()` indépendants. Mais sur un `Carré`, les deux doivent toujours être égaux → le carré **viole le contrat** de Rectangle. Utiliser un `Carré` à la place d'un `Rectangle` casse le programme.

**La solution** : ne pas forcer cet héritage. `Carré` et `Rectangle` héritent d'une classe abstraite `Forme` commune. **"Est-un" doit être vrai comportementalement, pas seulement conceptuellement.**

---

### I — Interface Segregation Principle (Ségrégation des Interfaces)

**La règle** : une classe ne doit pas être forcée d'implémenter des méthodes qu'elle **n'utilise pas**. Préférer plusieurs interfaces spécifiques à une seule interface générale (fat interface).

**Le problème** : une interface `Machine` avec `imprimer()`, `scanner()`, `faxer()`. Une `Imprimante` simple doit implémenter `scanner()` et `faxer()` qu'elle ne peut pas faire → elle lève `NotImplementedError` → violation du LSP en chaîne.

**La solution** : trois interfaces séparées — `Imprimable`, `Scannable`, `Faxable`. Chaque machine implémente uniquement ce dont elle a besoin.

---

### D — Dependency Inversion Principle (Inversion des Dépendances)

**La règle** : il faut dépendre des **abstractions (interfaces)**, pas des **implémentations concrètes**. Les modules de haut niveau ne doivent pas dépendre des modules de bas niveau — les deux doivent dépendre d'abstractions.

**Le problème** : ta classe `Application` crée directement un objet `ClavierFrancais`. Si demain tu veux un `ClavierAnglais`, tu dois modifier toute l'`Application` → couplage fort, difficulté à tester.

**La solution** : `Application` dépend d'une interface `Clavier`. Tu **injectes** le type de clavier au moment voulu (Dependency Injection). L'`Application` ne sait pas quelle implémentation elle reçoit — elle sait seulement qu'elle a un `Clavier`.

```python
from abc import ABC, abstractmethod

class Clavier(ABC):
    @abstractmethod
    def saisir(self) -> str: ...

class ClavierFrancais(Clavier):
    def saisir(self): return "azerty"

class ClavierAnglais(Clavier):
    def saisir(self): return "qwerty"

class Application:
    def __init__(self, clavier: Clavier):  # injection de dépendance
        self.clavier = clavier             # dépend de l'abstraction

app = Application(ClavierAnglais())        # on change sans toucher à Application
```

**Tableau récapitulatif SOLID**

| Lettre | Principe | Mot-clé |
|---|---|---|
| S | Single Responsibility | Une seule raison de changer |
| O | Open/Closed | Étendre sans modifier |
| L | Liskov Substitution | Sous-classe substituable sans casser |
| I | Interface Segregation | Interfaces fines et spécifiques |
| D | Dependency Inversion | Dépendre des abstractions, pas du concret |

---

## 12. Clean Architecture

**Principe central** : les règles métier (Domain layer) sont **totalement indépendantes** des frameworks, bases de données, interfaces utilisateur et services externes. **La dépendance ne va que vers l'intérieur** — jamais une couche interne ne connaît une couche externe.

**Les 3 couches**

| Couche | Contenu | Dépendances |
|---|---|---|
| **Domain** | Entités métier, règles invariantes, value objects | Aucune — zéro import externe |
| **Application** | Use cases (orchestration des entités), Ports (interfaces abstraites vers l'infrastructure) | Domain uniquement |
| **Infrastructure** | Adapters (implémentations concrètes des ports) : DB repositories, HTTP clients, message brokers | Application + Domain |

**Ports & Adapters — le mécanisme clé**

- Un **Port** est une interface abstraite définie dans la couche Application : elle exprime ce dont le use case a *besoin* (ex : `UserRepository` avec `save()` et `find_by_id()`).
- Un **Adapter** est l'implémentation concrète dans l'Infrastructure : `PostgresUserRepository` implémente `UserRepository`. Le Domain ne sait pas que Postgres existe.

```python
# Domain — aucune dépendance
class User:
    def __init__(self, id: str, email: str): ...
    def change_email(self, new_email: str): ...   # règle métier pure

# Application — Port (interface abstraite)
from abc import ABC, abstractmethod

class UserRepository(ABC):          # Port : contrat, pas d'implémentation
    @abstractmethod
    def save(self, user: User): ...
    @abstractmethod
    def find_by_id(self, id: str) -> User: ...

class ChangeEmailUseCase:           # Use Case : orchestre sans connaître la DB
    def __init__(self, repo: UserRepository):
        self.repo = repo
    def execute(self, user_id: str, new_email: str):
        user = self.repo.find_by_id(user_id)
        user.change_email(new_email)
        self.repo.save(user)

# Infrastructure — Adapter (implémentation concrète)
class PostgresUserRepository(UserRepository):   # dépend du Port, pas l'inverse
    def save(self, user: User): ...              # SQL ici
    def find_by_id(self, id: str) -> User: ...  # SQL ici
```

**Pourquoi cette architecture**

1. **Testabilité maximale** : on peut substituer `PostgresUserRepository` par un `InMemoryUserRepository` en test — sans toucher au use case ni au domain.
2. **Indépendance technologique** : changer de base de données (Postgres → MongoDB) = réécrire uniquement l'Adapter, jamais le Domain.
3. **Règle de dépendance** : le flux de contrôle peut aller vers l'extérieur (Application appelle Infrastructure), mais la **dépendance de code** va toujours vers l'intérieur via l'interface — c'est l'**Inversion de Dépendance (DIP)** appliquée à l'architecture entière.

**Lien avec SOLID** : la Clean Architecture est une application systématique du **DIP** à l'échelle macro. Les Ports sont les abstractions ; les Adapters sont les implémentations concrètes. Le Domain est le module de haut niveau qui ne dépend de rien.

---

## 13. Principes de Conception — DRY, KISS, YAGNI, TDD

---

### DRY — Don't Repeat Yourself

**La règle** : **chaque connaissance ou logique doit avoir une représentation unique et non ambiguë** dans le système. Si tu copies-colles du code, tu crées deux sources de vérité qui vont diverger.

**Le problème** : une règle de validation d'email écrite dans 3 endroits du code. Le jour où la règle change, tu en modifies 2 sur 3 → bug silencieux en production.

**La solution** : une seule fonction `validate_email()` référencée partout. Un seul endroit à modifier.

**Nuance à l'oral** : DRY ne signifie pas "ne jamais dupliquer de code". Il signifie "ne jamais dupliquer de *connaissance*". Deux boucles `for` qui font des choses différentes ne violent pas DRY. Deux fonctions qui encodent la même règle métier, si.

---

### KISS — Keep It Simple, Stupid

**La règle** : **préférer systématiquement la solution la plus simple** qui résout le problème. La complexité non justifiée est un bug différé.

**Le problème** : un développeur implémente un système de plugins extensible avec une factory, un registry et une interface abstraite… pour gérer deux types de fichiers qui n'évolueront jamais.

**La solution** : un `if/elif` de 5 lignes. Si le besoin évolue, on refactorise alors — pas avant.

**Point oral** : KISS et YAGNI se renforcent mutuellement. La complexité coûte à l'écriture, à la lecture, aux tests et à la maintenance. La charge de la preuve est sur la complexité, pas sur la simplicité.

---

### YAGNI — You Aren't Gonna Need It

**La règle** : **n'implémente pas une fonctionnalité avant d'en avoir un besoin réel et immédiat**. Ne pas coder pour des besoins hypothétiques.

**Le problème** : "On pourrait avoir besoin d'un système multi-tenant plus tard, autant le prévoir maintenant." → 3 semaines de développement pour une feature qui ne sera jamais utilisée, et une base de code que personne ne comprend.

**La solution** : implémenter exactement ce que le besoin actuel requiert. Refactoriser quand le besoin réel arrive — à ce moment, tu connaîtras les contraintes réelles.

**Lien avec SOLID/OCP** : YAGNI n'est pas en contradiction avec OCP. OCP dit "écris du code *fermé* à la modification" — pas "anticipe toutes les extensions possibles". On respecte OCP en rendant le code extensible *là où c'est prévisible*, pas partout.

---

### TDD — Test-Driven Development

**Définition** : méthodologie de développement où les **tests sont écrits avant le code de production**. On ne code que ce qui est nécessaire pour faire passer un test qui échoue.

**La boucle Red-Green-Refactor**

```
RED    → Écrire un test qui échoue (le comportement voulu n'existe pas encore)
GREEN  → Écrire le minimum de code pour faire passer le test
REFACTOR → Nettoyer le code sans casser les tests
           └──────────────────────────────▶ recommencer
```

**Pourquoi ce n'est pas "juste tester"**

- Le test écrit *avant* force à **définir l'interface** (nom de méthode, paramètres, valeur de retour) avant l'implémentation — on pense du point de vue de l'utilisateur du code.
- Le suite de tests devient un **filet de sécurité** : toute régression est détectée immédiatement.
- Le code TDD est naturellement **découplé et testable** — si c'est difficile à tester, c'est un signal que le design est mauvais.

**Les 3 types de tests à citer**

| Type | Scope | Vitesse | Exemple |
|---|---|---|---|
| **Unit** | Une fonction / une classe isolée | Millisecondes | `test_validate_email_rejects_missing_at()` |
| **Integration** | Plusieurs composants ensemble | Secondes | `test_user_creation_persists_to_db()` |
| **E2E** | Système complet, scénario utilisateur | Minutes | `test_login_then_checkout_flow()` |

**Pyramide des tests** : beaucoup d'unit tests (rapides, stables), moins d'integration, peu d'E2E (lents, fragiles). Inverser la pyramide = suite de tests lente et difficile à maintenir.

---

## 14. REST — Representational State Transfer

**Définition en une phrase** : REST est un **ensemble de 6 contraintes architecturales** définies par Roy Fielding ; une API est dite "RESTful" si et seulement si elle les respecte toutes (la 6ème étant optionnelle).

**Concept fondateur — Ressource vs Représentation** : on conçoit l'API autour de **noms de domaine identifiables par un URI** (ex : `/users/42`), non autour de fonctions (ex : `deleteUser()`). La ressource est l'entité abstraite ; le JSON renvoyé n'est qu'une **représentation de son état actuel**. On la manipule via les verbes HTTP standards (GET, POST, PUT, DELETE).

**Les 6 contraintes à citer**

| # | Contrainte | Bénéfice clé |
|---|---|---|
| 1 | **Client-Serveur** | Séparation claire UI / données → changer l'un sans toucher l'autre |
| 2 | **Stateless** | Chaque requête est auto-suffisante (token inclus) → **scalabilité** : n'importe quel serveur peut répondre |
| 3 | **Cacheable** | Le serveur déclare si la réponse est cachable → évite les requêtes SQL redondantes |
| 4 | **Uniform Interface** | Contrat standard : URI + JSON + HATEOAS (le serveur renvoie les liens vers les actions suivantes) |
| 5 | **Layered System** | Le client ignore s'il parle au serveur final ou à un intermédiaire (proxy, load balancer) |
| 6 | **Code-On-Demand** *(opt.)* | Le serveur peut envoyer du code exécutable (JS) — seule contrainte non obligatoire |

**Point oral sur Stateless** : c'est la clé de la scalabilité horizontale. Si la session était stockée côté serveur, chaque requête devrait revenir au **même** serveur — impossible à scaler. Stateless = n'importe quel nœud peut répondre.

**Idempotence HTTP** : GET, PUT, DELETE sont idempotents (appeler N fois = même résultat que 1 fois). POST ne l'est pas — il crée à chaque appel.

---

## 15. Monolithe vs Microservices

**Définition du problème** : comment découper et déployer une application ? Deux modèles s'opposent — le monolithe (tout dans un seul bloc) et les microservices (services indépendants communiquant entre eux).

---

### Monolithe

**Principe** : l'ensemble des fonctionnalités (authentification, paiement, catalogue, notifications) est **déployé en un seul artefact**. Un seul processus, une seule base de code, un seul déploiement.

**Avantages** :
- Simplicité de développement, de debug et de déploiement au démarrage.
- Pas de latence réseau entre les composants — les appels sont en mémoire.
- Transactions ACID natives — une opération peut toucher plusieurs domaines en une seule transaction.

**Limites** :
- Un bug dans un module peut faire tomber **toute** l'application.
- Scalabilité monolithique : on scale l'application entière même si seul le module "Catalogue" est sous charge.
- Plus l'équipe grandit, plus les conflits de merge et le couplage deviennent ingérables.

**Cas d'usage** : startup, MVP, petite équipe. Le monolithe est le point de départ rationnel — ne pas over-architecturer dès le début (YAGNI).

---

### Microservices

**Principe** : l'application est découpée en **services indépendants**, chacun responsable d'un domaine métier précis (User Service, Payment Service, Notification Service). Chaque service :
- A sa propre base de données (isolation des données).
- Est déployé, scalé et mis à jour indépendamment.
- Communique via **API REST, gRPC ou messaging asynchrone** (Kafka, RabbitMQ).

**Avantages** :
- **Scalabilité indépendante** : on scale uniquement le service sous charge.
- **Résilience** : un service qui tombe n'impacte pas les autres (avec des Circuit Breakers).
- **Autonomie des équipes** : chaque équipe possède son service, choisit sa stack, déploie à son rythme.

**Limites** :
- Complexité opérationnelle élevée : orchestration (Kubernetes), discovery de services, observabilité distribuée.
- **Transactions distribuées** : sans transaction ACID inter-services, on utilise le pattern **SAGA** (séquence de transactions locales compensables).
- Latence réseau entre services — chaque appel HTTP/gRPC a un coût vs un appel en mémoire.

**Cas d'usage** : grande équipe, produit mature, besoins de scalabilité différenciés par domaine.

---

### Tableau de décision

| Critère | Monolithe | Microservices |
|---|---|---|
| Taille d'équipe | Petite (< 10) | Grande (équipes par domaine) |
| Stade du produit | MVP / exploration | Produit mature et stable |
| Scalabilité | Globale uniquement | Par service, indépendante |
| Transactions | ACID natives | SAGA (compensations) |
| Complexité opérationnelle | Faible | Élevée (K8s, observabilité) |
| Déploiement | Simple | Indépendant par service |

**Point oral** : le passage de monolithe à microservices est une **décision d'architecture évolutive**, pas une règle absolue. Martin Fowler recommande de commencer par un **monolithe modulaire** (bien découpé en modules internes) et d'extraire en microservices uniquement quand les limites de scalabilité ou d'autonomie des équipes sont atteintes.

---

# SECTION III — GenAI

---

## 14. Le Transformer

**Définition en une phrase** : le Transformer est une architecture de deep learning basée sur le mécanisme de **self-attention**, qui permet de traiter les données en **parallèle** plutôt que séquentiellement — contrairement aux RNN qui lisent token par token.

**Pourquoi c'est révolutionnaire** : les RNN souffrent du problème de la **dépendance à longue distance** — l'information d'un mot lointain s'atténue au fil des étapes. Le Transformer capture ces dépendances **directement**, quelle que soit la distance, grâce à l'attention qui relie chaque token à tous les autres en une seule opération.

**Architecture générale**

```
         Encoder                         Decoder
  ┌─────────────────────┐       ┌──────────────────────────┐
  │  Input Embedding    │       │  Output Embedding        │
  │  + Positional Enc.  │       │  + Positional Enc.       │
  │  Multi-Head Attn    │──────▶│  Masked Multi-Head Attn  │
  │  Feed-Forward       │       │  Cross-Attention         │
  │  Layer Norm + Skip  │       │  Feed-Forward            │
  └─────────────────────┘       │  Layer Norm + Skip       │
                                └──────────────────────────┘
```

- **Encoder** : lit la séquence entière, extrait le contexte global → produit des représentations riches de chaque token.
- **Decoder** : génère la réponse token par token, en s'appuyant sur le contexte de l'Encoder et sur ce qu'il a déjà généré (Masked Attention).

---

### Composant 1 — Multi-Head Attention

**Rôle** : permettre au modèle de se concentrer sur **plusieurs types de relations** entre les tokens simultanément, en parallèle.

**Mécanisme** : on projette les tokens en `h` espaces différents (les "têtes"). Chaque tête calcule sa propre attention indépendamment, puis les résultats sont **concaténés** et reprojetés.

**Pourquoi plusieurs têtes ?** Chaque tête spécialise sa lecture :
- Une tête gère la **grammaire** (relation sujet–verbe)
- Une autre le **contexte sémantique** (de quoi parle-t-on)
- Une autre les **références pronominales** (à qui renvoie "il")

Un seul mécanisme d'attention ne pourrait capturer qu'un seul type de relation à la fois.

**Formule de l'attention (Scaled Dot-Product)** :

```
Attention(Q, K, V) = softmax(QKᵀ / √dₖ) · V
```

- **Q** (Query) : "quelle information je cherche ?"
- **K** (Key) : "quelle information j'offre ?"
- **V** (Value) : "quelle information je transmets si on me choisit ?"
- **√dₖ** : facteur d'échelle pour éviter que le produit scalaire explose et sature le softmax → gradients qui disparaissent.

---

### Composant 2 — Feed-Forward Networks

**Rôle** : réseau de neurones classique (deux couches linéaires + activation ReLU/GELU) appliqué à **chaque token indépendamment**, après l'attention.

**Pourquoi** : l'attention capture les *relations entre tokens* ; le Feed-Forward **affine la représentation individuelle** de chaque token en intégrant ces relations. C'est lui qui encode les connaissances factuelles dans les poids du modèle.

---

### Composant 3 — Layer Norm & Residual Connections

**Rôle** : stabilisation de l'entraînement sur des modèles très profonds.

- **Residual Connection (skip connection)** : `output = LayerNorm(x + sublayer(x))`. On additionne l'entrée et la sortie de chaque sous-couche → le gradient peut "court-circuiter" les couches profondes sans s'écraser (problème du *vanishing gradient*).
- **Layer Normalization** : normalise les activations à chaque couche → stabilise les valeurs, accélère la convergence.

**Sans ces mécanismes** : les modèles géants comme GPT (96 couches) ne pourraient pas apprendre — le signal se diluerait et les gradients disparaîtraient avant d'atteindre les premières couches.

---

### BERT vs GPT — Encoder-only vs Decoder-only

| | **BERT** | **GPT** |
|---|---|---|
| Architecture | Encoder seul | Decoder seul |
| Sens de lecture | **Bidirectionnel** — lit gauche ET droite simultanément | **Unidirectionnel** — lit de gauche à droite uniquement |
| Objectif de pré-entraînement | **MLM** (Masked Language Model) : prédit les tokens masqués | **CLM** (Causal Language Model) : prédit le token suivant |
| Masquage | Aucun (accès à toute la séquence) | **Causal mask** — un token ne voit que les tokens précédents |
| Usage | Compréhension : classification, NER, QA extractif | Génération : complétion de texte, dialogue, LLM |
| Analogie | Étudiant qui relit tout le paragraphe avant de répondre à un QCM | Écrivain qui improvise mot à mot sans revenir en arrière |

**Point oral clé** : BERT est bidirectionnel car il prédit des tokens *masqués au milieu* de la phrase — il peut regarder à gauche et à droite. GPT ne peut pas faire ça : il génère de gauche à droite, donc chaque token ne voit que son passé (causal masking). Cette contrainte est aussi sa force : elle le rend naturellement adapté à la génération.

---

## 15. RAG — Retrieval-Augmented Generation

**Définition en une phrase** : le RAG est une architecture qui augmente un LLM avec une **recherche d'information externe** — au lieu de se fier uniquement à ses paramètres appris à l'entraînement, le modèle reçoit dans son prompt des **extraits de documents pertinents** récupérés à la volée.

**Pourquoi c'est le standard en entreprise** : les LLM seuls hallucinent sur des données absentes de leur entraînement et ne connaissent pas les données privées ou récentes. Le RAG résout les deux : il **réduit les hallucinations** (le modèle répond sur la base de preuves fournies) et permet de travailler sur **n'importe quelle base documentaire privée, en temps réel**, sans réentraîner le modèle.

---

### Étape A — Indexing (Préparation hors-ligne)

Avant toute requête, les documents sont transformés en vecteurs stockables :

| Étape | Opération | Détail |
|---|---|---|
| **1. Chunking** | Segmentation des documents | Découpage en morceaux de ~500 tokens. Stratégie critique : trop petit → perte de contexte ; trop grand → bruit dans le prompt |
| **2. Embedding** | Transformation en vecteur | Un modèle d'embedding (ex : `text-embedding-3-small`) encode chaque chunk en un vecteur dense qui capture son **sens sémantique** |
| **3. Stockage** | Base de données vectorielle | Les vecteurs sont indexés dans une vector DB (ChromaDB, Pinecone, Milvus) pour une recherche par similarité en millisecondes |

---

### Étape B — Retrieval & Generation (En ligne, à chaque requête)

```
Question utilisateur
        │
        ▼
  [Embedding de la question]
        │
        ▼
  [Recherche vectorielle] ──▶ Top-K chunks les plus proches
        │
        ▼
  [Construction du prompt]
  "Contexte : [chunks]. Question : [question]. Réponds uniquement sur la base du contexte."
        │
        ▼
  [LLM génère la réponse]
```

1. **Retrieval** : la question est encodée en vecteur. La vector DB retourne les `k` chunks dont le vecteur est le plus proche (similarité cosinus ou produit scalaire).
2. **Augmentation** : les chunks sont injectés dans le prompt comme contexte explicite.
3. **Génération** : le LLM produit une réponse ancrée dans les preuves fournies, pas dans ses paramètres seuls.

---

### Bi-Encoder vs Cross-Encoder vs Reranker

**Le problème fondamental** : il faut arbitrer entre **vitesse** (comparer la question à des millions de chunks) et **précision** (comprendre finement si un chunk répond vraiment à la question).

**Bi-Encoder (le Retriever)**
- Encode la question et chaque document **séparément**, compare les vecteurs.
- **Avantage** : ultra-rapide — les vecteurs de documents sont pré-calculés, la recherche est en O(log n) via ANN.
- **Limite** : la question et le document ne se "voient" jamais directement → peut rater des nuances fines.

**Cross-Encoder (le Juge)**
- Prend la question ET le document **concaténés** en entrée d'un seul modèle. Calcule un score de pertinence token par token.
- **Avantage** : précision maximale — interaction complète entre question et document.
- **Limite** : impossible à scaler en retrieval (pas de pré-calcul possible, coût O(n) par requête).

**Pipeline de Reranking (le standard production)**

La combinaison des deux pour avoir le meilleur des deux mondes :

```
Vector DB (millions de chunks)
        │
        ▼  Bi-Encoder (rapide)
  Top-100 candidats
        │
        ▼  Cross-Encoder (précis)
  Reranking : réordonne les 100 selon pertinence réelle
        │
        ▼
  Top-3 à 5 chunks injectés dans le prompt LLM
```

**Point oral — "Lost in the Middle"** : les LLM ont tendance à accorder moins d'attention aux informations placées au milieu d'un long contexte. Le reranker adresse ce problème en deux temps : (1) il place les chunks les plus pertinents en tête de prompt, et (2) il réduit le nombre de chunks injectés — moins de contexte = moins de risque que l'information critique se perde. Effet de bord positif : moins de tokens → **coût d'inférence réduit**.

---

## 16. Fine-Tuning

**Définition en une phrase** : le fine-tuning consiste à reprendre un modèle **pré-entraîné** (qui sait déjà parler) et à continuer son entraînement sur un jeu de données **petit mais très ciblé**, pour lui faire acquérir un vocabulaire technique, un format de réponse précis ou des connaissances pointues absentes de son pré-entraînement.

**Analogie** : c'est envoyer un étudiant à culture générale immense faire une **spécialisation** — droit, médecine, code. Il ne repart pas de zéro ; il affine.

---

### Les 3 types de Fine-Tuning

| Type | Principe | Cas d'usage |
|---|---|---|
| **SFT** (Supervised Fine-Tuning) | Paires `Instruction → Réponse attendue` | Apprendre à suivre des instructions, adopter un format de sortie |
| **Instruction Tuning** | Variété de SFT sur des milliers de tâches différentes | Rendre le modèle polyvalent face à des instructions variées |
| **Domain-Specific Fine-Tuning** | Entraînement sur un corpus de textes bruts (ex : rapports juridiques) | Imprégner le modèle d'un jargon technique et de connaissances sectorielles |

---

### PEFT — Parameter-Efficient Fine-Tuning

**Problème** : entraîner l'intégralité des milliards de paramètres d'un LLM est prohibitif en coût GPU. Le PEFT résout ça en ne modifiant qu'**une fraction des paramètres**.

**LoRA (Low-Rank Adaptation)**

Au lieu de modifier directement la matrice de poids `W` du modèle, on lui **ajoute deux petites matrices** `A` et `B` de rang faible `r` :

```
W' = W₀ + BA     (B ∈ ℝ^{d×r}, A ∈ ℝ^{r×k}, r ≪ min(d,k))
```

- `W₀` est **gelé** — les poids originaux ne bougent pas.
- Seules `A` et `B` sont entraînées → 10 000× moins de paramètres à mettre à jour.
- À l'inférence, `BA` est fusionné dans `W₀` → **zéro surcoût de latence**.

**QLoRA** : LoRA + **quantification 4-bit (NF4)** des poids gelés. Permet d'entraîner un modèle de 70B sur un seul GPU grand public en réduisant l'empreinte mémoire de ~75 %.

---

### Pipeline de Fine-Tuning

```
1. Data Preparation  →  2. Training  →  3. Evaluation
```

1. **Data Preparation** *(80 % du travail)* : données de haute qualité, nettoyées, formatées en JSONL. La qualité du dataset détermine la qualité du modèle final — garbage in, garbage out.
2. **Training** : choix des hyperparamètres clés — learning rate (souvent 1e-4 à 1e-5 pour du fine-tuning), nombre d'époques, taille de batch, rang `r` de LoRA.
3. **Evaluation** : benchmarks standardisés (MMLU, HumanEval) ou **LLM-as-a-Judge** — un LLM évalue la qualité des réponses du modèle fine-tuné vs le modèle de base.

---

### Risque : l'Oubli Catastrophique (Catastrophic Forgetting)

**Définition** : à force de s'entraîner intensivement sur un domaine précis (ex : code Python), le modèle **dégrade ses capacités générales** — il perd sa fluidité en langage naturel ou oublie des connaissances hors-domaine.

**Mécanisme** : le gradient des nouvelles données écrase les poids qui encodaient les anciennes connaissances.

**Solution** : **mélanger des données générales** dans le jeu de fine-tuning (data mixing). Un ratio typique : ~10 % de données générales pour stabiliser les capacités acquises pendant le pré-entraînement.

**Point oral** : c'est pour cette raison que le SFT dans RLHF est toujours suivi d'une évaluation sur des benchmarks généraux — pour détecter toute régression avant le déploiement.

---

## 17. RLHF — Reinforcement Learning from Human Feedback

**Définition en une phrase** : le RLHF est la méthode d'**alignement** qui transforme un LLM brut (prédicteur de tokens) en un assistant dont le comportement est conforme aux préférences humaines — utile, honnête et inoffensif (**Helpful, Honest, Harmless**).

**Pourquoi c'est indispensable** : sans RLHF, le modèle optimise une seule chose — la vraisemblance du token suivant. Il peut donc répondre à des prompts nuisibles si la suite statistiquement probable du texte le justifie. Le RLHF injecte une notion de "bonne réponse" définie par des humains, et punit les réponses toxiques ou inutiles.

---

### Pipeline en 3 étapes

**Étape 1 — SFT (Supervised Fine-Tuning)**

Des humains rédigent des réponses exemplaires à un ensemble de prompts représentatifs. On fine-tune le modèle pré-entraîné sur ces paires `prompt → réponse idéale`. C'est la base comportementale : le modèle apprend à quoi ressemble une bonne réponse avant même que la boucle de renforcement commence.

**Étape 2 — Reward Model (Modèle de Récompense)**

Le modèle SFT génère plusieurs variantes de réponses (A, B, C, D) pour chaque prompt. Des annotateurs humains les **classent** de la meilleure à la moins bonne. On entraîne ensuite un second modèle — le **Reward Model** — à prédire le score qu'un humain attribuerait à une réponse donnée. Le Reward Model devient un **juge automatisé** qui encode le "goût humain" sous forme de signal scalaire.

**Étape 3 — Optimisation PPO (Proximal Policy Optimization)**

Le LLM (la "policy") génère des réponses. Le Reward Model attribue un score à chacune. L'algorithme **PPO** met à jour les paramètres du LLM pour maximiser ce score, tout en ajoutant une contrainte **KL-divergence** pour éviter que le modèle dévie trop du modèle SFT initial (ce qui produirait des réponses absurdes mais bien notées — le "reward hacking").

```
LLM génère une réponse
        │
        ▼
Reward Model attribue un score
        │
        ▼
PPO met à jour le LLM pour maximiser le score
(contrainte KL : ne pas trop s'éloigner du SFT)
        │
        └──────────────────────────────▶ boucle
```

**Résumé oral** : "On entraîne d'abord un Reward Model sur des classements humains, puis on guide le LLM via PPO pour maximiser ce score — en ajoutant une contrainte KL pour éviter le reward hacking. L'objectif est d'aligner le comportement du modèle sur les préférences humaines : Helpful, Honest, Harmless."

---

### DPO — Direct Preference Optimization

**Motivation** : le pipeline RLHF est complexe — entraîner et maintenir un Reward Model séparé, stabiliser PPO (sensible aux hyperparamètres) est coûteux et fragile.

**Principe** : le DPO supprime le Reward Model et PPO. Il optimise **directement** le LLM sur les paires de préférences `(réponse préférée, réponse rejetée)` via une loss dérivée analytiquement de l'objectif de renforcement.

| | **RLHF (PPO)** | **DPO** |
|---|---|---|
| Reward Model | Entraîné séparément | Implicite dans la loss |
| Complexité | Élevée (2 modèles + boucle RL) | Simple (1 modèle, 1 loss supervisée) |
| Stabilité | Sensible aux hyperparamètres PPO | Plus stable, plus reproductible |
| Adoption | GPT-4, Claude (historique) | Llama 3, Mistral (récent) |

**Point oral** : DPO est plus simple mais suppose que les préférences humaines peuvent être capturées directement dans la loss sans passer par un signal de récompense explicite. En pratique, les deux approches produisent des résultats comparables, ce qui explique l'adoption croissante du DPO en production.

---

## 18. Évaluation d'un RAG — RAGas, LLM-as-a-Judge, SLM

**Principe** : évaluer un RAG ne se limite pas à une précision globale. Il faut **décorréler** la qualité du Retriever de celle du Generator — une bonne réponse peut venir d'un mauvais retrieval (chance), et un bon retrieval peut être mal exploité par le LLM. Le framework RAGas structure cette évaluation en métriques orthogonales.

---

### Métriques opérationnelles

| Métrique | Définition | Pourquoi ça compte |
|---|---|---|
| **Latence / TTFT** | Time To First Token — délai avant le premier token généré | Critique pour l'UX en temps réel |
| **Token Usage** | Nombre de tokens consommés par requête | Budget direct : un contexte RAG volumineux coûte cher |
| **Throughput** | Requêtes traitées par seconde | Dimensionnement de l'infrastructure |

---

### Framework RAGas — le "RAG Triad"

Trois métriques pour isoler précisément où le système échoue :

| Métrique | Question posée | Ce qu'elle détecte |
|---|---|---|
| **Faithfulness / Groundedness** | La réponse est-elle ancrée dans les documents fournis ? | Hallucinations — le modèle invente ou utilise sa connaissance paramétrique |
| **Answer Relevance** | La réponse répond-elle vraiment à la question ? | Réponses correctes mais hors-sujet |
| **Context Precision / Recall** | Le Retriever a-t-il ramené les bons documents ? | Défaillance du Retriever, indépendamment du Generator |

**Point oral** : si la Faithfulness est basse mais le Context Recall est élevé → le problème est dans le Generator (il ignore les documents). Si le Context Recall est bas → le problème est dans le Retriever. Les deux sont diagnostiquables séparément.

---

### Le problème de la Connaissance Paramétrique

**Définition** : le modèle répond à partir de ses poids appris à l'entraînement plutôt qu'à partir du contexte injecté — "Selon mes connaissances générales..." — au lieu d'exploiter les documents RAG.

**C'est le biais fondamental des LLM en production** : leur connaissance interne entre en compétition avec le contexte externe, surtout quand les documents contredisent ce qu'ils ont appris.

**3 solutions à citer**

1. **Prompt Engineering strict** : forcer le comportement via le system prompt — *"Tu dois répondre uniquement en utilisant le contexte fourni. Si l'information n'y est pas, réponds que tu ne sais pas."* La règle doit être explicite et répétée.
2. **Few-Shot Prompting** : fournir des exemples dans le prompt où le modèle ignore délibérément sa connaissance interne et s'appuie sur le contexte — conditionner le comportement par l'exemple.
3. **LLM-as-a-Judge** : un modèle "juge" plus puissant (ex : GPT-4o) évalue les réponses du modèle de production et détecte automatiquement si une réponse contient des informations absentes du contexte injecté.

---

### SLM vs LLM-as-a-Judge

**SLM (Small Language Model)** : modèles légers déployés en local pour des raisons de vitesse et de coût. Mais ils suivent moins fidèlement les instructions complexes et sont plus sujets à la connaissance paramétrique — le risque de non-groundedness est plus élevé.

**LLM-as-a-Judge** : pipeline d'évaluation automatisée où un modèle expert note les réponses du modèle de production selon une grille de critères formalisés (Groundedness, Pertinence, Ton, Précision). C'est la méthode moderne pour scaler l'évaluation sans annotateurs humains à chaque itération.

```
Requête utilisateur
        │
        ▼
    Agent (SLM / LLM de prod)  ──▶  Réponse générée
                                            │
                                            ▼
                                    Juge (LLM expert)
                                    Score : Groundedness / Relevance / Tone
```

---

### Imbalanced Dataset en évaluation GenAI

**Problème** : si le jeu d'évaluation contient trop de questions "faciles" et trop peu de cas limites (edge cases), les métriques semblent bonnes alors que le système échoue précisément là où c'est critique.

**Solution — Synthèse de données** : utiliser un LLM pour **générer automatiquement** des questions complexes, contradictoires ou ambiguës à partir des documents réels. Cela permet de peupler les zones rares du dataset et de tester la robustesse du RAG sur des cas que les annotateurs humains n'auraient pas pensé à couvrir.

---

## 19. MCP — Model Context Protocol

**Définition en une phrase** : le MCP est un **protocole open-source de standardisation** qui permet à un système IA de se connecter à des sources de données et des outils externes de manière **sécurisée et uniforme** — sans réécrire du code d'intégration à chaque nouvel outil.

**Analogie** : c'est le **USB des agents IA**. Avant USB, chaque périphérique nécessitait son propre port et pilote. Le MCP joue le même rôle : un contrat universel entre les agents et les systèmes externes.

---

### Architecture : Client vs Serveur

**Point critique à l'oral** : le LLM ne connaît pas le protocole MCP et ne se connecte jamais directement au serveur. C'est le **Client** qui fait l'intermédiaire.

| | **Client MCP (ton système IA)** | **Serveur MCP (le système externe)** |
|---|---|---|
| Rôle | Hôte qui maintient la connexion et expose les capacités au LLM | Possède les données ou les outils |
| Exemples | Claude Desktop, application Python LangGraph | Base SQL, API GitHub, système de fichiers local |
| Qui le contrôle | Toi (le développeur de l'agent) | Le fournisseur du service externe |

---

### Les 3 types de ressources exposées par le serveur

| Type | Nature | Exemple |
|---|---|---|
| **Resources** | Données en **lecture seule** — documents, logs, fichiers | "Voici le contenu du dossier Projet, lis-le si nécessaire" |
| **Tools / Functions** | **Actions exécutables** — fonctions appelables par l'agent | "Envoie un mail", "Exécute cette requête SQL" |
| **Prompts** | Templates pré-configurés fournis par le serveur | Modèles d'interaction pour guider l'agent sur ce service |

---

### Flux de communication

```
Utilisateur pose une question
        │
        ▼
Client IA  ──interroge──▶  Serveur MCP
                           "Quels outils/ressources sont disponibles ?"
        │
        ◀── liste des outils ──
        │
        ▼
Client IA envoie la liste au LLM
        │
        ▼
LLM décide : "J'ai besoin de l'outil X"
        │
        ▼
Client IA exécute l'action sur le Serveur MCP
        │
        ◀── résultat ──
        │
        ▼
Client IA renvoie le résultat au LLM → génération de la réponse finale
```

**Règle à retenir** : le flux de contrôle est `LLM → Client → Serveur`. Le LLM délègue, il ne se connecte jamais directement.

---

### Pourquoi c'est une rupture architecturale

**Avant MCP** : chaque intégration (Slack, Jira, SQL, GitHub) nécessitait un code spécifique côté agent. Ajouter un outil = réécrire de l'intégration.

**Avec MCP** :
- Le serveur est écrit **une seule fois** selon le protocole standard.
- **N'importe quel client compatible** peut s'y connecter immédiatement — pas de re-développement.
- **Découplage total** : la logique de l'agent est indépendante de l'implémentation technique des outils.
- **Sécurité** : le serveur contrôle les droits d'accès à ses ressources. Le LLM ne peut pas outrepasser les permissions définies côté serveur — le Client applique ces restrictions avant d'exécuter quoi que ce soit.

**Lien avec l'architecture logicielle** : le MCP est une application du principe **Ports & Adapters** à l'échelle des agents IA. Le Client est le Port (interface abstraite) ; chaque Serveur MCP est un Adapter (implémentation concrète d'un service externe).

---

# SECTION IV — Machine Learning Fundamentals

---

## 20. Les 3 types de problèmes ML

| Type | Définition | Exemples réels |
|---|---|---|
| **Régression** | Prédire une **valeur continue** | Prix d'un appartement, température demain, durée d'un vol |
| **Classification** | Prédire une **classe discrète** | Spam ou non, diagnostic médical, sentiment positif/négatif |
| **Clustering** | **Regrouper** des données sans étiquette | Segmentation clients, détection de communautés, anomalies |

---

## 21. Modèles Classiques — Comment chacun fonctionne

---

### Régression Linéaire

**Principe** : trouver la droite (ou hyperplan) `ŷ = θ₀ + θ₁x₁ + ... + θₙxₙ` qui minimise l'erreur quadratique entre les prédictions et les vraies valeurs. Les paramètres θ sont estimés par la **Normal Equation** `θ̂ = (XᵀX)⁻¹Xᵀy` ou par descente de gradient.

**Hypothèse forte** : relation linéaire entre les features et la cible. Si la relation est non-linéaire → sous-performance.

**Cas réel** : prédire le prix d'un appartement en fonction de la surface, du nombre de pièces et de la localisation.

---

### Régression Logistique *(Classification malgré le nom)*

**Principe** : modèle de classification binaire. Il applique une fonction **sigmoïde** sur une combinaison linéaire des features pour produire une probabilité entre 0 et 1 : `p = σ(Xθ) = 1 / (1 + e^{-Xθ})`. On classe à 1 si p > 0.5.

**Optimisation** : minimise la **Binary Cross-Entropy** (pas le MSE — la MSE produit une loss non-convexe sur la sigmoïde).

**Cas réel** : détection de spam (spam / pas spam), scoring de crédit.

---

### Decision Tree

**Principe** : partitionne récursivement l'espace des features en posant des questions binaires (`age > 30 ?`). À chaque nœud, on choisit le split qui maximise le **gain d'information** (réduction d'entropie ou impureté de Gini).

**Avantage** : interprétable, pas de normalisation requise.
**Limite** : très sujet à l'**overfitting** — un arbre profond mémorise les données d'entraînement.

---

### Random Forest (Bagging)

**Principe** : ensemble de `N` Decision Trees entraînés en **parallèle** sur des sous-ensembles aléatoires des données (bootstrap) et des features. La prédiction finale est la **moyenne** (régression) ou le **vote majoritaire** (classification).

**Pourquoi ça marche** : chaque arbre est biaisé différemment. En moyennant, les erreurs se compensent → réduction de la **variance** sans augmenter le biais.

**Cas réel** : scoring de risque crédit, détection de fraude, importance de features.

---

### Gradient Boosting (XGBoost, LightGBM)

**Principe** : ensemble de trees entraînés **séquentiellement**. Chaque nouvel arbre apprend à corriger les **résidus** (erreurs) du modèle précédent. La prédiction finale est la somme pondérée de tous les arbres.

**Différence clé avec Random Forest** : les arbres sont construits en séquence (pas en parallèle) — chaque arbre dépend de l'erreur du précédent.

**Avantage** : souvent plus performant que Random Forest sur des données tabulaires.
**Risque** : plus susceptible d'overfitter si mal régularisé (paramètres `max_depth`, `learning_rate`, `n_estimators`).

**Cas réel** : compétitions Kaggle sur données tabulaires, scoring crédit, prévision de demande.

---

### SVM — Support Vector Machine

**Principe** : trouver l'**hyperplan de séparation** qui maximise la **marge** entre les deux classes (distance entre l'hyperplan et les points les plus proches — les *support vectors*). Pour les données non-linéairement séparables : **kernel trick** (RBF, polynomial) qui projette dans un espace de dimension supérieure sans calculer explicitement la projection.

**Avantage** : efficace en haute dimension, robuste aux outliers.
**Limite** : ne passe pas à l'échelle sur de très grands datasets (coût O(n²) à O(n³)).

---

### K-Means (Clustering)

**Principe** : algorithme itératif en 3 étapes — (1) initialiser K centroides aléatoirement, (2) assigner chaque point au centroïde le plus proche, (3) recalculer les centroides comme moyenne du cluster. Répéter jusqu'à convergence.

**Limite** : K doit être fixé à l'avance. Sensible aux outliers et aux formes de clusters non-convexes. Utiliser la **méthode du coude** (Elbow Method) pour choisir K.

**Cas réel** : segmentation clients (high-value / at-risk / dormants), compression d'images.

---

### DBSCAN (Clustering)

**Principe** : regroupe les points **denses** sans fixer K à l'avance. Un point est *core* si au moins `MinPts` voisins sont dans un rayon `ε`. Les points inatteignables depuis un core sont classés comme **noise** (outliers).

**Avantage vs K-Means** : détecte des clusters de forme arbitraire, identifie automatiquement les outliers.
**Cas réel** : détection d'anomalies, géolocalisation (regrouper des zones urbaines).

---

## 22. Bagging vs Boosting vs Stacking

| | **Bagging** | **Boosting** | **Stacking** |
|---|---|---|---|
| Entraînement | Parallèle | Séquentiel | Deux niveaux |
| Objectif | Réduire la **variance** | Réduire le **biais** | Combiner des modèles hétérogènes |
| Exemple | Random Forest | XGBoost, LightGBM, AdaBoost | LR qui combine RF + SVM + XGB |
| Risque | Moins d'overfitting | Plus d'overfitting si mal réglé | Complexité, coût de calcul |

**Mémo oral** : Bagging = modèles indépendants en parallèle → réduit la variance. Boosting = modèles dépendants en séquence → réduit le biais. Stacking = méta-modèle qui apprend à combiner des modèles de base hétérogènes.

---

## 23. Pipeline ML — Les étapes de A à Z

```
1. Collecte & Compréhension des données
        │
        ▼
2. Preprocessing
   • Valeurs manquantes (imputation médiane/mode ou suppression)
   • Encodage des catégorielles (OneHot, Label Encoding, Target Encoding)
   • Normalisation / Standardisation (MinMax, StandardScaler)
        │
        ▼
3. Feature Engineering
   • Création de nouvelles features (interactions, ratios)
   • Sélection de features (corrélation, importance Random Forest, L1)
        │
        ▼
4. Séparation Train / Validation / Test
   • Train : entraîner le modèle
   • Validation : tuner les hyperparamètres
   • Test : évaluation finale — ne jamais toucher avant la fin
        │
        ▼
5. Entraînement + Hyperparameter Tuning
   (GridSearch, RandomSearch, Optuna/Bayesian Optimization)
        │
        ▼
6. Évaluation (métriques adaptées au problème)
        │
        ▼
7. Déploiement + Monitoring (drift détection)
```

**Point oral** : le jeu de test est **sacré** — le regarder avant la fin introduit un biais de sélection. Tout le tuning se fait sur la validation (ou cross-validation). Le test ne sert qu'à estimer la performance en production.

---

## 24. Overfitting, Underfitting & Compromis Biais-Variance

**Biais** : erreur due à des hypothèses trop simplistes. Le modèle ne capture pas la complexité des données → **underfitting** (mauvaise performance train ET test).

**Variance** : erreur due à une sensibilité excessive aux fluctuations du jeu d'entraînement. Le modèle mémorise le bruit → **overfitting** (excellente performance train, mauvaise en test).

```
Complexité du modèle →

Erreur totale = Biais² + Variance + Bruit irréductible

Modèle simple  : biais élevé, variance faible   → underfitting
Modèle complexe: biais faible, variance élevée  → overfitting
Optimum        : compromis entre les deux
```

**Solutions contre l'overfitting**

| Technique | Mécanisme |
|---|---|
| **Régularisation L1 (Lasso)** | Pénalise ‖θ‖₁ → met certains coefficients à exactement 0 → sélection de features |
| **Régularisation L2 (Ridge)** | Pénalise ‖θ‖₂² → réduit tous les coefficients sans en éliminer |
| **Dropout** | Désactive aléatoirement des neurones à l'entraînement → force la redondance |
| **Early Stopping** | Arrête l'entraînement quand la loss de validation remonte |
| **Plus de données** | Réduit la variance — la solution la plus efficace quand elle est possible |

---

## 25. Cross-Validation

**Problème de la simple séparation train/test** : selon le tirage aléatoire, les métriques peuvent varier significativement — on ne sait pas si la performance mesurée est représentative.

**K-Fold Cross-Validation** : on divise les données en K blocs égaux. On entraîne K fois en utilisant K-1 blocs comme train et 1 bloc comme validation, en rotation. La métrique finale est la **moyenne des K scores**.

```
K=5 :
Fold 1 : [VAL][TRN][TRN][TRN][TRN]
Fold 2 : [TRN][VAL][TRN][TRN][TRN]
Fold 3 : [TRN][TRN][VAL][TRN][TRN]
Fold 4 : [TRN][TRN][TRN][VAL][TRN]
Fold 5 : [TRN][TRN][TRN][TRN][VAL]
         ──────────────────────────
         Score final = moyenne des 5 scores ± écart-type
```

**Stratified K-Fold** : préserve la proportion des classes dans chaque fold — indispensable sur des datasets déséquilibrés pour éviter qu'un fold ne contienne presque aucun exemple de la classe minoritaire.

---

## 26. Métriques d'Évaluation

---

### Métriques de Régression

| Métrique | Formule | Interprétation |
|---|---|---|
| **MAE** | `mean(|y - ŷ|)` | Erreur moyenne absolue — même unité que la cible, robuste aux outliers |
| **MSE** | `mean((y - ŷ)²)` | Pénalise fortement les grandes erreurs — sensible aux outliers |
| **RMSE** | `√MSE` | Même unité que la cible, conserve la sensibilité aux outliers |
| **R²** | `1 - SS_res/SS_tot` | Part de variance expliquée. R²=1 → parfait, R²=0 → modèle nul |

**Quand utiliser MAE vs RMSE** : MAE si les outliers ne doivent pas être sur-pénalisés (prévision de stock). RMSE si les grandes erreurs sont inacceptables (médical, sécurité).

---

### Métriques de Classification — La Matrice de Confusion

```
                  Prédit Positif   Prédit Négatif
Réel Positif   |      TP          |      FN       |
Réel Négatif   |      FP          |      TN       |
```

| Métrique | Formule | Ce qu'elle mesure |
|---|---|---|
| **Accuracy** | `(TP+TN) / total` | % de prédictions correctes — **trompeuse sur datasets déséquilibrés** |
| **Precision** | `TP / (TP+FP)` | Parmi les prédits positifs, combien sont vrais ? (évite les faux positifs) |
| **Recall** | `TP / (TP+FN)` | Parmi les vrais positifs, combien sont détectés ? (évite les faux négatifs) |
| **F1-Score** | `2 × P×R / (P+R)` | Moyenne harmonique Precision/Recall — équilibre les deux |
| **F0.5-Score** | `(1+0.25) × P×R / (0.25×P+R)` | Favorise la **Precision** sur le Recall (β < 1) |
| **F2-Score** | `(1+4) × P×R / (4×P+R)` | Favorise le **Recall** sur la Precision (β > 1) |

**Cas réels pour choisir la métrique**

| Contexte | Métrique | Pourquoi |
|---|---|---|
| Détection de cancer | **Recall** maximal | Un faux négatif (manquer un cancer) est catastrophique |
| Filtre anti-spam | **Precision** haute | Un faux positif (mail légitime en spam) est très gênant |
| Détection de fraude | **F1 ou F2** | On veut détecter les fraudes sans trop d'alarmes inutiles |
| Recommandation (top-K) | **F0.5** | On préfère peu de recommandations mais pertinentes |

---

### Courbe ROC & AUC

**ROC (Receiver Operating Characteristic)** : courbe qui trace le **Recall (TPR)** en fonction du **Taux de Faux Positifs (FPR = FP/(FP+TN))** pour tous les seuils de décision possibles.

**AUC (Area Under the Curve)** : aire sous la courbe ROC.

| AUC | Interprétation |
|---|---|
| 1.0 | Classifieur parfait |
| 0.5 | Classifieur aléatoire (diagonale) |
| < 0.5 | Pire que le hasard (labels inversés) |

**Point oral** : l'AUC mesure la capacité du modèle à **ordonner** les exemples positifs avant les négatifs, indépendamment du seuil choisi. C'est pourquoi c'est la métrique de référence pour comparer des modèles sur des datasets déséquilibrés — contrairement à l'accuracy qui peut être trompeuse (un modèle qui prédit toujours "négatif" obtient 99% d'accuracy si 1% seulement des exemples sont positifs).

---

# SECTION V — Réseau & Sécurité

---

## 27. Modèle OSI — Les 7 couches

**Définition** : le modèle OSI (Open Systems Interconnection) est un **cadre conceptuel en 7 couches** qui standardise la communication entre systèmes réseau. Chaque couche a une responsabilité précise et ne communique qu'avec les couches adjacentes.

| # | Couche | Rôle | Protocoles / Exemples |
|---|---|---|---|
| 7 | **Application** | Interface avec l'utilisateur et les applications | HTTP, HTTPS, FTP, DNS, SMTP |
| 6 | **Présentation** | Encodage, chiffrement, compression des données | TLS/SSL, JPEG, UTF-8 |
| 5 | **Session** | Établissement, gestion et fermeture des sessions | WebSockets (établissement) |
| 4 | **Transport** | Transmission fiable (ou non) de bout en bout | **TCP**, **UDP** |
| 3 | **Réseau** | Routage des paquets entre réseaux | IP, ICMP, routeurs |
| 2 | **Liaison** | Transfert de trames entre nœuds adjacents | Ethernet, MAC, switches |
| 1 | **Physique** | Transmission des bits sur le support physique | Câbles, Wi-Fi, fibre optique |

**Point oral** : en pratique, on raisonne surtout sur les couches 3-4-7. Quand tu parles d'une API HTTP, tu es couche 7. Quand tu parles de TCP vs UDP, tu es couche 4. Quand tu parles d'IP et de routage, tu es couche 3.

---

## 28. TCP vs UDP

**Problème fondamental** : comment transmettre des données sur un réseau non fiable (paquets pouvant se perdre, se réordonner, se dupliquer) ?

### TCP — Transmission Control Protocol

**Principe** : protocole **orienté connexion**. Avant tout échange, un **handshake en 3 étapes** établit la connexion :

```
Client ──SYN──────────▶ Serveur
Client ◀──SYN-ACK───── Serveur
Client ──ACK──────────▶ Serveur
       [connexion établie]
```

**Garanties** :
- **Fiabilité** : chaque paquet est accusé de réception (ACK). Si pas d'ACK → retransmission.
- **Ordre** : les paquets sont réassemblés dans le bon ordre (numéros de séquence).
- **Contrôle de flux** : le récepteur indique sa capacité d'absorption (fenêtre glissante).
- **Contrôle de congestion** : AIMD (Additive Increase, Multiplicative Decrease) — TCP ralentit si le réseau est saturé.

**Cas d'usage** : HTTP/HTTPS, transfert de fichiers, emails — tout ce qui ne peut pas se permettre de perdre des données.

---

### UDP — User Datagram Protocol

**Principe** : protocole **sans connexion**. On envoie les paquets (datagrammes) sans établir de connexion, sans accusé de réception, sans garantie d'ordre.

**Avantage** : **latence minimale** — pas de handshake, pas d'attente d'ACK.

**Inconvénient** : paquets peuvent se perdre, arriver dans le désordre ou être dupliqués. C'est à l'application de gérer (ou d'ignorer) ces cas.

**Cas d'usage** : streaming vidéo/audio (une frame perdue est moins grave qu'un lag), jeux en ligne, DNS, VoIP.

| | **TCP** | **UDP** |
|---|---|---|
| Connexion | Oui (3-way handshake) | Non |
| Fiabilité | Garantie (ACK + retransmission) | Aucune |
| Ordre | Garanti | Non garanti |
| Latence | Plus élevée | Minimale |
| Usage | HTTP, fichiers, emails | Streaming, gaming, DNS |

---

## 29. HTTP / HTTPS / TLS

### HTTP — HyperText Transfer Protocol

**Principe** : protocole de la couche Application (OSI 7) basé sur un modèle **requête-réponse**. Le client envoie une requête avec un verbe (GET, POST, PUT, DELETE, PATCH), le serveur répond avec un code de statut et un body.

**Codes de statut clés** :

| Famille | Signification | Exemples |
|---|---|---|
| 2xx | Succès | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirection | 301 Moved Permanently, 304 Not Modified |
| 4xx | Erreur client | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests |
| 5xx | Erreur serveur | 500 Internal Server Error, 503 Service Unavailable |

**Versions** :
- **HTTP/1.1** : une requête à la fois par connexion (head-of-line blocking). Keep-Alive permet de réutiliser la connexion TCP.
- **HTTP/2** : multiplexage — plusieurs requêtes en parallèle sur une seule connexion TCP. Compression des headers (HPACK).
- **HTTP/3** : remplace TCP par **QUIC** (basé sur UDP) — élimine le head-of-line blocking au niveau transport, meilleure performance sur réseaux instables.

---

### HTTPS & TLS — Transport Layer Security

**Définition** : HTTPS = HTTP + **TLS**. TLS opère à la couche Présentation (OSI 6) et chiffre le canal de communication avant tout échange applicatif.

**TLS 1.3 — Handshake simplifié**

```
Client ──ClientHello (suites cryptographiques supportées)──▶ Serveur
Client ◀──ServerHello + Certificat + ServerFinished────────  Serveur
Client ──ClientFinished (clé de session dérivée)───────────▶ Serveur
       [canal chiffré établi — 1 aller-retour seulement]
```

**Mécanismes clés** :
- **ECDHE** (Elliptic Curve Diffie-Hellman Ephemeral) : échange de clé sans jamais transmettre la clé secrète sur le réseau.
- **Forward Secrecy** : chaque session génère des clés éphémères. Si la clé privée du serveur est compromise à l'avenir, les sessions passées restent **impossibles à déchiffrer**.
- **Certificat X.509** : le serveur prouve son identité via un certificat signé par une Autorité de Certification (CA) de confiance. Le client vérifie la chaîne de confiance.

---

## 30. WebSockets vs Webhooks

**Contexte** : HTTP est un protocole requête-réponse — le client initie toujours. Comment gérer les communications en temps réel ou les notifications asynchrones ?

---

### WebSockets

**Principe** : protocole qui établit une connexion **bidirectionnelle persistante** entre client et serveur sur une seule connexion TCP. Après un handshake HTTP initial (upgrade), les deux parties peuvent envoyer des messages **à tout moment**, sans nouvelle requête.

```
Client ──HTTP Upgrade: websocket──▶ Serveur
Client ◀──101 Switching Protocols── Serveur
       [connexion WebSocket ouverte]
Client ◀────────────────────────── Serveur  (push du serveur)
Client ────────────────────────────▶ Serveur (message du client)
```

**Cas d'usage** : chat en temps réel, tableaux de bord live, jeux multijoueurs, cours de bourse en direct.

**Avantage vs polling** : pas de requêtes HTTP répétées à intervalles réguliers — le serveur pousse l'information dès qu'elle est disponible → latence minimale, économie de bande passante.

---

### Webhooks

**Principe** : mécanisme de **notification push HTTP**. Au lieu que le client interroge le serveur régulièrement ("est-ce qu'il y a du nouveau ?"), le serveur appelle **directement** une URL fournie par le client dès qu'un événement se produit.

```
Serveur externe ──POST /webhook──▶ Ton serveur
                  {event: "payment.success", amount: 99}
```

**Cas d'usage** : notifications de paiement Stripe, événements GitHub (push, PR), alertes monitoring.

**Différence clé avec WebSockets** :

| | **WebSocket** | **Webhook** |
|---|---|---|
| Connexion | Persistante, bidirectionnelle | Stateless, unidirectionnelle |
| Initiateur | Client (puis les deux) | Serveur externe |
| Usage | Temps réel continu | Événements ponctuels asynchrones |
| Infrastructure | Connexion maintenue en mémoire | Simple endpoint HTTP |

---

## 31. JWT & Authentification

### JWT — JSON Web Token

**Définition** : standard open (RFC 7519) pour transmettre des informations de manière sécurisée entre parties sous forme de **token auto-porteur** — le serveur n'a pas besoin de stocker la session, toutes les informations sont dans le token.

**Structure — 3 parties séparées par des points** :

```
Header.Payload.Signature

eyJhbGciOiJIUzI1NiJ9  .  eyJ1c2VyX2lkIjoxMjN9  .  SflKxwRJSMeKKF2QT4fwpMeJ
      Header                    Payload                    Signature
  (algo + type)            (claims : userId,            (HMAC ou RSA
                            role, exp...)               du Header+Payload)
```

- **Header** : algorithme de signature (`HS256` ou `RS256`) + type (`JWT`).
- **Payload** : claims — informations utiles (`user_id`, `role`, `exp` pour l'expiration). **Non chiffré**, juste encodé en Base64 → ne jamais mettre de données sensibles.
- **Signature** : garantit que le token n'a pas été altéré. Calculée avec la clé secrète (HS256) ou la clé privée (RS256).

**HS256 vs RS256**

| | **HS256** (symétrique) | **RS256** (asymétrique) |
|---|---|---|
| Clé | Une clé secrète partagée | Paire clé privée / clé publique |
| Vérification | Même clé pour signer et vérifier | N'importe qui avec la clé publique peut vérifier |
| Usage | Service unique, architecture simple | Microservices, OAuth2, fédération d'identité |

**Flux JWT typique** :

```
1. Utilisateur s'authentifie (login + password)
2. Serveur génère un JWT signé et le retourne
3. Client stocke le JWT (localStorage ou cookie HttpOnly)
4. Chaque requête inclut le JWT dans le header :
   Authorization: Bearer <token>
5. Serveur vérifie la signature → pas besoin de BDD pour chaque requête
```

**Limites** : un JWT ne peut pas être invalidé avant son expiration (stateless). Solution : **refresh tokens** (courte durée pour le JWT, longue durée pour le refresh) ou blacklist en cache Redis.

---

### OAuth2 — Délégation d'autorisation

**Définition** : protocole de **délégation d'autorisation** — permet à une application tierce d'accéder à des ressources d'un utilisateur sans que celui-ci ne lui communique ses identifiants.

**Flux Authorization Code + PKCE** (le standard pour les apps web/mobile) :

```
1. L'app redirige l'utilisateur vers le serveur d'autorisation (Google, GitHub...)
2. L'utilisateur s'authentifie et consent
3. Le serveur d'autorisation retourne un Authorization Code à l'app
4. L'app échange ce code contre un Access Token (+ Refresh Token)
5. L'app utilise l'Access Token pour appeler l'API protégée
```

**PKCE** (Proof Key for Code Exchange) : extension qui protège contre l'interception du code d'autorisation — indispensable pour les clients publics (SPA, mobile) qui ne peuvent pas garder un secret côté client.

**Différence OAuth2 vs OpenID Connect** : OAuth2 gère l'**autorisation** (accès à des ressources). OpenID Connect (OIDC) ajoute une couche d'**authentification** (identité de l'utilisateur) via un `id_token` JWT.

---

# SECTION VI — Infrastructure & DevOps

---

## 32. Docker — Containerisation

**Définition en une phrase** : Docker est un outil de **containerisation** qui empaquète une application avec toutes ses dépendances dans une unité isolée et portable — le container — qui s'exécute de manière identique sur n'importe quel environnement.

**Problème résolu** : "Ça marche sur ma machine." Un container encapsule le code, le runtime, les bibliothèques et la configuration système → **l'environnement est reproductible** du laptop au serveur de production.

---

### VM vs Container

| | **Machine Virtuelle** | **Container** |
|---|---|---|
| Isolation | OS complet virtualisé | Partage le kernel de l'hôte |
| Poids | Go (image OS complète) | Mo (uniquement les diffs) |
| Démarrage | Minutes | Secondes |
| Overhead | Élevé (hyperviseur) | Minimal |

**Pourquoi les containers sont plus légers** : ils s'appuient sur deux mécanismes du kernel Linux :
- **Namespaces** : isolent les ressources (PID, réseau, système de fichiers, utilisateurs) — le container croit être seul sur la machine.
- **cgroups** : limitent les ressources consommées (CPU, RAM, I/O) — le container ne peut pas monopoliser l'hôte.

---

### Images & Layers — OverlayFS

Une **image Docker** est un empilement de couches (layers) en lecture seule. Chaque instruction `Dockerfile` crée une nouvelle couche :

```dockerfile
FROM python:3.11-slim        # Layer 1 : image de base
WORKDIR /app                 # Layer 2 : répertoire de travail
COPY requirements.txt .      # Layer 3 : fichier de dépendances
RUN pip install -r requirements.txt  # Layer 4 : installation
COPY . .                     # Layer 5 : code source
CMD ["uvicorn", "main:app"]  # Layer 6 : commande de démarrage
```

Au démarrage d'un container, Docker ajoute une **couche d'écriture** (writable layer) par-dessus les layers read-only via **OverlayFS**. Les layers partagées entre images ne sont stockées qu'une seule fois → économie de disque et de temps de téléchargement.

**Point oral** : mettre le `COPY . .` en **dernier** dans le Dockerfile. Si le code change, seules les layers suivantes sont reconstruites. Mettre les dépendances avant le code exploite le **cache des layers**.

---

### Commandes essentielles

```bash
docker build -t mon-app:1.0 .        # construire une image
docker run -p 8000:8000 mon-app:1.0  # lancer un container
docker ps                             # containers en cours
docker logs <container_id>            # logs du container
docker exec -it <id> /bin/bash        # shell interactif
docker-compose up -d                  # orchestration multi-services
```

---

## 33. CI/CD — Intégration et Déploiement Continus

**Définition** :
- **CI (Continuous Integration)** : chaque push déclenche automatiquement les tests et la validation du code — on détecte les régressions immédiatement, pas lors du merge en production.
- **CD (Continuous Delivery / Deployment)** : après validation, le code est automatiquement packagé et déployé en staging (Delivery) ou directement en production (Deployment).

**Objectif** : **éliminer les erreurs humaines** dans le processus de livraison et réduire le cycle entre l'écriture du code et sa mise en production.

---

### Pipeline typique (GitHub Actions / GitLab CI)

```
Push / Pull Request
        │
        ▼
[Lint & Format]           ← flake8, black, isort
        │
        ▼
[Tests unitaires]         ← pytest unit tests
        │
        ▼
[Tests d'intégration]     ← pytest avec DB de test
        │
        ▼
[Build de l'image Docker] ← docker build + push registry
        │
        ▼
[Déploiement Staging]     ← deploy automatique
        │
        ▼
[Tests E2E / Smoke tests] ← validation en environnement réel
        │
        ▼
[Déploiement Production]  ← manuel (Delivery) ou auto (Deployment)
```

**Exemple GitHub Actions (extrait)** :

```yaml
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with: { python-version: "3.11" }
      - run: pip install -r requirements.txt
      - run: pytest --cov=app tests/
```

**Stratégies de déploiement** :

| Stratégie | Principe | Risque |
|---|---|---|
| **Rolling** | Remplacement progressif des instances | Faible — rollback possible |
| **Blue/Green** | Deux environnements identiques, bascule instantanée | Nul — retour immédiat sur l'ancien |
| **Canary** | 5-10% du trafic vers la nouvelle version, puis augmentation | Très faible — exposition limitée |

---

## 34. Git — Gestion de Versions

**Principe** : Git est un système de contrôle de version **distribué**. Chaque clone est un dépôt complet avec tout l'historique. Les commits forment un **DAG (Directed Acyclic Graph)** d'objets immuables identifiés par leur hash SHA-1.

---

### Objets internes Git

| Objet | Contenu | Rôle |
|---|---|---|
| **blob** | Contenu brut d'un fichier | Stockage des données |
| **tree** | Liste de blobs + sous-trees | Représente un répertoire |
| **commit** | Pointeur vers un tree + métadonnées + parent(s) | Snapshot de l'état du projet |
| **tag** | Pointeur nommé vers un commit | Marquage de versions |

**Immuabilité** : chaque objet est identifié par le SHA-1 de son contenu. Modifier un commit change son hash → les commits ne sont jamais modifiés, ils sont remplacés (c'est pourquoi `rebase` réécrit l'historique).

---

### Merge vs Rebase

**Merge** : crée un **commit de fusion** qui unit deux branches. L'historique est fidèle à la réalité — on voit les branches et leur point de convergence.

```
main:    A──B──────M
              \   /
feature:       C──D
```

**Rebase** : réécrit les commits de la branche en les **rejouant** sur la pointe de la branche cible. L'historique est linéaire — comme si le développement avait eu lieu après le dernier commit de main.

```
main:    A──B──C'──D'   (C et D réécrits sur B)
```

**Règle d'or du rebase** : **ne jamais rebaser une branche publique/partagée**. Réécrire des commits déjà poussés force les autres développeurs à réconcilier leurs historiques divergents.

| | **Merge** | **Rebase** |
|---|---|---|
| Historique | Fidèle, non-linéaire | Propre, linéaire |
| Commits | Préservés | Réécrits (nouveaux SHA) |
| Branche publique | Sûr | Dangereux |
| Usage typique | Merge de feature → main | Mettre à jour une branche locale |

---

### Commandes essentielles

```bash
git log --oneline --graph          # visualiser le DAG
git stash / git stash pop          # sauvegarder des changements temporaires
git cherry-pick <sha>              # appliquer un commit isolé
git reset --soft HEAD~1            # annuler le dernier commit, garder les changements
git bisect start/good/bad          # trouver le commit qui a introduit un bug
```

---

## 35. Monitoring — Surveiller la prod

**Principe** : un système en production génère en permanence trois types de données observables — les **3 piliers de l'observabilité** :

| Pilier | Définition | Outils |
|---|---|---|
| **Logs** | Enregistrements textuels des événements | ELK Stack, Loki, CloudWatch |
| **Métriques** | Mesures numériques agrégées dans le temps | Prometheus + Grafana |
| **Traces** | Suivi d'une requête à travers tous les services | Jaeger, OpenTelemetry |

---

### Métriques clés à monitorer

**Les 4 Golden Signals** (Google SRE Book) :

| Signal | Définition | Exemple d'alerte |
|---|---|---|
| **Latency** | Temps de réponse des requêtes | p99 > 500ms |
| **Traffic** | Volume de requêtes (req/s) | Spike anormal détecté |
| **Errors** | Taux d'erreurs (4xx, 5xx) | Taux 5xx > 1% |
| **Saturation** | Utilisation des ressources (CPU, RAM, disk) | CPU > 80% sustained |

**Point oral** : monitorer le **p99** (99ème percentile de latence) plutôt que la moyenne. La moyenne masque les pics — un p99 à 2s signifie que 1 % des utilisateurs attendent 2 secondes, ce qui peut représenter des milliers d'utilisateurs à grande échelle.

---

### Stack Prometheus + Grafana

```
Application
    │ expose /metrics (format texte)
    ▼
Prometheus ── scrape toutes les X secondes ──▶ stockage time-series
    │
    ▼
Grafana ── requêtes PromQL ──▶ dashboards + alertes
```

**MLOps spécifique** : au-delà des métriques système, on monitore les **dérives de modèle** (data drift, concept drift) — si la distribution des données en entrée s'écarte significativement de celle de l'entraînement → alerte pour réentraînement.

---

## 36. AWS Fundamentals

**Principe** : AWS (Amazon Web Services) est le cloud provider dominant. Les services sont regroupés par catégorie — compute, storage, database, messaging.

---

### Compute

| Service | Rôle | Cas d'usage |
|---|---|---|
| **EC2** | Machine virtuelle dans le cloud | Serveur applicatif, training ML |
| **Lambda** | Fonction serverless — exécutée à la demande | Traitement d'événements, API légères |
| **ECS / EKS** | Orchestration de containers (Docker/Kubernetes managé) | Microservices en production |

**Serverless (Lambda) — Trade-offs** :
- **Avantages** : pas de serveur à gérer, facturation à l'exécution, scalabilité automatique.
- **Limites** : **cold start** (latence au premier appel après inactivité), durée max 15 min, stateless obligatoire.

---

### Storage

| Service | Type | Cas d'usage |
|---|---|---|
| **S3** | Object storage (clé → objet) | Fichiers, datasets ML, artefacts de modèles, backups |
| **EBS** | Block storage attaché à une EC2 | Disque dur d'une VM |
| **EFS** | File system partagé entre EC2 | Stockage partagé multi-instances |

**S3 — Points clés** : durabilité 99.999999999% (11 nines), accès via URL ou SDK, versioning, lifecycle policies (archivage automatique vers Glacier).

---

### Databases

| Service | Type | Cas d'usage |
|---|---|---|
| **RDS** | SQL managé (PostgreSQL, MySQL, Aurora) | Base relationnelle production |
| **DynamoDB** | NoSQL clé-valeur / document | Haute scalabilité, faible latence, accès par clé |
| **ElastiCache** | Cache in-memory (Redis, Memcached) | Sessions, résultats d'API, feature store ML |

---

### Messaging & Events

| Service | Modèle | Cas d'usage |
|---|---|---|
| **SQS** | Queue (point-à-point) | Découplage de workers, traitement asynchrone |
| **SNS** | Pub/Sub (fan-out) | Notifications multi-abonnés (email, SMS, Lambda) |
| **EventBridge** | Event bus | Orchestration d'événements inter-services |

**Point oral — SQS vs SNS** : SQS est une file d'attente — un seul consommateur traite chaque message. SNS est un bus de publication — un message est livré à **tous** les abonnés simultanément. En pratique, on combine les deux : SNS → fan-out vers plusieurs SQS → chaque queue traitée par son propre worker.

---

# SECTION VII — Fondamentaux École

---

## 37. Algorithmique & Structures de Données

### Complexité — Big O

**Définition formelle** : `f(n) = O(g(n))` si `∃ c, n₀ > 0` tels que `∀ n > n₀, f(n) ≤ c·g(n)`. Big O borne la croissance dans le **pire cas**.

| Complexité | Nom | Exemple |
|---|---|---|
| O(1) | Constante | Accès dict par clé, accès tableau par index |
| O(log n) | Logarithmique | Recherche binaire, B-Tree |
| O(n) | Linéaire | Parcours de liste, recherche séquentielle |
| O(n log n) | Linéarithmique | Tri optimal (Tim Sort, Merge Sort) — borne inférieure Ω(n log n) pour les tris par comparaison |
| O(n²) | Quadratique | Bubble Sort, boucle imbriquée naïve |
| O(2ⁿ) | Exponentielle | Algorithmes récursifs naïfs (Fibonacci sans mémo) |

**Complexité amortie** : coût moyen sur une séquence d'opérations. Exemple : `list.append()` en Python est O(1) amorti — la copie lors du redimensionnement est O(n), mais elle survient si rarement que le coût moyen reste O(1).

---

### Structures de Données Fondamentales

**Pile (Stack) — LIFO**
- Opérations : `push` (empiler), `pop` (dépiler), `peek` — toutes O(1).
- Implémentation : `list` Python ou `collections.deque`.
- Cas d'usage : pile d'appels (call stack), évaluation d'expressions, undo/redo, DFS itératif.

**File (Queue) — FIFO**
- Opérations : `enqueue`, `dequeue` — O(1) avec `deque` (pas avec `list` → O(n) en tête).
- Cas d'usage : BFS, traitement de jobs, buffers réseau.

**Arbre Binaire de Recherche (BST)**
- Propriété : nœud gauche < racine < nœud droit.
- Recherche/Insertion/Suppression : O(log n) moyen, **O(n) pire cas** (arbre dégénéré = liste chaînée).
- Arbres équilibrés (AVL, Red-Black) : garantissent O(log n) dans tous les cas.

**Heap (Tas)**
- Min-Heap : le plus petit élément est toujours à la racine.
- `heapq` en Python. Insertion : O(log n). Extraction du min : O(log n).
- Cas d'usage : Dijkstra, file de priorité, algorithme de Huffman.

**Traversées d'arbres** :

```python
# In-order (gauche → racine → droite) → tri croissant sur BST
def inorder(node):
    if node: inorder(node.left); visit(node); inorder(node.right)

# Pre-order (racine → gauche → droite) → copie d'arbre, sérialisation
def preorder(node):
    if node: visit(node); preorder(node.left); preorder(node.right)

# Post-order (gauche → droite → racine) → suppression, calcul de taille
def postorder(node):
    if node: postorder(node.left); postorder(node.right); visit(node)
```

**Graphes**
- Représentation : **liste d'adjacence** (O(V+E) mémoire) ou **matrice d'adjacence** (O(V²)).
- **BFS** (Breadth-First Search) : file → plus court chemin en nombre d'arêtes, O(V+E).
- **DFS** (Depth-First Search) : pile/récursion → détection de cycles, tri topologique, O(V+E).
- **Dijkstra** : plus court chemin pondéré (poids positifs), heap → O((V+E) log V).
- **Bellman-Ford** : poids négatifs tolérés, détecte les cycles négatifs → O(V·E).

**Hash Table**
- Accès/Insertion/Suppression : O(1) moyen, O(n) pire cas (collisions).
- Résolution des collisions : chaining (liste chaînée par bucket) ou open addressing.
- En Python : `dict` et `set` sont des hash tables — `in` est O(1), pas O(n) comme pour `list`.

---

### Paradigmes Algorithmiques

| Paradigme | Principe | Exemple |
|---|---|---|
| **Divide & Conquer** | Diviser en sous-problèmes indépendants, combiner | Merge Sort, Quick Sort, Binary Search |
| **Dynamic Programming** | Mémoïser les sous-problèmes qui se chevauchent | Fibonacci, plus longue sous-séquence commune |
| **Greedy** | Choisir localement l'optimum à chaque étape | Huffman, Dijkstra, algorithme de Kruskal |

**Récursion & Tail Recursion** : une fonction récursive qui s'appelle comme dernière opération (tail call) peut être optimisée en boucle par certains compilateurs → O(1) stack au lieu de O(n). Python n'effectue pas cette optimisation (limite de récursion à 1000 par défaut).

---

## 38. Bases de Données — ACID & Formes Normales

### Propriétés ACID

**Définition** : ensemble de propriétés garantissant la fiabilité des transactions dans un SGBD relationnel, même en cas de panne ou d'accès concurrent.

| Propriété | Définition | Mécanisme |
|---|---|---|
| **Atomicité** | La transaction est tout ou rien — pas d'état intermédiaire visible | Rollback si une opération échoue |
| **Cohérence** | La transaction fait passer la BD d'un état valide à un autre état valide | Contraintes (clés, types, triggers) vérifiées avant commit |
| **Isolation** | Les transactions concurrentes ne s'interfèrent pas | Verrous, MVCC (Multi-Version Concurrency Control) |
| **Durabilité** | Un commit est permanent, même après crash | WAL (Write-Ahead Log) — les changements sont écrits sur disque avant le commit |

**Niveaux d'isolation** (du moins au plus strict) :

| Niveau | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| Read Uncommitted | Possible | Possible | Possible |
| Read Committed | Impossible | Possible | Possible |
| Repeatable Read | Impossible | Impossible | Possible |
| Serializable | Impossible | Impossible | Impossible |

**Point oral** : le WAL (Write-Ahead Log) est le mécanisme central de la durabilité. Avant d'écrire dans les pages de données, PostgreSQL écrit d'abord dans le journal séquentiel. En cas de crash, on peut rejouer le WAL pour retrouver un état cohérent.

---

### Normalisation — Les Formes Normales

**Objectif** : éliminer la **redondance** et les **anomalies de mise à jour** (modifier un fait en un seul endroit).

**1NF — Première Forme Normale**
- Chaque colonne contient des valeurs **atomiques** (pas de listes, pas de groupes répétés).
- Chaque ligne est unique (clé primaire).

**2NF — Deuxième Forme Normale**
- 1NF + chaque attribut non-clé dépend de la **totalité** de la clé primaire (pas d'une partie seulement).
- Concerne les tables avec des **clés composées**.

**3NF — Troisième Forme Normale**
- 2NF + aucun attribut non-clé ne dépend d'un autre attribut non-clé (**pas de dépendance transitive**).
- Exemple : si `code_postal → ville`, séparer `ville` dans sa propre table.

**BCNF — Forme Normale de Boyce-Codd**
- Renforcement de la 3NF : chaque déterminant fonctionnel est une **super-clé**.

**Règle mnémotechnique** : *"The key, the whole key, and nothing but the key."*
- 1NF : the key (chaque ligne est identifiable).
- 2NF : the whole key (dépendance à la clé complète).
- 3NF : nothing but the key (pas de dépendance transitive).

---

### Index — B+-Tree

**Principe** : structure arborescente équilibrée qui accélère les requêtes en évitant le full table scan.

- **B+-Tree** : les données sont stockées uniquement dans les **feuilles**, reliées entre elles par une liste chaînée → range scan efficace (`WHERE age BETWEEN 20 AND 30`).
- Recherche : O(log_b n) où b est l'ordre de l'arbre (typiquement 100-1000 en BDD).
- **Covering Index** : l'index contient toutes les colonnes de la requête → pas d'accès aux pages de données.
- **Leftmost Prefix Rule** : un index composite `(A, B, C)` accélère les requêtes sur `A`, `(A,B)`, `(A,B,C)` — mais pas `(B,C)` seul.

---

## 39. Mathématiques — Statistiques & Probabilités

### Statistiques Descriptives

| Mesure | Définition | Sensibilité aux outliers |
|---|---|---|
| **Moyenne** | `Σxᵢ/n` | Très sensible |
| **Médiane** | Valeur centrale après tri | Robuste |
| **Mode** | Valeur la plus fréquente | — |
| **Variance** | `E[(X - μ)²]` | Très sensible |
| **Écart-type σ** | `√Variance` | Même unité que X |

**Point oral** : toujours regarder la **médiane** en plus de la moyenne sur des données asymétriques (salaires, prix immobiliers) — la moyenne est tirée vers le haut par quelques valeurs extrêmes.

---

### Distributions Fondamentales

**Loi Normale (Gaussienne)** : `X ~ N(μ, σ²)`. Symétrique autour de μ. Règle 68-95-99.7 : 68% des valeurs dans [μ-σ, μ+σ], 95% dans [μ-2σ, μ+2σ], 99.7% dans [μ-3σ, μ+3σ].

**Loi de Bernoulli** : expérience binaire (succès/échec). P(X=1) = p. Fondement de la régression logistique et du Naive Bayes.

**Loi de Poisson** : modélise le **nombre d'événements rares** dans un intervalle de temps : `P(X=k) = λᵏe^{-λ}/k!`. Usage : nombre de requêtes par seconde, nombre de pannes par jour.

---

### Probabilités & Théorème de Bayes

**Probabilité conditionnelle** : `P(A|B) = P(A∩B) / P(B)` — probabilité de A sachant que B est survenu.

**Théorème de Bayes** :

```
P(H|E) = P(E|H) · P(H) / P(E)

  Postérieure = Vraisemblance × Prieure / Evidence
```

- **P(H)** : prior — croyance avant d'observer les données.
- **P(E|H)** : likelihood — probabilité d'observer les données si H est vrai.
- **P(H|E)** : posterior — croyance mise à jour après observation.

**Application ML** : Naive Bayes pour la classification de texte (spam detection). "Naïf" car il suppose l'**indépendance conditionnelle** des features sachant la classe — hypothèse rarement vraie mais qui fonctionne bien en pratique.

---

### Concepts Statistiques Clés pour le ML

**Loi des Grands Nombres** : quand n → ∞, la moyenne empirique converge vers l'espérance théorique. Justifie l'utilisation de grands datasets — plus on a de données, plus l'estimation est précise.

**Théorème Central Limite (TCL)** : la somme de n variables aléatoires indépendantes (quelle que soit leur distribution) converge vers une loi normale quand n → ∞. Justifie pourquoi beaucoup de phénomènes naturels suivent une gaussienne.

**Corrélation vs Causalité** : la corrélation de Pearson mesure l'association linéaire (`r ∈ [-1, 1]`). `r = 1` → parfaitement corrélés linéairement. Une corrélation forte ne prouve pas la causalité — il peut exister une variable confondante.

**p-value & Tests d'hypothèse** :
- **H₀ (hypothèse nulle)** : pas d'effet, pas de différence.
- **p-value** : probabilité d'observer des données aussi extrêmes si H₀ est vraie.
- Si p < 0.05 → on rejette H₀ (résultat "statistiquement significatif").
- **Attention** : significatif statistiquement ≠ significatif pratiquement. Avec assez de données, un effet infime devient significatif.

**Entropie (Shannon)** : `H(X) = -Σ p(x) log₂ p(x)`. Mesure l'**incertitude** d'une distribution. Entropie maximale = distribution uniforme. Entropie nulle = distribution déterministe. Utilisée dans les Decision Trees (gain d'information = réduction d'entropie) et la Cross-Entropy loss.
