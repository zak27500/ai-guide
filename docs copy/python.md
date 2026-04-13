# Python 

---

## I. Qu'est-ce que le GIL et quel est son impact en production ?

**Question** : "Python possède un mécanisme appelé le GIL. Qu'est-ce que c'est, pourquoi existe-t-il, et quel est son impact concret sur les performances d'une application multi-threadée ?"

**Réponse** :

Le **GIL (Global Interpreter Lock)** est un verrou global dans CPython qui garantit qu'un seul thread exécute du bytecode Python à la fois, même sur une machine multi-cœurs.

**Pourquoi il existe** : CPython gère la mémoire via un compteur de références. Sans le GIL, deux threads pourraient modifier ce compteur simultanément, corrompre l'état interne et provoquer des crashes ou des fuites mémoire. Le GIL est une solution pragmatique à ce problème de thread-safety.

**Impact concret** :
- **CPU-bound tasks** (calcul numérique, compression, chiffrement) : le multithreading n'apporte aucun gain — les threads se disputent le GIL et s'exécutent séquentiellement. Il faut utiliser `multiprocessing` pour exploiter les cœurs.
- **I/O-bound tasks** (requêtes HTTP, lecture fichier, base de données) : le GIL est relâché pendant les appels I/O bloquants. Le multithreading est donc efficace pour ce type de tâches.

**Contournements** : `multiprocessing` (plusieurs processus, chacun avec son GIL), `asyncio` (concurrence coopérative sans threads), ou des bibliothèques C/Cython qui relâchent le GIL manuellement (NumPy, Pandas).

```python
# CPU-bound : utiliser multiprocessing, pas threading
from multiprocessing import Pool

def calculer(n):
    return sum(i * i for i in range(n))

with Pool(4) as p:
    resultats = p.map(calculer, [10**6] * 4)
```

---

## II. Comment Python gère-t-il la mémoire ?

**Question** : "Expliquez comment Python alloue et libère la mémoire. Qu'est-ce que le comptage de références et quelles sont ses limites ?"

**Réponse** :

Python gère la mémoire via deux mécanismes complémentaires.

**Comptage de références (Reference Counting)** : chaque objet en mémoire possède un compteur `ob_refcnt`. Quand un objet est assigné à une variable ou passé en argument, son compteur s'incrémente. Quand une variable sort de scope ou est réassignée, le compteur décrémente. Quand le compteur atteint 0, l'objet est immédiatement désalloué.

```python
import sys
a = [1, 2, 3]
print(sys.getrefcount(a))  # 2 (a + argument de getrefcount)
b = a
print(sys.getrefcount(a))  # 3
del b
print(sys.getrefcount(a))  # 2
```

**Limite : les références circulaires** : si deux objets se référencent mutuellement, leurs compteurs ne descendent jamais à 0 même quand aucune variable externe ne les pointe — fuite mémoire garantie sans intervention.

```python
a = {}
b = {}
a["ref"] = b
b["ref"] = a  # cycle : a et b ne seront jamais libérés par refcount seul
```

**Le Garbage Collector (GC)** résout ce problème — voir question suivante.

**L'allocateur Python** : Python utilise un allocateur de mémoire en couches — `pymalloc` pour les petits objets (< 512 octets) via un système de pools et d'arènes, pour éviter la fragmentation et accélérer les allocations fréquentes.

---

## III. Comment fonctionne le Garbage Collector Python ?

**Question** : "Le comptage de références ne suffit pas à gérer les cycles. Comment le Garbage Collector de Python détecte-t-il et libère-t-il les objets cycliques ?"

**Réponse** :

Le **GC Python** est un collecteur générationnel qui complète le comptage de références en détectant les références circulaires.

**Le principe générationnel** : les objets sont répartis en trois générations (0, 1, 2) selon leur ancienneté.
- **Génération 0** : objets nouvellement créés. Collectée le plus fréquemment.
- **Génération 1** : objets ayant survécu à une collecte de la génération 0.
- **Génération 2** : objets de longue durée (caches, singletons). Collectée rarement.

**Pourquoi ce modèle** : l'hypothèse générationnelle — la plupart des objets meurent jeunes. Collecter fréquemment les jeunes objets est peu coûteux et capture la majorité des cycles.

**Déclenchement** : la collecte se déclenche automatiquement quand le nombre d'allocations minus désallocations dépasse un seuil (700 pour la génération 0 par défaut).

```python
import gc

gc.collect()          # forcer une collecte manuelle
gc.get_threshold()    # (700, 10, 10) -- seuils par génération
gc.disable()          # désactiver si on gère tout par refcount (perf critique)

# Détecter les cycles résiduels
gc.set_debug(gc.DEBUG_LEAK)
```

**Bonnes pratiques** : éviter les cycles inutiles dans les classes Python, utiliser `weakref` pour les références qui ne doivent pas empêcher la désallocation, et désactiver le GC dans des contextes haute performance où les cycles sont absents.

---

## IV. Quelle différence entre Threading, Multiprocessing et Asyncio ?

**Question** : "Vous avez besoin d'exécuter des tâches en parallèle en Python. Quand choisissez-vous `threading`, `multiprocessing` ou `asyncio` ?"

**Réponse** :

Les trois modèles répondent à des problèmes différents.

**Threading** : plusieurs threads dans le même processus, même espace mémoire. Efficace pour les tâches **I/O-bound** (requêtes HTTP, fichiers) car le GIL est relâché lors des appels I/O bloquants. Mémoire partagée = moins de surcharge, mais risque de race conditions si les threads modifient des données communes.

```python
from concurrent.futures import ThreadPoolExecutor
import requests

urls = ["https://api.example.com/1", "https://api.example.com/2"]

with ThreadPoolExecutor(max_workers=10) as executor:
    resultats = list(executor.map(requests.get, urls))
```

**Multiprocessing** : plusieurs processus indépendants, chacun avec son propre interpréteur Python et son propre GIL. Seul moyen d'exploiter plusieurs cœurs pour des tâches **CPU-bound**. Surcharge plus élevée (mémoire dupliquée, IPC pour communiquer entre processus).

```python
from concurrent.futures import ProcessPoolExecutor

def tache_cpu(n):
    return sum(i**2 for i in range(n))

with ProcessPoolExecutor(max_workers=4) as executor:
    resultats = list(executor.map(tache_cpu, [10**7] * 4))
```

**Asyncio** : concurrence coopérative dans un seul thread via une boucle d'événements. Pas de GIL à gérer, pas de race conditions sur les données. Idéal pour des milliers de tâches **I/O-bound** simultanées (serveur web, websockets, scraping massif). Le code doit être explicitement async/await — une coroutine qui bloque (`time.sleep` au lieu de `asyncio.sleep`) bloque toute la boucle.

```python
import asyncio
import aiohttp

async def fetch(session, url):
    async with session.get(url) as response:
        return await response.json()

async def main():
    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, f"https://api.example.com/{i}") for i in range(100)]
        resultats = await asyncio.gather(*tasks)
```

**Règle de décision** :
| Problème | Solution |
|---|---|
| Beaucoup de requêtes réseau / I/O | `asyncio` |
| I/O avec bibliothèques non-async | `threading` |
| Calcul intensif (CPU) | `multiprocessing` |
| Calcul intensif + Python pur | `multiprocessing` + `numpy` |

---

## V. Comment fonctionnent les WebSockets en Python ?

**Question** : "Quelle est la différence entre une requête HTTP classique et une connexion WebSocket ? Dans quel cas l'utiliser et comment l'implémenter côté serveur ?"

**Réponse** :

**HTTP** est un protocole requête-réponse stateless : le client envoie une requête, le serveur répond, la connexion se ferme. Pour obtenir des mises à jour, le client doit interroger le serveur régulièrement (polling) — coûteux et peu réactif.

**WebSocket** établit une connexion TCP persistante et bidirectionnelle après un handshake HTTP initial (Upgrade). Une fois ouverte, le serveur peut pousser des données au client à tout moment sans que le client ne demande.

**Cas d'usage** : chat en temps réel, notifications live, tableaux de bord de monitoring, streaming de tokens LLM, jeux multijoueurs.

**Implémentation avec FastAPI** :

```python
from fastapi import FastAPI, WebSocket
from fastapi.websockets import WebSocketDisconnect

app = FastAPI()

connected_clients = []

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    connected_clients.append(websocket)
    try:
        while True:
            data = await websocket.receive_text()
            # broadcast à tous les clients connectés
            for client in connected_clients:
                await client.send_text(f"Message reçu : {data}")
    except WebSocketDisconnect:
        connected_clients.remove(websocket)
```

**WebSocket vs SSE (Server-Sent Events)** : SSE est unidirectionnel (serveur → client uniquement), plus simple à implémenter, suffisant pour le streaming de tokens LLM. WebSocket est bidirectionnel, nécessaire pour des interactions en temps réel.

---

## VI. Qu'est-ce qu'un décorateur Python ?

**Question** : "Expliquez ce qu'est un décorateur Python, comment il fonctionne mécaniquement, et donnez un exemple d'usage en production."

**Réponse** :

Un **décorateur** est une fonction qui prend une autre fonction en argument, l'enveloppe dans une nouvelle fonction, et retourne cette nouvelle fonction. C'est du sucre syntaxique pour la composition de fonctions.

**Mécanisme** :

```python
def mon_decorateur(func):
    def wrapper(*args, **kwargs):
        print("Avant l'appel")
        resultat = func(*args, **kwargs)  # appel de la fonction originale
        print("Après l'appel")
        return resultat
    return wrapper

@mon_decorateur
def saluer(nom):
    return f"Bonjour {nom}"

# Equivalent exact à : saluer = mon_decorateur(saluer)
```

**Usage production — décorateur de retry** :

```python
import time
import functools

def retry(max_attempts=3, delay=1.0):
    def decorator(func):
        @functools.wraps(func)  # préserve __name__, __doc__
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts - 1:
                        raise
                    time.sleep(delay * (2 ** attempt))  # backoff exponentiel
        return wrapper
    return decorator

@retry(max_attempts=3, delay=0.5)
def appel_api(url):
    return requests.get(url)
```

**Points clés** : toujours utiliser `@functools.wraps(func)` pour que le wrapper conserve les métadonnées de la fonction originale (nom, docstring, signature) — sinon les outils d'introspection et les debuggers voient `wrapper` au lieu du nom réel.

---

## VII. Qu'est-ce qu'un générateur Python ?

**Question** : "Quelle est la différence entre une fonction classique et un générateur ? Dans quel cas les générateurs sont-ils indispensables en production ?"

**Réponse** :

Un **générateur** est une fonction qui utilise `yield` au lieu de `return`. Au lieu de calculer et retourner toutes les valeurs d'un coup, il les produit une par une à la demande — c'est de l'**évaluation paresseuse (lazy evaluation)**.

```python
# Fonction classique : charge tout en mémoire
def lire_fichier_classique(chemin):
    with open(chemin) as f:
        return f.readlines()  # tout en RAM

# Générateur : une ligne à la fois, mémoire constante
def lire_fichier_lazy(chemin):
    with open(chemin) as f:
        for ligne in f:
            yield ligne.strip()

# Usage identique, mais le générateur ne charge jamais tout le fichier
for ligne in lire_fichier_lazy("gros_fichier.csv"):
    traiter(ligne)
```

**Pourquoi c'est indispensable** : un fichier CSV de 10 Go chargé avec `readlines()` consomme 10 Go de RAM. Le générateur consomme quelques Ko quelle que soit la taille du fichier.

**Pipelines de traitement** :

```python
def lire(chemin):
    with open(chemin) as f:
        yield from f

def filtrer(lignes, mot_cle):
    for ligne in lignes:
        if mot_cle in ligne:
            yield ligne

def transformer(lignes):
    for ligne in lignes:
        yield ligne.upper()

# Pipeline composé : aucune étape ne matérialise l'ensemble des données
pipeline = transformer(filtrer(lire("data.csv"), "error"))
```

**`yield from`** : délègue la génération à un autre itérable, utile pour composer des générateurs ou aplatir des structures récursives.

---

## VIII. Modules, packages et bibliothèques — quelle différence ?

**Question** : "Quelle est la différence entre un module, un package et une bibliothèque en Python ? Comment Python résout-il les imports ?"

**Réponse** :

**Module** : un fichier `.py` unique. Tout fichier Python est un module importable.

```python
# utils.py  --> module "utils"
def helper(): ...

# Dans un autre fichier
import utils
from utils import helper
```

**Package** : un répertoire contenant un fichier `__init__.py` (Python 3.3+ supporte aussi les namespace packages sans `__init__.py`). Un package peut contenir des modules et d'autres packages.

```
mon_package/
    __init__.py       # exécuté à l'import du package
    module_a.py
    sous_package/
        __init__.py
        module_b.py
```

**Bibliothèque** : terme informel désignant un ensemble de packages/modules distribués (via PyPI avec `pip install`).

**Résolution des imports** : Python cherche dans cet ordre (`sys.path`) :
1. Le répertoire courant du script
2. Les répertoires de `PYTHONPATH`
3. Les bibliothèques standard
4. Les packages installés (`site-packages`)

```python
import sys
print(sys.path)  # voir l'ordre de résolution complet
```

**`__init__.py`** : permet de définir l'API publique du package via `__all__`, d'exécuter du code d'initialisation, et de raccourcir les imports.

```python
# mon_package/__init__.py
from .module_a import ClassePrincipale  # raccourci l'import

__all__ = ["ClassePrincipale"]  # contrôle ce que "from package import *" expose
```

---

## IX. Quelle différence entre `@classmethod`, `@staticmethod` et `@property` ?

**Question** : "Expliquez la différence entre `@classmethod`, `@staticmethod` et `@property`. Dans quel cas concret utilisez-vous chacun ?"

**Réponse** :

**`@staticmethod`** : méthode qui n'a accès ni à l'instance (`self`) ni à la classe (`cls`). C'est une fonction ordinaire logée dans le namespace de la classe pour des raisons d'organisation. Elle ne peut ni lire ni modifier l'état de la classe ou de l'instance.

```python
class Calculateur:
    @staticmethod
    def valider_montant(montant):
        return isinstance(montant, (int, float)) and montant > 0
```

**`@classmethod`** : méthode qui reçoit la classe elle-même (`cls`) comme premier argument, pas l'instance. Utilisée pour des constructeurs alternatifs ou des méthodes qui opèrent sur la classe entière.

```python
class Date:
    def __init__(self, jour, mois, annee):
        self.jour, self.mois, self.annee = jour, mois, annee

    @classmethod
    def from_string(cls, date_str):
        jour, mois, annee = map(int, date_str.split("-"))
        return cls(jour, mois, annee)  # cls = Date, fonctionne aussi pour les sous-classes

d = Date.from_string("25-12-2024")
```

**`@property`** : transforme une méthode en attribut calculé. Permet d'accéder à une valeur dérivée comme si c'était un attribut, avec contrôle optionnel de l'écriture via `@attr.setter`.

```python
class Cercle:
    def __init__(self, rayon):
        self._rayon = rayon

    @property
    def rayon(self):
        return self._rayon

    @rayon.setter
    def rayon(self, valeur):
        if valeur < 0:
            raise ValueError("Le rayon doit être positif")
        self._rayon = valeur

    @property
    def aire(self):
        return 3.14159 * self._rayon ** 2  # calculé à la volée

c = Cercle(5)
print(c.aire)   # appel sans parenthèses
c.rayon = -1    # lève ValueError
```

---

## X. Comment fonctionne FastAPI et pourquoi l'utiliser ?

**Question** : "Qu'est-ce que FastAPI, quels sont ses avantages par rapport à Flask ou Django REST Framework, et comment structure-t-on une API avec validation des données ?"

**Réponse** :

**FastAPI** est un framework web Python asynchrone, construit sur Starlette (ASGI) et Pydantic. Il est conçu pour construire des APIs REST performantes avec un minimum de boilerplate.

**Avantages clés** :
- **Validation automatique** : Pydantic valide les données entrantes et sortantes en se basant sur les type hints Python — zéro code de validation manuelle.
- **Documentation auto-générée** : FastAPI génère Swagger UI (`/docs`) et ReDoc (`/redoc`) automatiquement à partir des signatures de fonctions.
- **Performances** : asynchrone natif via asyncio, performances comparables à Node.js et Go pour les workloads I/O-bound.
- **Type safety** : les type hints servent à la fois de documentation, de validation, et d'autocomplétion IDE.

```python
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel, Field
from typing import Optional

app = FastAPI(title="Mon API", version="1.0.0")

# Schéma de données avec validation Pydantic
class ArticleCreate(BaseModel):
    titre: str = Field(..., min_length=3, max_length=200)
    contenu: str = Field(..., min_length=10)
    tags: list[str] = []
    prix: Optional[float] = Field(None, gt=0)

class ArticleResponse(BaseModel):
    id: int
    titre: str
    contenu: str

    class Config:
        from_attributes = True  # compatibilité ORM

# Route avec validation automatique
@app.post("/articles", response_model=ArticleResponse, status_code=201)
async def creer_article(article: ArticleCreate):
    # article est déjà validé et typé à cette ligne
    nouvel_article = await db.save(article)
    return nouvel_article

@app.get("/articles/{article_id}", response_model=ArticleResponse)
async def lire_article(article_id: int):
    article = await db.get(article_id)
    if not article:
        raise HTTPException(status_code=404, detail="Article introuvable")
    return article
```

**Dépendances (Depends)** : mécanisme d'injection de dépendances pour factoriser l'authentification, la connexion DB, ou les paramètres communs.

```python
from fastapi.security import HTTPBearer

security = HTTPBearer()

async def verifier_token(token = Depends(security)):
    if not valider_jwt(token.credentials):
        raise HTTPException(status_code=401, detail="Token invalide")
    return token

@app.get("/profil", dependencies=[Depends(verifier_token)])
async def profil():
    return {"message": "Accès autorisé"}
```

---

## XI. Qu'est-ce qu'une REST API et comment concevoir ses ressources ?

**Question** : "Quels sont les principes d'une API REST ? Comment nommez-vous les endpoints et gérez-vous les codes de statut HTTP ?"

**Réponse** :

**REST (Representational State Transfer)** est un style architectural pour les APIs web basé sur six contraintes, dont les plus importantes en pratique sont le **stateless** (chaque requête contient toutes les informations nécessaires, pas de session côté serveur) et l'**interface uniforme** (ressources identifiées par des URIs, manipulation via les verbes HTTP).

**Les verbes HTTP et leur sémantique** :
| Verbe | Action | Idempotent | Body |
|---|---|---|---|
| GET | Lire | Oui | Non |
| POST | Créer | Non | Oui |
| PUT | Remplacer entièrement | Oui | Oui |
| PATCH | Modifier partiellement | Non | Oui |
| DELETE | Supprimer | Oui | Non |

**Nommage des ressources** : les URIs désignent des **noms** (ressources), jamais des **verbes** (actions). On utilise le pluriel pour les collections.

```
# Correct
GET    /articles          -- liste tous les articles
POST   /articles          -- crée un article
GET    /articles/42       -- lit l'article 42
PUT    /articles/42       -- remplace l'article 42
PATCH  /articles/42       -- modifie partiellement l'article 42
DELETE /articles/42       -- supprime l'article 42
GET    /articles/42/tags  -- relation imbriquée

# Incorrect (verbes dans l'URI)
POST   /creer-article
GET    /getArticle?id=42
POST   /articles/42/delete
```

**Codes de statut HTTP à maîtriser** :
| Code | Signification | Usage |
|---|---|---|
| 200 | OK | GET, PUT, PATCH réussis |
| 201 | Created | POST réussi (+ header Location) |
| 204 | No Content | DELETE réussi |
| 400 | Bad Request | Données invalides |
| 401 | Unauthorized | Non authentifié |
| 403 | Forbidden | Authentifié mais non autorisé |
| 404 | Not Found | Ressource introuvable |
| 409 | Conflict | Doublon, état incohérent |
| 422 | Unprocessable Entity | Validation échouée (FastAPI) |
| 429 | Too Many Requests | Rate limiting |
| 500 | Internal Server Error | Erreur serveur inattendue |

**Versioning** : préfixer les routes avec la version (`/v1/articles`) pour permettre l'évolution de l'API sans casser les clients existants.
