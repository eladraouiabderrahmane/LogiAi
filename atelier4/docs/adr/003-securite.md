# ADR 003 : sécurité de l'API LogiAI

## Décision 1 : où transmettre la clé (en-tête, URL ou corps) ?
- **Votre choix :** dans l'en-tête `X-API-Key`. La clé est ainsi séparée des données métier, et FastAPI la contrôle avant de valider le corps : sans clé, une pluie de 500 mm donne 401 et non 422.
- **Ce que l'autre option ferait perdre :**
  - **URL :** on perdrait la confidentialité de la clé, car une URL est enregistrée dans les journaux du serveur et dans l'historique du navigateur, et elle se copie facilement.
  - **Corps :** on perdrait la séparation entre authentification et données. Le schéma `Livraison` refuse les champs inconnus, il faudrait donc y mêler la clé, et une route GET comme `/v1/model-info` n'a pas de corps.

## Décision 2 : quelles origines autoriser (CORS) ? Une liste précise ou « * » ?
- **Votre choix :** une liste précise, `http://localhost:5500` et `http://127.0.0.1:5500` (`config.ALLOWED_ORIGINS`). Seule la page prévue peut lire les réponses depuis un navigateur ; essai fait, le port 5501 est bloqué.
- **Ce que l'autre option ferait perdre :** avec `*`, n'importe quel site ouvert dans le navigateur d'un utilisateur pourrait appeler l'API et lire la réponse. On perdrait le contrôle des pages autorisées et le navigateur ne protégerait plus l'utilisateur. CORS ne remplace pas la clé : `curl` n'est pas concerné. Le coût de la liste est de devoir l'ajuster et redémarrer l'API à chaque nouvelle origine.
