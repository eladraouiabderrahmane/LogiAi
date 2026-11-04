# Compte rendu : TP API LogiAI

**Noms :** El Adraoui Abderrahmane  **Groupe :** G3

## 1. Environnement et premiers tests

- Sortie de `python -m pytest` au départ (commit `d7b0050`, « TP API : starter ») :

```text
============================= test session starts =============================
platform win32 -- Python 3.11.0, pytest-9.2.0, pluggy-1.7.0
rootdir: C:\Users\anask.ANAS\Desktop\dev\LogiAI
plugins: anyio-4.15.1
collected 13 items

tests\test_api.py ..FFFF......F                                          [100%]

=========================== short test summary info ===========================
FAILED tests/test_api.py::test_predict_valide_renvoie_200 - assert 422 == 200
FAILED tests/test_api.py::test_predict_forme_de_la_reponse - AssertionError: ...
FAILED tests/test_api.py::test_predict_probabilite_entre_0_et_1 - KeyError: '...
FAILED tests/test_api.py::test_predict_trajet_risque_plus_eleve_que_trajet_facile
FAILED tests/test_api.py::test_modele_absent_renvoie_503 - assert 422 == 503
=================== 5 failed, 8 passed, 1 warning in 3.02s ====================
```

- Pourquoi certains tests qui attendent un 422 passent-ils déjà, alors que rien n'est écrit ?

Au départ, le schéma `Livraison` est presque vide et refuse tout champ inconnu. Résultat : **toute requête est rejetée avec un 422**, même une livraison correcte, car les champs envoyés (`distance_km`, `poids_kg`, etc.) ne sont pas déclarés dans le schéma. Les tests qui attendent un 422 passent donc, mais **pour la mauvaise raison** : la donnée invalide n'est pas détectée par une borne, elle est refusée parce que *tout* est refusé. C'est un faux vert.

Les résultats d'entraînement relevés avec `python -m logiai.train` :

- Accuracy de la baseline (toujours répondre « à l'heure ») : 0.726 (72,6 %)
- Accuracy du modèle : 0.774 (77,4 %)
- Rappel du modèle (sur 100 retards réels, combien sont détectés) : 0.464, soit environ 46 retards détectés sur 100

À quoi sert l'API si le rappel est faible ? Même avec un rappel limité, le dispatcher obtient un signal qu'il n'avait pas du tout avant : une probabilité de retard pour chaque livraison, accessible depuis n'importe quel outil, et une alerte sur une partie des cas à risque. Un modèle imparfait reste plus utile que l'absence d'information, à condition de connaître ses limites. Le modèle gagne seulement 4,8 points d'accuracy sur la baseline, mais la baseline ne détecte aucun retard (rappel 0), alors que le modèle en détecte environ la moitié.

## 2. Mon schéma

- Mes bornes et leur justification physique (une ligne par variable) :

| Variable | Min | Max | Pourquoi |
|---|---|---|---|
| distance_km | 1 | 1000 | Un trajet de livraison fait au moins 1 km ; au-delà de 1000 km, ce n'est plus une livraison de ce type (une journée de route). |
| poids_kg | 0.1 | 30000 | Un colis pèse au moins quelques centaines de grammes ; 30 tonnes correspond à la charge utile d'un gros camion. |
| heure_depart | 0 | 23 | Une heure du jour, entier de 0 à 23. |
| pluie_mm | 0 | 200 | La pluie ne peut pas être négative ; 200 mm est un cumul extrême déjà exceptionnel. |
| trafic_index | 0 | 10 | Échelle définie par le contrat : 0 = fluide, 10 = saturé. |
| nb_arrets | 0 | 50 | Pas d'arrêt négatif ; 50 arrêts est déjà une tournée très chargée. |

`type_vehicule` est un `Literal["moto", "fourgon", "camion"]` : toute autre valeur est refusée.

Les bornes sont des **bornes physiques** (ce qu'un trajet réel peut afficher), et non le min et le max de `livraisons.csv`, sinon une livraison réelle un peu plus longue serait refusée à tort.

## 3. Ma fonction predict()

```python
@app.post("/v1/predict", response_model=PredictionOut)
def predict(livraison: Livraison):
    predictor = state.get("predictor")
    if predictor is None:
        raise HTTPException(
            status_code=503,
            detail="Modèle non chargé : lancez 'python -m logiai.train' puis redémarrez le serveur.",
        )
    return PredictionOut(**predictor.predict(livraison.model_dump()))
```

## 4. Quatre requêtes

| Requête | Code obtenu | Pourquoi |
|---|---|---|
| livraison normale | 200 | Les 7 champs respectent le schéma ; le modèle est chargé et renvoie `probabilite_retard`, `alerte`, `seuil` et `model_version`. |
| distance_km = -5 | 422 | Violation de la borne minimale (`ge=1`) : FastAPI refuse la requête avant d'appeler `predict()`. |
| type_vehicule = "avion" | 422 | « avion » n'appartient pas au `Literal["moto", "fourgon", "camion"]`. |
| modèle absent (supprimez models/ et relancez) | `/health` : 200 avec `model_loaded: false` ; `/v1/predict` : 503 | Le serveur démarre, mais `state.get("predictor")` vaut `None` : le service est indisponible, ce n'est pas une erreur du client. |

## 5. Questions de compréhension

1. **Pourquoi charge-t-on le modèle au démarrage et pas à chaque requête ?**
   Lire et désérialiser `model.joblib` depuis le disque est lent. Le faire à chaque requête ajouterait une latence importante, consommerait inutilement des ressources et ne tiendrait pas la charge avec plusieurs clients simultanés. Le charger une seule fois dans `lifespan` le garde en mémoire, ce qui permet de répondre vite à chaque requête.

2. **Qui écrit le contrôle « pluie <= 100 », et où est-il écrit ?**
   C'est nous (le développeur de l'API) qui l'écrivons, une seule fois, dans le schéma Pydantic `Livraison` du fichier `logiai/api/schemas.py`, par exemple `pluie_mm: float = Field(ge=0, le=100, ...)`. C'est ensuite **FastAPI/Pydantic qui l'exécute** automatiquement avant d'appeler la fonction ; on n'écrit pas de `if` à la main dans la route.

3. **Différence entre 422 et 503 ?**
   Le **422** est une erreur du client (famille 4xx) : les données envoyées sont invalides, et le client doit les corriger. Le **503** est une erreur du serveur (famille 5xx) : la requête est correcte, mais le service ne peut pas y répondre pour le moment (modèle non chargé). La même requête réussira une fois le modèle disponible.

4. **Pourquoi une route `/v1/predict` plutôt que `/predict` ?**
   Le préfixe `/v1` est la **version** de l'API. Il permet de faire évoluer le contrat (nouveaux champs, nouvelle réponse) en publiant une `/v2` sans casser les clients existants qui utilisent encore `/v1`.

5. **Que se passerait-il si la distance était déclarée `int` ?**
   Une distance décimale comme `120.5` serait refusée avec un 422 (« valeur avec une partie fractionnaire »), alors qu'elle est parfaitement réaliste. Le contrat serait plus strict que le besoin réel et ferait échouer des clients légitimes. Un `float` est le bon type pour une distance.

## 6. Décision : modèle absent

**Choix : option A, démarrer quand même et répondre 503 aux demandes de prédiction.**

- **Ce que A fait perdre :** le service paraît « en vie » alors qu'il ne sert à rien ; les clients reçoivent des erreurs au moment de la requête plutôt qu'un échec immédiat au démarrage. Il faut surveiller le service pour s'en apercevoir.
- **Ce que B (refuser de démarrer) fait perdre :** plus aucun processus ne répond, donc l'endpoint `/health` est injoignable. Celui qui supervise ne peut pas distinguer « modèle absent » de « serveur planté », et un simple message dans les logs remplace un diagnostic exploitable.
- **Pourquoi A :** avec A, `/health` reste disponible et indique `model_loaded: false`. Le superviseur voit précisément ce qui manque, et les clients reçoivent un 503 clair avec un message qui dit quoi faire. Le diagnostic est donc plus facile.


