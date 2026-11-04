# Compte rendu : tester, sécuriser et appeler l'API LogiAI

**Noms :** EL ADRAOUI Abderrahmane  **Groupe :** G3

## 1. Casser son API (activité 1)
| Cas | Requête (corps ou URL) | Prévu | Obtenu |
|---|---|---|---|
| champ manquant | `POST /v1/predict` sans `poids_kg` | 422 | 422 (`Field required`) |
| valeur absurde | `POST /v1/predict` avec `poids_kg: 99999` | 422 | 422 (`less than or equal to 3500`) |
| texte au lieu d'un nombre | `POST /v1/predict` avec `distance_km: "loin"` | 422 | 422 (`valid number`) |
| champ en trop | `POST /v1/predict` avec `couleur: "rouge"` | 422 | 422 (`Extra inputs are not permitted`) |
| mauvaise route | `POST /v1/predire` avec une livraison valide | 404 | 404 (`Not Found`) |

- Une requête acceptée alors qu'elle devait être refusée ? Laquelle ?

  Aucune parmi ces cinq : les prévisions étaient justes. En revanche, avant le sprint 1, une livraison valide envoyée sans clé recevait 200 (`probabilite_retard: 0.732`). Les deux valeurs impossibles acceptées n'apparaissent qu'avec les tests aux bornes (activité 2).

## 2. Les 2 bugs (activité 2)
| Bug | Test qui l'a révélé | Comportement avant | Correction |
|---|---|---|---|
| 1 : l'heure 24 est acceptée | `test_valeurs_limites[heure_depart-24-422]` | `heure_depart=24` : attendu 422, obtenu 200 | `schemas.py` : `le=24` devient `le=23` |
| 2 : un nombre d'arrêts négatif est accepté | `test_valeurs_limites[nb_arrets--1-422]` | `nb_arrets=-1` : attendu 422, obtenu 200 (aucune borne basse) | `schemas.py` : ajout de `ge=0` |

- Sortie de `python -m pytest` à la fin de l'activité 2 :

```
python -m pytest tests/test_casser.py tests/test_limites.py tests/test_api.py
43 passed, 1 warning

python -m pytest
10 failed, 44 passed, 1 warning    (les 10 échecs : tests/test_securite.py, pas encore écrits)
```

## 3. La clé API (sprint 1)
- Ma fonction `verify_api_key()` (collez le code) :

```python
def verify_api_key(x_api_key: str | None = Header(default=None)) -> str:
    """Authentification : renvoie le rôle ('user' ou 'admin') ou lève une 401."""
    if not x_api_key:
        raise HTTPException(
            status_code=401,
            detail="Clé API manquante : ajoutez l'en-tête X-API-Key.",
            headers={"WWW-Authenticate": "ApiKey"},
        )
    if _egal(x_api_key, config.ADMIN_KEY):
        return "admin"
    if _egal(x_api_key, config.API_KEY):
        return "user"
    raise HTTPException(
        status_code=401,
        detail="Clé API invalide",
        headers={"WWW-Authenticate": "ApiKey"},
    )
```

| Requête | Code obtenu |
|---|---|
| `/v1/predict` sans clé | 401 |
| `/v1/predict` fausse clé | 401 |
| `/v1/predict` clé utilisateur | 200 |
| `/v1/model-info` clé utilisateur | 403 |
| `/v1/model-info` clé administrateur | 200 |
| `/health` sans clé | 200 |

- Preuve que `.env` n'est pas dans Git (sortie de `git status`) :

```
$ git status --short
M  COMPTE_RENDU.md
M  docs/adr/003-securite.md
A  livraison.json
M  logiai/api/main.py
M  logiai/api/schemas.py
M  logiai/api/security.py
M  logiai/config.py
M  logiai/dataset.py
M  logiai/predict.py
M  logiai/train.py
M  tests/conftest.py
M  tests/test_api.py
M  tests/test_casser.py
M  tests/test_limites.py
M  tests/test_securite.py
M  web/index.html

$ git check-ignore -v .env
atelier4/.gitignore:5:.env	.env
```

`.env` n'apparaît pas dans la liste : il est ignoré par Git.

## 4. Le client web (sprint 2)
- Mon `fetch` (collez le code) :

```javascript
async function appelerApi(livraison, cle) {
  const reponse = await fetch(API_URL, {
    method: "POST",
    headers: { "Content-Type": "application/json", "X-API-Key": cle },
    body: JSON.stringify(livraison),
  });
  const corps = await reponse.json();
  return { status: reponse.status, corps };
}
```

- Erreur dans la console avant d'ajouter le CORS :

```
Access to fetch at 'http://localhost:8000/v1/predict' from origin 'http://localhost:5500' has been blocked by CORS policy: Response to preflight request doesn't pass access control check: No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

- Ce qui s'affiche : livraison normale / pluie de 500 mm / fausse clé :
  - **Livraison normale :** « Probabilité de retard : 73.2 %. Alerte : livraison à risque. »
  - **Pluie de 500 mm :** « pluie_mm : Input should be less than or equal to 100 »
  - **Fausse clé :** « Clé API absente ou invalide. »
  - **API arrêtée :** « API injoignable : le serveur est-il lancé ? Sinon, blocage CORS possible (voir la console, F12). »
- Expérience CORS (page servie sur le port 5501) : que disent la page et la console (F12) ? Pourquoi `curl` marche-t-il quand même ?
  - **La page** affiche le message « API injoignable… ».
  - **La console** indique : `Access to fetch at 'http://localhost:8000/v1/predict' from origin 'http://localhost:5501' has been blocked by CORS policy: Response to preflight request doesn't pass access control check: No 'Access-Control-Allow-Origin' header is present on the requested resource.`
  - **La requête OPTIONS** avec l'origine 5501 reçoit `400 Disallowed CORS origin`, alors que l'origine 5500 reçoit 200 avec `access-control-allow-origin`.
  - **`curl` marche** parce que CORS est une règle du navigateur : `curl` ne demande pas de permission et n'applique pas ce blocage. Même avec l'en-tête `Origin: http://localhost:5501` et la bonne clé, il obtient 200. Seule la clé protège l'API.
- Où voyez-vous la clé dans F12 ?

  Onglet Network, requête POST `predict`, en-têtes de la requête : `X-API-Key` contient la clé en clair. C'est pourquoi la page la demande à l'utilisateur. Écrite dans le JavaScript, elle serait lisible par tout visiteur dans le code source, et donc utilisable par n'importe qui.

- Remarque : sur ce poste, le port 5500 est occupé par le service Oracle `OracleOraDB12Home1TNSListener`. Pour les essais, la page a été servie sur le port 5502 et Chrome a redirigé `localhost:5500` vers ce port : l'origine vue par l'API reste bien `http://localhost:5500`.

## 5. Questions de compréhension
1. **Quelle différence entre 401, 403 et 422 ?** 401 : l'API ne sait pas qui appelle (clé absente ou fausse). 403 : l'appelant est identifié mais n'a pas le droit (clé utilisateur sur `/v1/model-info`). 422 : l'appelant est accepté mais les données ne respectent pas le schéma. La clé étant vérifiée avant les données, une pluie de 500 mm sans clé donne 401.
2. **Pourquoi la clé n'est-elle ni dans le code ni dans Git ?** Tout ce qui est commité reste dans l'historique, même après suppression du fichier, et toute personne ayant accès au dépôt pourrait la lire. Elle est donc dans `.env`, ignoré par Git, et `.env.example` indique seulement quoi configurer. Une clé déjà commitée doit être changée.
3. **Pourquoi comparer avec `compare_digest` plutôt qu'avec `==` ?** `==` s'arrête au premier caractère différent : la durée de la réponse pourrait révéler combien de caractères sont justes et permettre de deviner la clé petit à petit. `compare_digest` prend un temps qui ne dépend pas de la position de cette différence.
4. **Pourquoi `curl` fonctionne-t-il alors que la page du port 5501 est bloquée ?** Le blocage est fait par le navigateur, qui protège son utilisateur ; l'API répond sans savoir qui est le client. `curl` n'applique pas CORS, seule la clé compte pour lui.
5. **Pourquoi la validation faite par le navigateur ne remplace-t-elle pas celle de l'API ?** Elle se contourne facilement : `curl`, un programme Java ou la console F12 peuvent envoyer n'importe quelle requête sans passer par la page. Elle améliore le confort de l'utilisateur, mais seule la validation côté serveur garantit le contrôle des données.

## 6. Lien de la pull request
À compléter après création de la pull request.
