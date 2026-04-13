# Entretien Final — Athoria AI Engineer

*Ciblé sur : Python · LangChain/LangGraph · AWS · GenAI Transversal · Posture Consultante*

---

# PARTIE 1 — Python

---

## GIL — Global Interpreter Lock

**Définition** : mutex CPython qui garantit qu'**un seul thread exécute du bytecode à la fois**, quel que soit le nombre de cœurs.

**Pourquoi** : CPython gère la mémoire par comptage de références (`ob_refcnt`). Sans verrou → deux threads modifient le même compteur simultanément → race condition → corruption mémoire ou libération prématurée. Verrouiller l'interpréteur entier est plus simple et moins coûteux que de verrouiller chaque objet.

**Impact** : annule le bénéfice du multi-cœur pour les tâches **CPU-bound**. Le GIL est libéré automatiquement pendant les I/O bloquantes (réseau, disque) — les C-extensions sous-jacentes n'ont pas besoin du bytecode Python.

**Commutation** : CPython relâche le GIL toutes les **5 ms** (`sys.getswitchinterval()`) pour permettre à un autre thread de l'acquérir.

**Question piège** : "Comment paralléliser du calcul en Python ?" → `multiprocessing` (chaque processus a son propre interpréteur et son propre GIL), pas `threading`.

**À mentionner si poussé** : Python 3.13 introduit un mode **free-threaded** (PEP 703 — nogil expérimental) qui supprime le GIL et permet un vrai parallélisme multi-cœur. Encore instable en production.

---

## Mémoire & Garbage Collector

**Comptage de références** : `ob_refcnt` incrémenté à chaque nouvelle référence, décrémenté à chaque suppression. Libération **immédiate** quand `ob_refcnt == 0`.

**Limite** : les cycles (A → B → A) maintiennent les compteurs à des valeurs non nulles même sans référence externe. Le comptage de références est aveugle aux cycles.

**GC générationnel (solution)** : détecte les cycles en simulant la suppression des références internes au groupe. Tout objet dont le compteur simulé tombe à 0 est inatteignable → collecté.

| Génération | Contenu | Seuil de déclenchement |
|---|---|---|
| Gen 0 | Nouveaux objets | allocations − désallocations > 700 |
| Gen 1 | Survivants Gen 0 | après 10 collectes Gen 0 |
| Gen 2 | Survivants Gen 1 | après 10 collectes Gen 1 |

**Point prod** : `gc.disable()` sur des services critiques en latence (les objets sans cycles sont libérés immédiatement de toute façon par le comptage de références).

---

## Threading vs Multiprocessing vs Asyncio

| Scénario | Solution | Raison |
|---|---|---|
| I/O-bound (API, BDD, LLM calls) | `asyncio` ou `threading` | GIL relâché pendant l'attente I/O |
| CPU-bound (inférence, preprocessing) | `multiprocessing` | Contourne le GIL via processus séparés |
| Milliers de connexions simultanées | `asyncio` | Un seul thread, zéro context switch OS |
| Libs bloquantes non-async (ex: `requests`) | `threading` | Pas d'alternative asyncio |

**Asyncio en détail** : modèle **coopératif** — le développeur cède le contrôle via `await`. L'event loop (basée sur `epoll` sous Linux) surveille les descripteurs de fichiers et reprend les coroutines dès que l'I/O est prête.

```python
import asyncio
import httpx

async def fetch(url: str) -> str:
    async with httpx.AsyncClient() as client:
        response = await client.get(url)   # cède le contrôle pendant l'attente
        return response.text

async def main():
    # 3 requêtes en parallèle — un seul thread
    results = await asyncio.gather(
        fetch("https://api-1.com"),
        fetch("https://api-2.com"),
        fetch("https://api-3.com"),
    )

asyncio.run(main())
```

**Cas concret AI Engineer** : un serveur FastAPI avec plusieurs appels LLM simultanés utilise asyncio — chaque requête `await llm.ainvoke(...)` libère le thread pour traiter d'autres requêtes pendant que le modèle génère.

---

## Décorateurs

**Définition** : callable qui prend une fonction, retourne une version augmentée. `@decorator` = sucre syntaxique pour `func = decorator(func)`.

**3 piliers** : (1) first-class functions — les fonctions sont des objets passables en argument, (2) closures — `wrapper` capture `func` via `__closure__` même après la fin de l'exécution du décorateur, (3) LEGB scope — `wrapper` accède aux variables de l'enclosing scope.

```python
import functools

def decorator(func):
    @functools.wraps(func)  # préserve __name__, __doc__, __annotations__ — toujours mettre
    def wrapper(*args, **kwargs):
        # logique avant
        result = func(*args, **kwargs)
        # logique après
        return result
    return wrapper
```

**Cas prod** : décorateurs de retry, de cache (`@functools.lru_cache`), de validation de types, de logging automatique, de rate limiting.

---

## Générateurs

**Définition** : itérateur lazy — produit les valeurs à la demande, **O(1) mémoire** quelle que soit la taille du flux.

**Mécanisme** : `yield` suspend l'exécution et préserve l'état complet (variables locales + pointeur d'instruction dans le `PyFrameObject`) jusqu'au prochain `next()`. La fonction ne "meurt" pas — elle est suspendue.

**Usage unique** : un générateur s'épuise. Pour le réutiliser → recréer l'objet.

```python
def read_large_file(path):
    with open(path) as f:
        for line in f:
            yield line.strip()   # une ligne à la fois — jamais tout le fichier en RAM

# Cas réel : traitement de fichiers de logs volumineux en streaming
for line in read_large_file("logs_production.txt"):
    process(line)
```

**`send()` — générateur comme coroutine** : `yield` peut aussi **recevoir** une valeur via `gen.send(value)`. C'est le mécanisme sous-jacent d'asyncio avant `async/await`.

---

## MRO — Method Resolution Order

**Problème** : avec l'héritage multiple, dans quel ordre chercher une méthode ?

**C3 Linearization** : Python calcule un ordre de résolution déterministe — depth-first, gauche à droite, sans répétition. Consultable via `ClassName.__mro__`.

```python
class A: pass
class B(A): pass
class C(A): pass
class D(B, C): pass   # Diamond Problem

print(D.__mro__)      # D → B → C → A → object
```

**Règle** : `super()` suit le MRO — pas nécessairement la classe parente directe. Permet la coopération correcte dans l'héritage multiple.

---

## `@classmethod` vs `@staticmethod` vs `@property`

```python
class Date:
    def __init__(self, y, m, d):
        self._y, self._m, self._d = y, m, d

    @classmethod
    def from_string(cls, s: str):       # constructeur alternatif — cls préservé dans les sous-classes
        y, m, d = s.split('-')
        return cls(int(y), int(m), int(d))

    @staticmethod
    def is_valid_year(year: int) -> bool:  # utilitaire — pas besoin de cls ou self
        return 1900 <= year <= 2100

    @property
    def year(self) -> int:              # accès sans parenthèses, logique de validation possible
        return self._y

    @year.setter
    def year(self, value: int):
        if not self.is_valid_year(value):
            raise ValueError(f"Année invalide : {value}")
        self._y = value
```

---

## FastAPI & Pydantic — Stack Production

**FastAPI** : framework ASGI async basé sur les type hints Python. Génère automatiquement la doc OpenAPI (Swagger). Utilisé sur SITA et le projet RAG.

**Pydantic** : validation de données à runtime via les type hints. `BaseModel` génère `__init__`, validation automatique et sérialisation JSON.

```python
from fastapi import FastAPI
from pydantic import BaseModel, Field

app = FastAPI()

class QueryRequest(BaseModel):
    question: str = Field(..., min_length=1, max_length=500)
    top_k: int = Field(default=5, ge=1, le=20)

class QueryResponse(BaseModel):
    answer: str
    sources: list[str]
    faithfulness_score: float

@app.post("/query", response_model=QueryResponse)
async def query(request: QueryRequest) -> QueryResponse:
    # Pydantic valide automatiquement la requête — 422 si invalide
    result = await rag_pipeline.ainvoke(request.question, top_k=request.top_k)
    return QueryResponse(**result)
```

**Points à citer** :
- `async def` + `await` → FastAPI gère des milliers de requêtes concurrentes sur un seul thread.
- `response_model` → sérialisation et validation automatique de la réponse.
- Dependency Injection native (`Depends`) → injecter des services (DB session, auth) sans couplage.

---

## Pièges Python courants (questions piège fréquentes)

**Argument mutable par défaut** :
```python
# FAUX — la liste est créée une seule fois à la définition de la fonction
def append_to(element, lst=[]):
    lst.append(element)
    return lst

append_to(1)  # [1]
append_to(2)  # [1, 2]  ← surprise !

# CORRECT
def append_to(element, lst=None):
    if lst is None:
        lst = []
    lst.append(element)
    return lst
```

**Late binding dans les closures** :
```python
# FAUX — i est résolu au moment de l'appel, pas à la création
funcs = [lambda: i for i in range(3)]
print([f() for f in funcs])  # [2, 2, 2]

# CORRECT — capturer i par valeur
funcs = [lambda i=i: i for i in range(3)]
print([f() for f in funcs])  # [0, 1, 2]
```

---

# PARTIE 2 — LangChain / LangGraph

---

## LangChain — Architecture LCEL

**Définition** : framework Python d'abstractions standardisées pour construire des applications LLM — chaînes, agents, mémoire, retrieval.

**LCEL (LangChain Expression Language)** : syntaxe pipe `|` pour composer des Runnables. Chaque composant expose `.invoke()`, `.stream()`, `.batch()`, `.ainvoke()` — la composition est automatiquement streamable et async.

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_messages([
    ("system", "Tu es un expert juridique. Réponds uniquement à partir du contexte fourni."),
    ("human", "Contexte : {context}\n\nQuestion : {question}")
])
llm = ChatOpenAI(model="gpt-4o", temperature=0)
parser = StrOutputParser()

chain = prompt | llm | parser   # chaque | connecte des Runnables

# Invocation sync
answer = chain.invoke({"context": "...", "question": "Qu'est-ce que l'AI Act ?"})

# Streaming (token par token) — même interface
for chunk in chain.stream({"context": "...", "question": "..."}):
    print(chunk, end="", flush=True)
```

**Composants clés** :

| Composant | Rôle | Exemple |
|---|---|---|
| `ChatPromptTemplate` | Template paramétrable avec rôles system/human/ai | Prompt RAG avec contexte injecté |
| `ChatOpenAI` / `ChatAnthropic` | Interface unifiée vers les modèles | Swappable sans changer la chain |
| `StrOutputParser` / `JsonOutputParser` | Parse la sortie du LLM | Extraction structurée |
| `RunnablePassthrough` | Passe les données telles quelles dans la chain | Combiner retrieval + question |
| `RunnableLambda` | Enveloppe une fonction Python en Runnable | Preprocessing custom |

---

## LangChain Tools & Agents

**Tools** : fonctions décorées que le LLM peut décider d'appeler. Le LLM reçoit la description et les paramètres, décide de l'appeler, et reçoit le résultat.

```python
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI

@tool
def search_legal_database(query: str) -> str:
    """Recherche des articles de loi dans la base juridique. Utilise pour toute question légale."""
    # appel ChromaDB ou API
    return "Article L.123-4 du Code Civil..."

@tool
def calculate_deadline(date_str: str, delay_days: int) -> str:
    """Calcule une échéance légale à partir d'une date et d'un délai en jours."""
    from datetime import datetime, timedelta
    date = datetime.strptime(date_str, "%Y-%m-%d")
    return str(date + timedelta(days=delay_days))

llm = ChatOpenAI(model="gpt-4o")
llm_with_tools = llm.bind_tools([search_legal_database, calculate_deadline])
# Le LLM peut maintenant décider d'appeler ces outils
```

---

## LangGraph — Agents Stateful

**Définition** : extension LangChain pour construire des **workflows sous forme de graphes orientés avec état partagé**. Contrairement aux chains linéaires, LangGraph permet des **cycles**, des **branchements conditionnels** et un **état persistant** entre les nœuds.

**Pourquoi LangGraph plutôt que LangChain Agents standard** :
- Les agents réels nécessitent des **boucles** (retry, réflexion, planification itérative).
- L'état doit être **partagé et modifiable** à travers toutes les étapes.
- Contrôle fin sur le flux : interrompre pour validation humaine, checkpointer, reprendre après erreur.

```python
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode
from langchain_core.messages import HumanMessage, AIMessage
from typing import TypedDict, Annotated
import operator

# 1. État partagé — Annotated + operator.add pour accumuler les messages
class AgentState(TypedDict):
    messages: Annotated[list, operator.add]   # add = append, pas remplacer
    context: str

# 2. Nœud LLM — appelle le modèle avec les tools bindés
def call_llm(state: AgentState) -> dict:
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": [response]}   # ajouté à la liste existante via operator.add

# 3. Nœud Tools — exécute les tool calls retournés par le LLM
tool_node = ToolNode([search_legal_database, calculate_deadline])

# 4. Router — continue si le LLM a appelé un tool, termine sinon
def should_continue(state: AgentState) -> str:
    last_message = state["messages"][-1]
    if hasattr(last_message, "tool_calls") and last_message.tool_calls:
        return "tools"
    return END

# 5. Construction du graphe
graph = StateGraph(AgentState)
graph.add_node("llm", call_llm)
graph.add_node("tools", tool_node)

graph.set_entry_point("llm")
graph.add_conditional_edges("llm", should_continue, {"tools": "tools", END: END})
graph.add_edge("tools", "llm")   # après les tools → retour au LLM (boucle ReAct)

app = graph.compile()

# Invocation
result = app.invoke({
    "messages": [HumanMessage(content="Quelle est l'échéance pour contester un licenciement ?")],
    "context": ""
})
```

**Patterns avancés** :

- **Checkpointing** : `graph.compile(checkpointer=MemorySaver())` — sauvegarde l'état à chaque nœud → reprise après erreur, sessions multi-tours, audit trail.
- **Human-in-the-loop** : `graph.compile(interrupt_before=["tools"])` — interrompt avant l'exécution des tools pour validation humaine.
- **Multi-agent** : un graphe orchestrateur crée des sous-graphes spécialisés et leur délègue des tâches via des nœuds d'appel.
- **ReAct pattern** : la boucle `llm → tools → llm` ci-dessus **est** ReAct — le LLM raisonne (Thought), appelle un outil (Action), reçoit le résultat (Observation), et recommence jusqu'à converger.

**Lien avec le projet RAG juridique** : LangGraph orchestre le pipeline complet — retrieval → reranking → génération → vérification Ragas (faithfulness). Si le score de faithfulness est insuffisant, le graphe boucle sur le retrieval avec une reformulation de la requête (query rewriting).

---

## RAG en Production — Pipeline Complet

```python
from langchain_community.vectorstores import Chroma
from langchain_openai import OpenAIEmbeddings
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

# ── INDEXING (hors-ligne) ──────────────────────────────────────────
splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50,          # chevauchement pour ne pas couper une info en deux
    separators=["\n\n", "\n", ". ", " "]  # respecte les séparateurs naturels
)
chunks = splitter.split_documents(docs)
vectorstore = Chroma.from_documents(chunks, OpenAIEmbeddings())

# ── RETRIEVAL + GENERATION (en ligne) ─────────────────────────────
retriever = vectorstore.as_retriever(search_kwargs={"k": 20})  # large first pass

rag_prompt = ChatPromptTemplate.from_messages([
    ("system", "Réponds UNIQUEMENT à partir du contexte ci-dessous. "
               "Si l'information n'y est pas, dis-le explicitement."),
    ("human", "Contexte :\n{context}\n\nQuestion : {question}")
])

def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | rag_prompt
    | ChatOpenAI(model="gpt-4o", temperature=0)
    | StrOutputParser()
)

answer = rag_chain.invoke("Quelles sont les obligations RGPD pour un système de scoring ?")
```

**Points production à citer** :
- `temperature=0` sur les LLM de génération RAG — on veut la réponse la plus déterministe, pas de créativité.
- `k=20` au retriever + reranker Cross-Encoder pour sélectionner les top-3 → moins de tokens, meilleure faithfulness.
- System prompt durcissant l'ancrage dans le contexte — contre la connaissance paramétrique.

---

# PARTIE 3 — GenAI Transversal

---

## Transformer — Ce qu'il faut savoir

**Architecture** : Encoder (lit toute la séquence, bidirectionnel, extrait le contexte) + Decoder (génère token par token, masked self-attention + cross-attention sur l'Encoder).

**Self-Attention — formule** :
```
Attention(Q, K, V) = softmax(QKᵀ / √dₖ) · V
```
- **Q** (Query) : ce que je cherche. **K** (Key) : ce que j'offre. **V** (Value) : ce que je transmets si choisi.
- **√dₖ** : normalisation pour éviter que le produit scalaire explose en haute dimension → saturation du softmax → gradients qui disparaissent.
- **Multi-Head** : H têtes en parallèle, chacune apprend des relations différentes (syntaxe, sémantique, coréférence). Résultats concaténés puis reprojetés.

**Composants critiques** :
- **Positional Encoding** : sinusoïdes ajoutées aux embeddings pour injecter l'ordre — le Transformer est naturellement invariant à la position sans ça.
- **Residual Connections + LayerNorm** : `output = LayerNorm(x + sublayer(x))` → court-circuite le vanishing gradient sur les modèles profonds (GPT-4 : 96 couches).
- **Feed-Forward** : réseau 2 couches appliqué token par token après l'attention — affine la représentation individuelle.

**BERT vs GPT** :

| | BERT | GPT |
|---|---|---|
| Architecture | Encoder seul | Decoder seul |
| Lecture | Bidirectionnelle — voit tout | Gauche → droite (causal mask) |
| Tâche de pré-entraînement | MLM — prédit des tokens masqués | CLM — prédit le token suivant |
| Usage | Compréhension, classification, NER | Génération, LLM conversationnels |
| Exemples | BERT, RoBERTa, CamemBERT | GPT-4, Claude, Llama |

**Point oral** : BERT est bidirectionnel parce qu'il prédit des tokens **masqués au milieu** — il peut regarder à gauche et à droite. GPT ne peut pas : il génère gauche → droite, chaque token ne voit que son passé. Cette contrainte est aussi sa force pour la génération.

---

## Hallucinations — Types & Mitigation

**Définition** : le modèle génère du contenu factuellement incorrect, inventé, ou non fondé dans le contexte fourni.

**2 types** :
- **Intrinsèque** : contradiction avec le contexte fourni ("le document dit X mais le modèle dit Y").
- **Extrinsèque** : information inventée non vérifiable à partir du contexte (le modèle utilise sa connaissance paramétrique).

**Causes principales** :
1. Connaissance paramétrique qui "écrase" le contexte injecté.
2. Instructions de prompt trop laxistes.
3. Contexte trop long → Lost in the Middle.
4. Température trop élevée.

**Solutions** :
1. System prompt strict : "Réponds UNIQUEMENT à partir du contexte. Si l'info n'y est pas, dis-le."
2. `temperature=0` pour la génération RAG.
3. Reranker → moins de chunks, mieux ordonnés.
4. Faithfulness monitoring (RAGas) en production → alerte si score < seuil.
5. LLM-as-a-Judge : un modèle évalue si la réponse est grounded dans le contexte.

---

## RAG — Points Clés Entretien

**Pourquoi c'est le standard entreprise** : données privées sans réentraînement + réduction des hallucinations + données à jour en temps réel.

**Pipeline complet** :
```
Chunking → Embedding → Vector DB → Retrieval (Bi-Encoder, top-100)
        → Reranking (Cross-Encoder, top-5) → Injection prompt → LLM
```

**Bi-Encoder vs Cross-Encoder** :
- Bi-Encoder : question et document encodés **séparément** → comparaison vectorielle → rapide, scalable, précision moyenne.
- Cross-Encoder : question + document **concaténés** dans un seul modèle → score token par token → lent, non-scalable, précision maximale.
- En prod : Bi-Encoder pour le retrieval initial, Cross-Encoder pour le reranking.

**Lost in the Middle** : les LLM accordent moins d'attention au milieu d'un long contexte → placer les chunks les plus pertinents en tête + réduire le nombre de chunks injectés.

**RAGas Triad** :
- **Faithfulness** : la réponse est-elle ancrée dans les documents ? → détecte les hallucinations extrinsèques.
- **Answer Relevance** : la réponse répond-elle vraiment à la question ?
- **Context Recall** : le retriever a-t-il ramené les bons documents ?

**Diagnostic rapide** : Faithfulness basse + Context Recall élevé → problème Generator. Context Recall bas → problème Retriever. Les deux se corrigent indépendamment.

---

## Prompt Engineering — Techniques à citer

**Zero-shot** : juste l'instruction, sans exemple. Suffit pour les tâches simples avec un modèle puissant.

**Few-shot** : quelques exemples `(input, output)` dans le prompt pour conditionner le comportement. Efficace pour imposer un format ou un style.

**Chain-of-Thought (CoT)** : demander au modèle de raisonner étape par étape avant de répondre — "Let's think step by step". Améliore drastiquement les performances sur les tâches de raisonnement complexe.

**ReAct (Reasoning + Acting)** : alternance de pensées (`Thought`) et d'actions (`Action` → `Observation`). Standard pour les agents avec tools. Implémenté nativement dans LangGraph.

**Structured Output** : forcer le modèle à répondre en JSON avec un schéma Pydantic — `.with_structured_output(MyModel)` dans LangChain.

---

## Fine-Tuning & LoRA

**Quand RAG vs Fine-tuning** :

| Critère | RAG | Fine-tuning |
|---|---|---|
| Données qui changent souvent | Idéal | Non — réentraîner à chaque update |
| Données privées volumineuses | Idéal | Coûteux (compute) |
| Changer le style/format de réponse | Difficile (prompt engineering) | Idéal |
| Vocabulaire ultra-spécifique | Partiel | Idéal |
| Réduire les coûts d'inférence | Non | Oui (modèle plus petit fine-tuné) |

**Règle de décision** : RAG d'abord. Fine-tuning si le comportement souhaité ne peut pas être capturé dans un prompt ou si les coûts d'inférence doivent être réduits par distillation.

**LoRA — mécanisme** : `W' = W₀ + BA` où `W₀` est gelé, et seules les petites matrices `A ∈ ℝ^{r×k}` et `B ∈ ℝ^{d×r}` sont entraînées. Rang `r ≪ min(d,k)` → 10 000× moins de paramètres. `BA` fusionné dans `W₀` à l'inférence → zéro surcoût de latence.

**QLoRA** : LoRA + quantification **4-bit NF4** des poids gelés → entraîner un 70B sur un seul GPU grand public (24 Go VRAM).

**Catastrophic Forgetting** : le gradient des nouvelles données écrase les poids des anciennes connaissances. Solution : **data mixing** (~10% de données générales dans le dataset de fine-tuning).

---

## RLHF & Alignement

**Pipeline** :
1. **SFT** : paires `instruction → réponse idéale` rédigées par des humains → base comportementale.
2. **Reward Model** : annotateurs classent des variantes de réponses → modèle secondaire qui encode le "goût humain" en score scalaire.
3. **PPO** : le LLM génère, le Reward Model note, PPO ajuste les poids pour maximiser le score avec contrainte **KL-divergence** (ne pas trop s'écarter du SFT → évite le **reward hacking** : réponses absurdes mais bien notées).

**Objectif HHH** : Helpful, Honest, Harmless.

**DPO (Direct Preference Optimization)** : supprime le Reward Model. Optimise directement le LLM sur des paires `(réponse préférée, réponse rejetée)` via une loss dérivée analytiquement. Plus simple, plus stable. Adopté par Llama 3, Mistral.

---

## MCP — Model Context Protocol

**Définition** : protocole open-source standardisant la connexion entre agents IA et systèmes externes. Analogie : **USB des agents IA** — un contrat universel, n'importe quel client compatible s'y connecte.

**Règle critique** : le LLM ne se connecte **jamais directement** au serveur. Le Client reçoit l'instruction du LLM, exécute l'action sur le Serveur, retourne le résultat au LLM.

**3 ressources** : Resources (lecture seule) / Tools (actions exécutables) / Prompts (templates).

**Lien architectural** : Client = Port (abstraction), Serveur = Adapter (implémentation) → Clean Architecture appliquée aux agents.

---

# PARTIE 4 — AWS Cloud

---

## Services Fondamentaux

**Compute — quand utiliser quoi** :

| Service | Cas d'usage | Limite |
|---|---|---|
| **EC2** | Serveur long-running, training ML, app stateful | Payer même si idle |
| **Lambda** | Événements ponctuels, traitements courts | 15 min max, cold start, stateless |
| **ECS** | Containers Docker managés, microservices | Moins flexible que K8s |
| **EKS** | Kubernetes managé, orchestration complexe | Overhead opérationnel |

**Serverless trade-offs** :
- **Cold start** : première invocation après inactivité → latence de 100ms à 1s selon le runtime. Mitigation : Provisioned Concurrency (Lambda pre-warmée).
- **Stateless obligatoire** : l'état doit être externalisé (DynamoDB, ElastiCache, S3).

**Storage** :
- **S3** : object storage, 11 nines de durabilité. Datasets ML, artefacts de modèles, logs. Accès via SDK ou URL signée.
- **EBS** : block storage attaché à une EC2 (disque dur virtuel, persistent).
- **EFS** : file system NFS partagé entre plusieurs EC2 — utile pour partager des modèles ou datasets entre instances.

**Databases** :
- **RDS** : SQL managé (PostgreSQL, Aurora). ACID, backups automatiques, Multi-AZ.
- **DynamoDB** : NoSQL clé-valeur, latence < 1ms, scalabilité horizontale infinie. Pas de JOIN — modélisation par accès pattern.
- **ElastiCache** (Redis) : cache in-memory. Sessions, feature store ML, résultats d'API coûteux.

**Messaging** :
- **SQS** : queue point-à-point. Un consommateur par message. Découplage de workers async.
- **SNS** : pub/sub fan-out. Un message → tous les abonnés simultanément.
- **Pattern production** : SNS → fan-out vers plusieurs SQS → chaque queue traitée par son propre worker indépendant.

---

## IAM — Identity & Access Management

**Concept central** : **principe du moindre privilège** — chaque ressource/utilisateur n'a que les permissions strictement nécessaires.

**Composants** :
- **IAM Role** : identité attachée à un service AWS (EC2, Lambda, ECS task) qui lui donne des permissions sans exposer de credentials.
- **IAM Policy** : document JSON qui définit les actions autorisées (`Allow`/`Deny`) sur quelles ressources et sous quelles conditions.
- **Instance Profile** : mécanisme qui attache un IAM Role à une EC2 — les credentials sont rotés automatiquement toutes les heures.

**Bonne pratique à citer** : ne jamais coder des credentials AWS en dur dans le code. Utiliser les IAM Roles → les credentials sont injectés automatiquement par le metadata service EC2.

---

## Architecture MLOps sur AWS — Vécu SITA

```
Data ingestion   → S3 (datasets bruts + features engineered)
Training         → EC2 GPU + MLflow (tracking des expériences)
Model registry   → S3 + MLflow (versioning des modèles)
Serving          → ECS + FastAPI + Docker (API d'inférence)
Monitoring       → CloudWatch logs + Prometheus + Grafana (métriques)
Drift detection  → Lambda (job planifié par EventBridge) → alerte SNS → SNS → SQS → worker de réentraînement
```

**Points à valoriser en entretien** :
- Architecture **cloud hybride** (on-premise + AWS) : contraintes de sécurité strictes — VPC privé, IAM roles, SSL/TLS end-to-end, pas d'exposition publique des services internes.
- **20 000 vols validés en prod** avec monitoring continu de drift — si la distribution des données de vol dérive → alerte → réentraînement automatique.
- Optimisation performance avec **Numba** (JIT compilation) — réduction temps de calcul sur les algorithmes de prédiction de connectivité en vol.

---

# PARTIE 5 — Posture Consultante

---

## Ce que "posture consultante" signifie concrètement

Un consultant AI Engineer chez Athoria est l'**interlocuteur technique du client** — pas seulement un développeur. L'entretien évalue simultanément :

1. **Maîtrise technique** : précision, profondeur, pas de bluff.
2. **Clarté de communication** : expliquer un concept complexe à un non-technicien, utiliser des analogies.
3. **Recul stratégique** : justifier un choix technique avec ses trade-offs, ne pas suivre des modes.
4. **Honnêteté calibrée** : savoir dire "je ne connais pas en détail, mais voici comment je l'aborderais" — les consultants qui bluffent se font démasquer immédiatement.
5. **Orientation client** : partir du problème métier, pas de la technologie.

---

## Formules prêtes à l'oral

**Justifier un choix technique** :
> "J'ai choisi X plutôt que Y parce que dans ce contexte, [contrainte principale], X offre [bénéfice clé]. Le trade-off est [inconvénient], qu'on adresse en [mitigation]. Si le contexte changeait vers [scénario alternatif], Y deviendrait plus pertinent."

**Gérer une technologie méconnue** :
> "Je n'ai pas travaillé avec cette technologie spécifiquement en production, mais j'ai [technologie équivalente]. Les concepts de [principe commun] s'appliquent — la courbe d'apprentissage serait courte. Je serais honnête avec le client sur ce delta et je l'adresserais en amont."

**Face à un problème flou** :
> "Avant de proposer une architecture, j'aurais besoin de comprendre [volume de données / SLA de latence / budget / fréquence de mise à jour des données]. Ces contraintes changent fondamentalement l'approche."

**Traduire du technique en business** :
> "Techniquement, ça s'appelle le 'drift de données'. Concrètement pour le client, ça signifie que le modèle qui prenait de bonnes décisions il y a 6 mois commence à se tromper — parce que la réalité a changé mais le modèle n'a pas été mis à jour. Le coût business de ne pas monitorer ça, c'est [impact métier chiffré]."

---

## Questions anticipées & angles de réponse

**"Parle-moi de ton projet RAG juridique."**

Structure STAR (Situation → Tâche → Action → Résultat) :
- **Situation** : LegalMap avait un problème de scalabilité des consultations juridiques — les juristes passaient du temps à rechercher manuellement des textes de loi.
- **Tâche** : concevoir un chatbot juridique fiable, capable de répondre sur une base de documents privés, sans halluciner des articles de loi inexistants.
- **Action** : architecture RAG avec LangGraph (orchestration), ChromaDB (vector store), pipeline de reranking (Cross-Encoder), évaluation continue avec Ragas (faithfulness, answer relevance, context recall). System prompt durci contre la connaissance paramétrique.
- **Résultat** : score de faithfulness > 0.85 en production, réduction des hallucinations de ~60% vs un LLM sans RAG.

---

**"Comment tu choisirais entre RAG et fine-tuning pour un client ?"**

> "Ma question de départ serait : est-ce que le problème est un problème de **connaissance** (le modèle ne sait pas ce que contient notre base de documents) ou un problème de **comportement** (le modèle ne répond pas dans le bon format ou style) ?
>
> Si c'est de la connaissance → RAG, surtout si les données changent fréquemment.
> Si c'est du comportement → fine-tuning avec LoRA ou QLoRA.
> En pratique je commencerais toujours par le RAG — c'est plus rapide à itérer et moins coûteux. Le fine-tuning est réservé aux cas où le RAG atteint ses limites."

---

**"Qu'est-ce que le GIL et pourquoi ça compte pour un AI engineer ?"**

> "Le GIL est un mutex de CPython qui empêche plusieurs threads d'exécuter du bytecode Python simultanément. Pour un AI engineer, ça signifie concrètement : si j'utilise `threading` pour paralléliser du preprocessing ou de l'inférence CPU-intensive, je ne gagnerai rien — voire je perdrai du temps à cause du context switching. La solution c'est `multiprocessing` pour les tâches CPU-bound. Pour les appels I/O vers un LLM ou une base de données, `asyncio` est le bon choix — le GIL est relâché pendant l'attente réseau. C'est précisément pour ça que FastAPI est async — il gère des milliers de requêtes LLM en parallèle sur un seul thread."

---

**"Tu t'y connais en LangGraph ?"**

> "Oui, c'est ce que j'ai utilisé pour le projet RAG juridique. LangGraph permet de construire des agents avec un état persistant partagé entre les étapes, des cycles et des branchements conditionnels — ce qu'une chain LangChain simple ne peut pas faire. Concrètement : dans mon RAG, si le score de faithfulness d'une réponse est insuffisant, le graphe reboucle sur le retrieval avec une requête reformulée, plutôt que de livrer une réponse potentiellement hallucinée. C'est le pattern ReAct implémenté au niveau de l'architecture."

---

**"Comment tu estimerais le coût d'un projet IA pour un client ?"**

> "Je décompose en 4 catégories : (1) **Compute d'entraînement** — combien d'heures GPU, quel type d'instance, fine-tuning ou RAG ? (2) **Coût d'inférence** — combien de requêtes par jour × coût par token × longueur moyenne des contextes. Sur un RAG, le contexte peut être 10× plus long qu'une requête simple. (3) **Infrastructure** — vector DB, serving (ECS/EKS), monitoring. (4) **Maintenance** — réentraînement, gestion du drift, évaluation continue. Le plus souvent sous-estimé par les clients c'est le (4) — un modèle sans monitoring coûte très cher à corriger après coup."

---

**"Qu'est-ce que l'AI Act et pourquoi ça compte ?"**

> "L'AI Act est le cadre réglementaire européen qui classe les systèmes IA par niveau de risque — inacceptable (interdit), élevé (obligations strictes), limité, minimal. Les systèmes à risque élevé — scoring de crédit, systèmes médicaux, outils RH, juridique — doivent être documentés, auditables, avec traçabilité complète et possibilité d'explication des décisions.
>
> Pour un consultant AI chez Athoria, ça signifie penser conformité dès la phase de design — 'compliance by design'. Chez LegalMap j'ai cartographié les obligations réglementaires pour leur produit juridique, qui entre dans la catégorie risque élevé. Ne pas anticiper ça en amont peut bloquer tout le déploiement produit."

---

**"Parle-moi d'une décision technique difficile que tu as prise."**

> "Sur SITA, j'ai proposé de refondre l'architecture d'un module critique en Clean Architecture, alors que l'équipe avait une pression calendaire forte. L'argument technique était clair — le couplage existant rendait les tests impossibles et chaque nouvelle feature cassait quelque chose. J'ai présenté le trade-off explicitement : 2 semaines d'investissement pour une réduction estimée de 40% du temps de debug à long terme. La décision a été validée. Rétrospectivement, c'est aussi une leçon de posture consultant — présenter les trade-offs factuellement plutôt que de vendre la solution idéale sans contrainte."

---

**"Comment tu gères un client qui veut un LLM custom plutôt qu'une solution RAG ?"**

> "Je questionne d'abord le besoin réel derrière la demande. Souvent 'je veux mon propre LLM' cache en réalité 'je veux que mes données restent privées' ou 'je veux des réponses cohérentes avec notre domaine'. Dans 80% des cas, un RAG bien conçu sur un LLM du marché répond à ces besoins pour 10× moins de coût et 10× moins de délai. Je propose toujours une démo comparative : RAG vs modèle brut sur leurs données réelles. Le client décide sur des faits, pas sur des intuitions."

---

## Ce qui te différencie des autres juniors

1. **Production réelle** : 20 000 vols validés en prod, monitoring de drift en continu — pas des notebooks de TP.
2. **Double compétence technique + réglementaire** : AI Act/RGPD appliqués chez LegalMap, capable d'accompagner un client sur la conformité dès le design.
3. **Architecture-first mindset** : Clean Architecture et SOLID appliqués sur un système critique aviation — pas des buzzwords.
4. **Multi-sectoriel** : aviation (SITA) + LegalTech (LegalMap) + retail (KRYS) = adaptabilité sectorielle, argument clé en conseil.
5. **GenAI end-to-end** : RAG de l'indexing au monitoring Ragas, LangGraph en production, contrôle des hallucinations — pas uniquement de la théorie.
6. **Orientation résultats** : chaque expérience a un impact chiffré ou mesurable — le langage des clients.
