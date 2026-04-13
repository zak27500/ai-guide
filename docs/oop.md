# Orienté Objet & Génie Logiciel 

---

## I. Quels sont les 4 piliers de la POO ?

**Question** : "Pouvez-vous expliquer les quatre piliers de la programmation orientée objet et donner un exemple concret pour chacun ?"

**Réponse** :

**Encapsulation** : regrouper les données (attributs) et les comportements (méthodes) dans une même unité, et restreindre l'accès direct aux données internes. On expose uniquement ce qui est nécessaire via une interface publique. En Python, on préfixe les attributs privés avec `_` ou `__`. L'objectif est de protéger l'état interne et de réduire les dépendances entre composants.

```python
class CompteBancaire:
    def __init__(self, solde):
        self.__solde = solde  # privé

    def deposer(self, montant):
        if montant > 0:
            self.__solde += montant

    def get_solde(self):
        return self.__solde
```

**Abstraction** : exposer uniquement les fonctionnalités pertinentes d'un objet, en cachant les détails d'implémentation. L'utilisateur d'une classe sait *quoi* elle fait, pas *comment*. En Python, on utilise les classes abstraites (`ABC`) pour définir des contrats que les sous-classes doivent respecter.

```python
from abc import ABC, abstractmethod

class Vehicule(ABC):
    @abstractmethod
    def demarrer(self):
        pass

class Voiture(Vehicule):
    def demarrer(self):
        return "Moteur thermique démarré"
```

**Héritage** : une classe enfant réutilise et étend le comportement d'une classe parent. Cela évite la duplication de code. En Python, une classe peut hériter de plusieurs parents (héritage multiple) — l'ordre de résolution est défini par la MRO (Method Resolution Order, algorithme C3 Linearization).

```python
class Animal:
    def respirer(self):
        return "Inspire / Expire"

class Chien(Animal):
    def aboyer(self):
        return "Woof"

rex = Chien()
rex.respirer()  # hérité d'Animal
rex.aboyer()    # propre à Chien
```

**Polymorphisme** : un même appel de méthode se comporte différemment selon le type réel de l'objet. Il permet d'écrire du code générique qui fonctionne sur des objets de types variés sans connaître leur classe concrète.

```python
class Chat(Animal):
    def parler(self):
        return "Miaou"

class Chien(Animal):
    def parler(self):
        return "Woof"

animaux = [Chat(), Chien(), Chat()]
for a in animaux:
    print(a.parler())  # même appel, comportement différent
```

---

## II. Qu'est-ce que les principes SOLID ?

**Question** : "Pouvez-vous expliquer les principes SOLID et donner un exemple de violation et de correction pour l'un d'eux ?"

**Réponse** :

SOLID est un acronyme de cinq principes de conception qui rendent le code plus maintenable, extensible et testable.

**S — Single Responsibility Principle (SRP)** : une classe ne doit avoir qu'une seule raison de changer. Si une classe gère à la fois la logique métier et la persistance en base de données, une modification du schéma SQL oblige à retoucher la logique métier — ce sont deux responsabilités distinctes qui doivent être séparées.

**O — Open/Closed Principle (OCP)** : une classe doit être ouverte à l'extension, fermée à la modification. On ajoute des comportements via de nouvelles classes (héritage ou composition), sans toucher au code existant déjà testé.

```python
# Violation : on modifie la classe à chaque nouveau format
class Exporteur:
    def exporter(self, data, format):
        if format == "csv": ...
        elif format == "json": ...  # modifier à chaque nouveau format

# Correction OCP : on étend sans modifier
class ExporteurCSV:
    def exporter(self, data): ...

class ExporteurJSON:
    def exporter(self, data): ...
```

**L — Liskov Substitution Principle (LSP)** : une sous-classe doit pouvoir remplacer sa classe parent sans que le comportement du programme ne change. Violation classique : une classe `Carre` hérite de `Rectangle` mais écrase `set_width` et `set_height` de façon incohérente — un `Carre` ne peut pas remplacer un `Rectangle` sans casser les invariants.

**I — Interface Segregation Principle (ISP)** : il vaut mieux plusieurs interfaces spécifiques qu'une seule interface générale. Un client ne doit pas être forcé d'implémenter des méthodes qu'il n'utilise pas.

**D — Dependency Inversion Principle (DIP)** : les modules de haut niveau ne doivent pas dépendre des modules de bas niveau. Les deux doivent dépendre d'abstractions. En pratique, on injecte les dépendances plutôt que de les instancier directement — ce qui rend le code testable (on peut injecter un mock).

```python
# Violation : dépendance directe sur l'implémentation
class Service:
    def __init__(self):
        self.db = MySQLDatabase()  # couplage fort

# Correction DIP : dépendance sur l'abstraction
class Service:
    def __init__(self, db: DatabaseInterface):  # injection
        self.db = db
```

---

## III. Qu'est-ce que le principe DRY ?

**Question** : "Qu'est-ce que le principe DRY et quels sont les risques de le violer dans une base de code en production ?"

**Réponse** :

**DRY — Don't Repeat Yourself** : chaque morceau de connaissance ou de logique doit avoir une représentation unique et faisant autorité dans le système. Toute duplication de logique — pas seulement de code — est une violation du DRY.

**Les risques de la violation** : si une règle métier est implémentée en trois endroits, une correction de bug ou un changement de règle doit être appliqué dans les trois endroits simultanément. Oublier l'un d'eux crée une incohérence silencieuse qui peut passer les tests et arriver en production.

**La nuance importante** : DRY ne signifie pas "ne jamais écrire deux fois le même code". Deux fonctions qui partagent une implémentation similaire mais servent des concepts métier différents ne doivent *pas* être factorisées — sinon un changement dans l'une affecte l'autre à tort. Le DRY s'applique aux connaissances et règles métier, pas à la forme syntaxique.

```python
# Violation DRY
def calculer_tva_fr(prix):
    return prix * 0.20

def calculer_tva_produit(prix):
    return prix * 0.20  # même logique dupliquée

# Correction
TVA_FR = 0.20

def calculer_tva(prix, taux=TVA_FR):
    return prix * taux
```

---

## IV. Qu'est-ce que le principe KISS ?

**Question** : "Qu'est-ce que le principe KISS et comment s'applique-t-il concrètement lors d'une revue de code ?"

**Réponse** :

**KISS — Keep It Simple, Stupid** : la plupart des systèmes fonctionnent mieux lorsqu'ils sont simples plutôt que complexes. La simplicité doit être un objectif de conception explicite, et la complexité inutile doit être évitée.

**En pratique lors d'une revue de code** : si une fonction de 5 lignes résout le problème, une version de 40 lignes avec des abstractions génériques n'est pas meilleure — elle est plus difficile à lire, à tester et à débugger. On applique KISS en se demandant à chaque décision : "est-ce que cette complexité est justifiée par un besoin réel et actuel ?"

**La tension avec d'autres principes** : KISS peut sembler en contradiction avec SOLID (qui encourage l'abstraction). La règle d'arbitrage : introduire une abstraction uniquement quand le besoin est avéré, pas par anticipation.

```python
# Violation KISS : sur-ingénierie
class SalutationFactory:
    def create_salutation(self, lang):
        return SalutationStrategy.for_lang(lang).generate()

# KISS : direct et lisible
def saluer(nom):
    return f"Bonjour, {nom}"
```

---

## V. Qu'est-ce que le principe YAGNI ?

**Question** : "Qu'est-ce que YAGNI et pourquoi est-il particulièrement important dans un contexte Agile ou de startup ?"

**Réponse** :

**YAGNI — You Aren't Gonna Need It** : n'implémentez pas une fonctionnalité tant qu'elle n'est pas nécessaire. Ne pas anticiper des besoins hypothétiques futurs au détriment de la clarté et de la rapidité de livraison présente.

**Pourquoi c'est critique en Agile** : dans un environnement où les priorités changent à chaque sprint, du code écrit "au cas où" est souvent du code qu'on maintient inutilement pendant des mois, qu'on n'utilise jamais, et qui complexifie les refactorisations futures.

**La manifestation courante** : ajouter des paramètres optionnels "pour être flexible", créer des systèmes de plugins avant d'avoir un second cas d'usage, ou abstraire une logique utilisée une seule fois.

**YAGNI vs DRY** : YAGNI dit "ne construis pas ce dont tu n'as pas besoin maintenant". DRY dit "ne duplique pas ce que tu as déjà". Les deux sont complémentaires : YAGNI évite la complexité accidentelle, DRY évite l'incohérence.

```python
# Violation YAGNI : flexibilité non demandée
def envoyer_email(destinataire, sujet, corps,
                  cc=None, bcc=None, priorite=None,
                  format_html=False, tracking=False):
    ...  # 80% de ces paramètres ne seront jamais utilisés

# YAGNI : on implémente ce qui est demandé
def envoyer_email(destinataire, sujet, corps):
    ...
```

---

## VI. Qu'est-ce que le TDD et pourquoi l'utiliser ?

**Question** : "Expliquez le cycle TDD et en quoi écrire les tests avant le code change fondamentalement la façon de concevoir un système."

**Réponse** :

**TDD — Test-Driven Development** : méthodologie de développement dans laquelle on écrit le test avant d'écrire le code de production. Le cycle se répète en trois étapes courtes.

**Le cycle Red → Green → Refactor** :

1. **Red** : écrire un test qui décrit le comportement attendu. Le test échoue car le code n'existe pas encore — c'est normal et voulu.
2. **Green** : écrire le minimum de code nécessaire pour faire passer le test. Pas plus.
3. **Refactor** : améliorer la structure du code (DRY, KISS, lisibilité) sans en changer le comportement — les tests garantissent qu'on ne casse rien.

```python
# 1. RED — on écrit le test d'abord
def test_calcul_tva():
    assert calculer_tva(100) == 20.0

# 2. GREEN — on implémente le minimum
def calculer_tva(prix):
    return prix * 0.20

# 3. REFACTOR — on nettoie si besoin
TVA = 0.20
def calculer_tva(prix, taux=TVA):
    return prix * taux
```

**Pourquoi ça change la conception** : écrire le test en premier oblige à penser à l'interface publique de la fonction avant son implémentation. Si le test est difficile à écrire, c'est un signal que le design est mauvais (trop couplé, trop de dépendances). Le TDD force naturellement vers des composants découplés et injectables — ce qui converge avec le principe DIP de SOLID.

**Les limites** : le TDD est moins adapté aux phases d'exploration rapide (prototypage, Data Science), aux interfaces graphiques, ou aux tests d'intégration complexes. Il brille sur la logique métier pure et les algorithmes.

---

## VII. Comment articuler SOLID, DRY, KISS et YAGNI ensemble ?

**Question** : "Ces principes semblent parfois contradictoires. Comment les articulez-vous au quotidien dans vos décisions de conception ?"

**Réponse** :

Ces principes ne sont pas des règles absolues — ce sont des outils de raisonnement avec des tensions assumées.

**La hiérarchie pragmatique** :

- **YAGNI prime sur SOLID** au départ : ne pas créer d'abstractions avant d'avoir deux cas d'usage concrets. La règle des trois — on abstrait à la troisième duplication, pas à la première.
- **DRY prime quand la logique est métier** : une règle de calcul dupliquée est dangereuse. Deux boucles similaires sans lien métier n'ont pas à être factorisées.
- **KISS prime sur OCP** quand la fonctionnalité est stable : ne pas créer un système extensible si une seule implémentation est prévue.
- **TDD rend DIP naturel** : tester en isolation force l'injection de dépendances.

**La question à se poser face à chaque décision** : "Est-ce que cette complexité résout un problème réel aujourd'hui, ou est-ce que j'anticipe un futur incertain ?" Si la réponse est "j'anticipe" — appliquer YAGNI et KISS. Si une règle métier commence à diverger — appliquer DRY. Si une nouvelle fonctionnalité oblige à modifier du code existant testé — appliquer OCP.
