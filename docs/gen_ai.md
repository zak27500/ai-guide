# GenAI

---

## I. Pourquoi ne pas simplement mettre tous les documents dans le contexte ?

**Question** : "On entend souvent parler de la fenêtre de contexte et des Tokens. Pourquoi est-il risqué d'augmenter la taille du contexte à l'infini pour donner tous les documents au modèle d'un coup, plutôt que d'utiliser une architecture plus complexe ?"

**Réponse** :

L'utilisation d'une fenêtre de contexte massive présente trois risques majeurs.

**Dégradation de l'attention (Lost in the Middle)** : les recherches montrent que les modèles mémorisent mieux les informations situées au début et à la fin du prompt. Plus le milieu est dense, plus le risque d'omettre une information cruciale est élevé — même si elle est techniquement présente dans le contexte.

**Explosion des coûts et de la latence** : le coût de l'inférence est proportionnel au nombre de tokens traités. Un contexte saturé ralentit considérablement le Time To First Token (TTFT), ce qui dégrade directement l'expérience utilisateur en production.

**Bruit informationnel** : injecter des données non pertinentes augmente le risque d'hallucinations, car le modèle peut établir des corrélations erronées entre informations qui n'auraient pas dû être traitées ensemble.

C'est pourquoi on privilégie des architectures RAG (Retrieval-Augmented Generation) pour ne fournir au modèle que les fragments strictement nécessaires à chaque requête.

---

## II. Qu'est-ce qui détermine réellement la qualité d'une réponse RAG ?

**Question** : "Dans un pipeline RAG classique, on transforme nos documents en vecteurs pour les stocker. Mais qu'est-ce qui détermine la qualité de la réponse finale : est-ce uniquement la puissance du LLM, ou y a-t-il une étape cruciale avant que le LLM ne reçoive la question ?"

**Réponse** :

La qualité d'un RAG dépend en premier lieu de la précision de la phase de **Retrieval**. Si le contexte récupéré est bruité ou hors sujet, même le meilleur LLM produira une réponse médiocre ou hallucinera — le modèle ne peut pas corriger ce qu'on ne lui a pas donné.

**Optimisation du Retrieval** : l'approche hybride Bi-Encoder + Cross-Encoder est la référence. Le Bi-Encoder (recherche vectorielle) offre une grande scalabilité mais manque parfois de nuance sémantique. Le Cross-Encoder agit comme un filtre de précision : il reclasse les résultats en traitant simultanément la requête et le document, capturant mieux les interactions complexes — au prix d'une latence plus élevée.

**Fiabilisation de la génération** : pour limiter les hallucinations, on contraint le LLM via le Prompt Engineering ("Réponds uniquement en utilisant le contexte fourni") et le Few-Shot Prompting pour ancrer le format attendu.

**Évaluation continue** : un système LLM-as-a-Judge — souvent un modèle plus petit (SLM) spécialisé — vérifie en temps réel la fidélité de la réponse par rapport au contexte (Faithfulness) et sa pertinence par rapport à la question (Relevance).

---

## III. Comment gérer une question qui nécessite plusieurs outils successifs ?

**Question** : "Un RAG classique est souvent 'statique' : il cherche, puis il répond. Comment feriez-vous si l'utilisateur pose une question qui nécessite plusieurs recherches successives, ou de consulter un outil externe (API météo, calculateur) avant de répondre ?"

**Réponse** :

Pour des besoins multi-étapes ou nécessitant des outils externes, on utilise des **Agents IA** basés sur le pattern **ReAct (Reason + Act)**.

**Fonctionnement** : l'agent reçoit une boîte à outils (Toolbelt). À chaque itération, il analyse l'état actuel, décide de l'outil à appeler, traite le résultat, et ré-évalue s'il dispose d'assez d'informations pour répondre ou s'il doit continuer à itérer.

**Choix d'orchestration** : pour une logique complexe avec des cycles et de la persistance d'état, LangGraph est adapté — il offre un contrôle fin sur les transitions et le checkpointing. Pour une application en production où la latence et le débogage sont critiques, coder l'orchestration from scratch via le Function Calling natif des LLM est souvent plus pertinent : cela évite les abstractions qui peuvent masquer des erreurs ou complexifier inutilement le code.

**Limite structurelle** : sans mécanisme de limite d'itérations (`max_iterations`), un agent peut boucler indéfiniment. En production, on impose toujours un plafond d'itérations et une gestion explicite des cas d'erreur.

---

## IV. Comment garantir qu'une IA ne va pas halluciner en production ?

**Question** : "C'est bien beau d'avoir un agent qui utilise des outils, mais comment pouvez-vous garantir à un client que votre IA ne va pas inventer une information ou utiliser un outil de manière erronée ?"

**Réponse** :

Pour garantir la fiabilité d'un système GenAI, on déploie une stratégie d'évaluation à plusieurs niveaux — aucun niveau seul n'est suffisant.

**La Triade RAG** : trois métriques complémentaires mesurées automatiquement via des frameworks comme Ragas ou TruLens.
- **Faithfulness** : la réponse est-elle ancrée dans le contexte fourni, ou le modèle invente-t-il ?
- **Context Relevance** : le retriever a-t-il récupéré les bons documents ?
- **Answer Relevance** : la réponse répond-elle réellement à la question posée ?

**LLM-as-a-Judge** : un modèle supérieur (ou un SLM spécialisé) évalue les sorties selon des critères précis — toxicité, ton, exactitude factuelle. Scalable et automatisable, il permet une surveillance continue sans coût humain constant.

**Human-in-the-Loop (HITL)** : pour les décisions critiques (juridiques, médicales, financières), l'humain intervient comme validateur final. L'IA propose, l'humain dispose. Ce mécanisme sert doublement : sécuriser la décision et collecter des données de feedback pour améliorer le modèle en continu (RLHF).

L'objectif est un **Guardrail multicouche** où les métriques automatiques filtrent les cas standards, et la validation humaine couvre les cas limites où les métriques atteignent leurs limites.

---

## V. Fine-tuning ou RAG — comment choisir ?

**Question** : "Si mon modèle ne connaît pas assez bien le jargon technique spécifique de mon entreprise, est-ce que je devrais lancer un Fine-tuning sur mes données ou rester sur un RAG ? Quels sont les critères de décision ?"

**Réponse** :

Le choix dépend de ce qu'on veut changer : la **connaissance** ou le **comportement** du modèle.

**RAG** : idéal pour fournir des faits à jour et réduire les hallucinations sur des données documentaires. La base de connaissances est modifiable à tout moment sans réentraînement. C'est la solution pour la connaissance factuelle.

**Fine-tuning** : indispensable pour que le modèle adopte un jargon technique précis, un format de réponse strict, un ton ou une "voix" spécifique, ou des comportements qui ne peuvent pas être injectés via le prompt. C'est la solution pour le comportement.

**Les deux combinés** : dans les cas exigeants, on fait du Fine-tuning pour le comportement + du RAG pour la connaissance — le modèle sait comment parler, et le RAG lui fournit quoi dire.

**Optimisation du Fine-tuning** : pour contrôler les coûts GPU, on utilise le PEFT (Parameter-Efficient Fine-Tuning) via **LoRA** — on insère des adaptateurs de rang faible (matrices A et B) dans les couches denses, et on n'entraîne qu'une fraction des paramètres totaux. Si les ressources sont très limitées, **QLoRA** ajoute une quantification en 4-bit NormalFloat (NF4) qui réduit l'empreinte mémoire de façon drastique tout en préservant les performances, permettant de fine-tuner des modèles massifs sur du matériel grand public.

---

## VI. Comment optimiser la latence et le débit en production ?

**Question** : "Une fois que vous avez votre modèle fine-tuné ou votre pipeline RAG, comment gérez-vous le compromis entre la latence (vitesse de réponse) et le débit (nombre d'utilisateurs simultanés) ? Quels outils ou techniques de compression de modèle pourriez-vous utiliser ?"

**Réponse** :

Pour optimiser l'inférence en production (AWS Bedrock, SageMaker, ou endpoint custom), on agit sur quatre leviers complémentaires.

**Distillation de connaissances (Knowledge Distillation)** : on entraîne un modèle plus petit (l'élève) à reproduire les probabilités de sortie d'un modèle plus large (le professeur). Résultat : un modèle avec beaucoup moins de paramètres qui conserve une grande partie des capacités de raisonnement du modèle original.

**Élagage (Pruning)** : on supprime les connexions ou les poids les moins utiles dans le réseau de neurones pour alléger le modèle sans dégrader significativement ses performances.

**Quantification (Post-Training Quantization)** : réduire la précision des poids (de FP32/FP16 vers INT8 ou INT4) réduit l'empreinte mémoire et accélère les calculs sur GPU/NPU. C'est la même technique que QLoRA, appliquée à l'inférence.

**Speculative Decoding** : un petit modèle très rapide génère une ébauche de réponse, que le grand modèle (plus lent mais plus précis) valide ou corrige en une seule passe. Cela réduit considérablement la latence perçue par l'utilisateur sans sacrifier la qualité finale.

---

## VII. Quel est l'avantage du mécanisme de Self-Attention sur les RNN ?

**Question** : "Quel est l'avantage technique majeur du mécanisme de Self-Attention des Transformers par rapport au traitement séquentiel des anciens modèles (RNN/LSTM), notamment en termes d'entraînement et de compréhension du contexte ?"

**Réponse** :

Avant les Transformers, les RNN et LSTM traitaient les mots un par un, dans l'ordre séquentiel. Ce traitement séquentiel posait deux problèmes majeurs.

**Disparition du gradient** : sur des séquences longues, les informations du début de la phrase avaient du mal à atteindre la fin de la chaîne de traitement — le modèle "oubliait" le contexte lointain.

**Impossibilité de paralléliser** : puisque chaque mot attendait le résultat du mot précédent, l'entraînement ne pouvait pas exploiter la puissance des GPU modernes pour traiter plusieurs tokens simultanément.

**L'apport du Self-Attention** : le mécanisme Query-Key-Value permet à chaque token de calculer sa relation avec tous les autres tokens de la séquence en une seule opération matricielle — quelle que soit leur distance. "Le chat que j'ai vu hier mange" : le verbe "mange" peut être relié directement à "chat" sans passer par tous les mots intermédiaires.

**L'Attention Multi-Têtes** : au lieu d'un seul regard global, le modèle divise son attention en plusieurs têtes indépendantes, chacune spécialisée sur un aspect différent — relations syntaxiques (sujet-verbe), relations sémantiques (désambiguïsation), dépendances lointaines (pronom et référent). La combinaison des têtes produit une représentation beaucoup plus riche qu'une attention unique.

---

## VIII. Comment le Transformer connaît-il l'ordre des mots ?

**Question** : "Si je dis 'Le chat mange la souris' ou 'La souris mange le chat', les mots sont les mêmes. Sans mécanisme spécial, le Transformer les verrait comme un simple sac de mots identique. Comment fait-il pour connaître l'ordre ?"

**Réponse** :

Puisque le Transformer traite tous les tokens simultanément (parallélisme), il perd naturellement la notion d'ordre séquentiel. Pour compenser, on utilise l'**Encodage Positionnel (Positional Encoding)**.

On ajoute à chaque vecteur de mot (embedding) un signal mathématique unique — dans le papier original "Attention Is All You Need", des fonctions sinus et cosinus à différentes fréquences — qui encode la position précise du token dans la séquence.

Grâce à cela, le vecteur représentant "chat" en position 2 est différent du vecteur "chat" en position 5. Le modèle peut donc distinguer "Le chat mange la souris" de "La souris mange le chat", même en traitant tous les mots en parallèle.

Les modèles plus récents utilisent des variantes comme le **RoPE (Rotary Positional Embedding)** ou l'**ALiBi**, qui gèrent mieux l'extrapolation vers des séquences plus longues que celles vues à l'entraînement.

---

## IX. Pourquoi avoir séparé Encoder et Decoder ?

**Question** : "Aujourd'hui, les modèles comme GPT sont appelés 'Decoder-only', tandis que BERT est 'Encoder-only'. Pourquoi a-t-on fini par séparer ces deux parties au lieu de toujours utiliser le Transformer complet ?"

**Réponse** :

L'architecture Encoder-Decoder originale (Transformer complet) était conçue pour la traduction — transformer une séquence source en une séquence cible. Pour d'autres tâches, chaque moitié s'est révélée plus efficace seule.

**Encoder (ex : BERT)** — bidirectionnel : il voit toute la phrase d'un coup dans les deux directions (gauche et droite). Cela produit des représentations contextuelles riches, idéales pour des tâches de **compréhension** — classification, extraction d'entités, similarité sémantique. L'entraînement se fait par Masked Language Modeling (MLM) : certains tokens sont masqués, le modèle apprend à les prédire en s'appuyant sur tout le contexte environnant.

**Decoder (ex : GPT)** — unidirectionnel (causal) : lors de l'entraînement, il n'a pas le droit de voir les tokens "du futur" — un masque (causal mask) bloque l'attention vers la droite. Il apprend à prédire le token suivant uniquement à partir du passé. C'est ce qui le rend performant pour la **génération** — complétion de texte, code, dialogue. Permettre au modèle de voir le futur pendant l'entraînement le ferait "tricher" et l'empêcherait d'apprendre à générer.

**En résumé** : Encoder = comprendre, Decoder = générer. Les LLM modernes (GPT, Claude, Llama) sont tous Decoder-only, car la génération auto-régressive est le paradigme dominant.

---

## X. Comment contrôler le niveau de créativité d'un LLM ?

**Question** : "Parfois l'IA est trop répétitive, et parfois elle part dans tous les sens. Quels paramètres permettent de régler ce niveau de créativité ou de déterminisme lors des appels API ?"

**Réponse** :

Le contrôle du comportement de génération repose sur deux paramètres clés.

**Température** : elle module la distribution de probabilité sur les tokens candidats. Une température de `0` rend le modèle "greedy" — il choisit toujours le token le plus probable, ce qui garantit des réponses reproductibles et précises, idéal pour du code ou des calculs. Une température élevée (`0.8`-`1.0`) aplatit la distribution, donnant une chance à des tokens moins probables — cela favorise la créativité et la variété, au risque d'incohérence.

**Top-P (Nucleus Sampling)** : au lieu de considérer tous les tokens possibles, on ne considère que le plus petit ensemble dont la probabilité cumulée atteint P (ex : `0.9`). Cela filtre les tokens totalement hors sujet tout en conservant une diversité contrôlée. Combiné à une température modérée, c'est le réglage le plus courant en production.

**Top-K** : variante qui limite le choix aux K tokens les plus probables (ex : Top-K=50). Moins flexible que Top-P car il ne s'adapte pas à la distribution réelle du modèle.

**Règles pratiques** :
- Code, SQL, réponses factuelles : `temperature=0`, `top_p=1`
- Chatbot conversationnel : `temperature=0.7`, `top_p=0.9`
- Génération créative (storytelling) : `temperature=0.9`, `top_p=0.95`
- Ne jamais fixer temperature ET top_p à des valeurs extrêmes simultanément.

---

## XI. Qu'est-ce que le preprocessing NLP et pourquoi est-il indispensable ?

**Question** : "Avant d'alimenter un modèle ou un moteur de recherche avec du texte brut, quelles étapes de prétraitement appliquez-vous et pourquoi ?"

**Réponse** :

Le **preprocessing NLP** est la chaîne de transformations appliquées au texte brut pour le rendre exploitable par un modèle ou un algorithme. Un texte mal préparé produit des représentations bruitées qui dégradent toutes les étapes suivantes.

**Les étapes classiques dans l'ordre** :

**Nettoyage** : suppression des balises HTML, des caractères spéciaux, de la ponctuation non significative, normalisation des espaces et des encodages.

**Lowercasing** : mise en minuscules pour que "Paris" et "paris" soient traités comme le même token. À désactiver si la casse est sémantiquement significative (NER, acronymes).

**Tokenisation** : découpage du texte en unités atomiques (tokens). Peut se faire au niveau mot, sous-mot (BPE, WordPiece — utilisé par BERT/GPT), ou caractère.

```python
from transformers import AutoTokenizer
tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")
tokens = tokenizer("Le RAG améliore les LLM")
# ['le', 'rag', 'améliore', 'les', 'll', '##m']
```

**Stopwords** : suppression des mots très fréquents et peu informatifs (le, la, de, et...). Utile pour TF-IDF et la recherche classique, mais souvent inutile pour les LLM qui ont déjà appris à les pondérer.

**Stemming / Lemmatisation** : réduction d'un mot à sa racine.
- Stemming (rapide, brutal) : "mangeait" → "mang"
- Lemmatisation (plus précise, contextualisée) : "mangeait" → "manger"

**Pourquoi moins critique avec les LLM** : les modèles comme BERT ou GPT ont été entraînés sur du texte brut et gèrent nativement la morphologie, les stopwords et la casse. Le preprocessing reste indispensable pour la recherche vectorielle (qualité des embeddings), les modèles classiques (TF-IDF, SVM), et la réduction du bruit dans les pipelines RAG.

---

## XII. Qu'est-ce que TF-IDF et comment mesure-t-il la pertinence d'un terme ?

**Question** : "Expliquez le principe de TF-IDF. Comment permet-il de distinguer un terme important d'un terme fréquent mais peu informatif ?"

**Réponse** :

**TF-IDF (Term Frequency — Inverse Document Frequency)** est une mesure statistique qui quantifie l'importance d'un mot dans un document par rapport à une collection de documents (corpus).

**TF — Term Frequency** : fréquence brute d'un terme dans un document. Plus un mot apparaît dans le document, plus son TF est élevé.

```
TF(t, d) = nombre d'occurrences de t dans d / nombre total de mots dans d
```

**IDF — Inverse Document Frequency** : pénalise les termes qui apparaissent dans beaucoup de documents (donc peu discriminants). Un terme présent dans tous les documents (comme "le") aura un IDF proche de 0.

```
IDF(t) = log( N / (1 + df(t)) )

N     = nombre total de documents
df(t) = nombre de documents contenant le terme t
```

**TF-IDF = TF × IDF** : un terme obtient un score élevé s'il est fréquent dans le document courant ET rare dans le reste du corpus — c'est un terme caractéristique et discriminant.

**Exemple** : dans un corpus de 1000 articles sur le sport, le mot "football" apparaît dans 500 articles (IDF faible). Le mot "tiki-taka" n'apparaît que dans 3 articles (IDF élevé). Si "tiki-taka" apparaît 5 fois dans un article, son TF-IDF sera bien plus élevé que celui de "football" — il caractérise mieux le sujet de l'article.

**Limites** : TF-IDF ne capture pas le sens sémantique — "voiture" et "automobile" sont traités comme des termes totalement différents. C'est pourquoi on utilise les embeddings denses (Word2Vec, BERT) pour la recherche sémantique, et TF-IDF pour la recherche lexicale exacte.

---

## XIII. Qu'est-ce qu'un index inversé et comment fonctionne-t-il ?

**Question** : "Comment un moteur de recherche comme Elasticsearch retrouve-t-il instantanément les documents contenant un terme parmi des millions de documents ?"

**Réponse** :

Un **index inversé** est la structure de données fondamentale des moteurs de recherche full-text. Au lieu d'indexer les documents par leur identifiant (index "avant"), on indexe les termes et pour chaque terme, on stocke la liste des documents qui le contiennent.

**Construction** :

```
Documents :
  Doc 1 : "Le chat mange la souris"
  Doc 2 : "Le chien court vite"
  Doc 3 : "Le chat court dans le jardin"

Index inversé :
  "chat"   --> [Doc 1, Doc 3]
  "mange"  --> [Doc 1]
  "souris" --> [Doc 1]
  "chien"  --> [Doc 2]
  "court"  --> [Doc 2, Doc 3]
  "jardin" --> [Doc 3]
  "vite"   --> [Doc 2]
```

**Recherche** : pour la requête "chat court", on intersecte les listes de postings → `[Doc 1, Doc 3] ∩ [Doc 2, Doc 3]` = `[Doc 3]` — résultat instantané, même sur des milliards de documents.

**Ce que stocke réellement un index inversé** : les **posting lists** contiennent plus que des identifiants — elles incluent la position des termes dans le document (pour la recherche de phrases exactes), la fréquence (pour le scoring TF-IDF), et des métadonnées de champ (titre vs corps).

---

## XIV. Qu'est-ce qu'Elasticsearch et comment s'intègre-t-il dans un pipeline RAG ?

**Question** : "Qu'est-ce qu'Elasticsearch, quelle est sa différence avec une base vectorielle comme Chroma ou Pinecone, et comment combiner les deux dans un pipeline RAG hybride ?"

**Réponse** :

**Elasticsearch** est un moteur de recherche et d'analyse distribué basé sur Apache Lucene. Il utilise des index inversés pour la recherche full-text (BM25) et supporte depuis la version 8 la recherche vectorielle dense (k-NN / ANN).

**Recherche BM25 (lexicale)** : variante probabiliste améliorée de TF-IDF, standard de facto pour la recherche full-text. Recherche les correspondances exactes de termes — efficace pour les requêtes avec des mots-clés précis, des noms propres, des codes produits.

**Recherche vectorielle dense** : Elasticsearch indexe des vecteurs d'embedding et retrouve les k vecteurs les plus proches via HNSW (Hierarchical Navigable Small World — algorithme ANN). Efficace pour la recherche sémantique — "voiture" retrouve "automobile".

**RAG hybride — combiner les deux** :

```
Requête utilisateur
      |
      +--------> [BM25 / index inversé]  --> top-k résultats lexicaux
      |
      +--------> [Embedding + ANN]       --> top-k résultats sémantiques
      |
      v
[Reciprocal Rank Fusion (RRF)]
(fusion et reranking des deux listes)
      |
      v
[Cross-Encoder Reranker]
(tri final par pertinence réelle)
      |
      v
[LLM] --> Réponse ancrée dans les documents
```

**Pourquoi hybride** : la recherche lexicale capture les correspondances exactes que la recherche sémantique rate (noms propres, codes, acronymes). La recherche sémantique capture les synonymes et la paraphrase que la recherche lexicale rate. La combinaison donne la meilleure couverture.

**Elasticsearch vs Pinecone/Chroma** : Elasticsearch est une solution complète (full-text + vectoriel + analytics + clustering). Pinecone et Chroma sont spécialisés uniquement dans la recherche vectorielle, plus simples à opérer pour du RAG pur.

---

## XV. Qu'est-ce que la similarité cosinus et pourquoi l'utilise-t-on pour comparer des embeddings ?

**Question** : "Pourquoi utilise-t-on la similarité cosinus plutôt que la distance euclidienne pour comparer des vecteurs d'embeddings dans un pipeline RAG ?"

**Réponse** :

La **similarité cosinus** mesure l'angle entre deux vecteurs dans un espace de haute dimension, indépendamment de leur magnitude (norme).

**Formule** :

```
cos(θ) = (A · B) / (||A|| × ||B||)

Résultat entre -1 et 1 :
  1   → vecteurs identiques (même direction)
  0   → vecteurs orthogonaux (aucune relation)
 -1   → vecteurs opposés
```

**Pourquoi pas la distance euclidienne** : les embeddings représentent des directions sémantiques dans l'espace vectoriel. Un texte court ("chat") et un texte long ("le chat est un animal domestique félicace") peuvent être sémantiquement très proches, mais leurs vecteurs auront des normes très différentes — la distance euclidienne serait grande même si les directions sont similaires. La similarité cosinus normalise cette magnitude et compare uniquement l'orientation.

**Exemple intuitif** :
```
Embedding("roi")    ≈ [0.8, 0.2, 0.1, ...]
Embedding("reine")  ≈ [0.75, 0.25, 0.15, ...] -- même direction, magnitude légèrement différente

cos("roi", "reine") ≈ 0.92  -- très similaires
cos("roi", "pizza") ≈ 0.12  -- peu similaires
```

**En pratique dans un pipeline RAG** : après avoir calculé l'embedding de la question, on cherche les k chunks dont le cosinus avec la question est le plus élevé — ce sont les passages sémantiquement les plus proches, donc les plus susceptibles de contenir la réponse.

**Normalisation** : si tous les vecteurs sont normalisés à la norme unitaire (pratique courante), la similarité cosinus est équivalente au produit scalaire — ce qui permet d'utiliser des opérations matricielles très rapides (BLAS) pour calculer des milliers de similarités simultanément.

---

## XVI. Qu'est-ce qu'un CNN et en quoi diffère-t-il d'un Transformer ?

**Question** : "Qu'est-ce qu'un réseau de neurones convolutif (CNN) et pourquoi les Transformers les ont-ils supplantés pour le NLP, alors que les CNN restent dominants en vision par ordinateur ?"

**Réponse** :

Un **CNN (Convolutional Neural Network)** est une architecture de réseau de neurones qui applique des filtres (noyaux de convolution) sur les données d'entrée pour détecter des motifs locaux — bords, textures, formes en vision, n-grammes en NLP.

**Fonctionnement** :

```
Image / Texte d'entrée
        |
  [Couche de convolution]   -- filtre glisse sur l'entrée, détecte des motifs locaux
        |
  [Couche de pooling]       -- réduit la dimension, conserve les caractéristiques saillantes
        |
  [Couche de convolution]   -- détecte des motifs de plus haut niveau
        |
  [Couche fully connected]  -- classification / régression finale
```

**CNN vs Transformer pour le NLP** :

| Critère | CNN | Transformer |
|---|---|---|
| Champ réceptif | Local (fenêtre fixe) | Global (toute la séquence) |
| Dépendances longues | Difficiles à capturer | Natives (Self-Attention) |
| Parallélisme | Partiel | Total |
| Coût mémoire | Faible | O(n²) avec la séquence |
| Vitesse d'entraînement | Rapide | Plus lent |

**Pourquoi les CNN ont été supplantés en NLP** : un CNN avec un filtre de taille 3 voit uniquement 3 mots consécutifs à la fois. Pour capturer une dépendance entre le début et la fin d'une phrase longue, il faudrait empiler de nombreuses couches. Le Transformer voit toute la séquence en une seule opération via Self-Attention.

**Pourquoi les CNN restent dominants en vision** : les images ont une structure locale forte — un pixel est fortement corrélé à ses voisins immédiats. Le biais inductif de localité des CNN est un avantage pour la vision. Les Vision Transformers (ViT) existent et sont performants, mais nécessitent beaucoup plus de données et de compute que les CNN pour des performances équivalentes sur des datasets moyens.
