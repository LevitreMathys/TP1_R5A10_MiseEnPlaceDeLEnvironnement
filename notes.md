# TP 1 — Réponses TP1

## Étape 1 — Lire avant de lancer (15 min)


**Q2.** Trois services ont une section `volumes`, un seul n'en a pas. Lequel, et qu'est-ce que ça implique pour ses données ?

> Le service qui ne possède pas de section `volumes` est le service redis. Cela implique que lorsque le conteneur se lancera, le conteneur gérera ses données mais qu'une fois que ce dernier sera `down` alors il perdera toute sa progression, comme un reset du conteneur.

**Q3.** Que signifie la ligne `"15432:5432"` ? Les deux nombres ne désignent pas la même chose.

> Cette ligne permet de configurer les ports sur lesquelles le service sera associé. Le port 15432 (gauche) et le port de la machine local sur lequel le conteneur tourne et le port 5432 (droite) est le port du conteneur.

**Q4.** Les mots de passe sont écrits en clair dans le fichier. Est-ce acceptable ici ? Le serait-ce sur un serveur de production ?

> Dans le cadre de ce TP ca n'est pas réellement dérangeant (sauf s'il est inspiré d'un projet réel). Mais pour un projet d'une plus grande empleur, cela peut l'être car il suffit qu'une personne ai accès a ce fichier pour pouvoir se connecter à tous les services et ainsi accéder aux données. Pour cela il faut utiliser un fichier .env

---

## Étape 2 — Démarrer les quatre moteurs (20 min)

**Q5.** Les commandes ci-dessus utilisent docker compose exec. Qu'est-ce que ça fait exactement ? Où s'exécute la commande redis-cli ?

> - ```Docker compose exec``` permet d'éxecuter des commandes directement dans les services du conteneur en cours. <u>Exemple de commande :</u> ```docker compose exec mysql -u username -p password databasename``` va se connecter à la base de données ```databasename``` avec l'utilisateur ```username``` dans le conteneur.
> - La commande ```redis-cli``` se lance directement dans le conteneur ```redis``` et non sur la machine locale.

---

## Étape 3 — L'application PixelHub (35 min)

**Q6.** La base de données tourne dans un conteneur Docker, et la chaîne de connexion dit Host=localhost. Pourquoi est-ce que ça fonctionne ? (Indice : relisez votre réponse à la Q3.)*

> Host est localhost dans le fichier ```appsettings.json``` car ```localhost``` définit la machine hôte sur laquelle le service tourne et comme la base de données postgres tourne dans le conteneur il est nécessaire de mettre le terme ```localhost``` dans l'URL de connexion à la base de données.

**Q7.** Arrêtez l'application (Ctrl+C), relancez-la. Les trois joueurs sont-ils dupliqués ? Pourquoi ?

> Les trois joueurs ne sont pas dupliqués après le redemarrage de l'API car ```Program.cs``` utilise la fonction ```EnsureCreated()``` qui permet de créer la base seulement si cette dernière existe.

---

## Étape 4 — Créer un joueur : `POST /players` (15 min)

**Q8.** Dans votre code, que vaut `player.Id` *avant* l'appel à `SaveChangesAsync()` ? Et après ? Qui a choisi la valeur 4 : votre code, EF Core ou PostgreSQL ?

> Le code envoie l'objet dans Id c'est donc Postgres qui choisi la valeur en fonction de ce qui est déjà inséré en base.

---

## Étape 5 — ⭐ PostgreSQL : la transaction qui protège la monnaie (25 min)

**Q9.** Que valent les soldes après le ROLLBACK ? Imaginez maintenant que le serveur s'éteigne entre les deux UPDATE, sans transaction : dans quel état serait la base, et qui y perdrait ?

> Après le ```ROLLBACK``` les valeurs reviennent à leur état initial (Nova = 1200 coins et Krayz = 350 coins). Si la panne arrive entre les 2 ```UPDATE``` alors seul ```Nova``` aura perdu 100 coins et ```Krayz``` ne les aura pas récupérer.

**Q10.** Pendant la transaction de A, quel solde de Nova voyaient B et l'API ? Pourquoi l'UPDATE de B a-t-il dû attendre ? Imaginez que A et B soient deux achats lancés au même instant par Nova sur deux téléphones : que pourrait-il se passer si la base ne bloquait pas B ?

> - Pendant la transaction de A, B et l'API voyaient toujours le solde de Nova à 1200 (état initial). 
> - L'UPDATE de B à dû attendre car A a lancer la commande ``` BEGIN``` et donc à pris la priorité sur la valeur de coins, donc B ne pouvait pas changer cette valeur tant que A a encore le contrôle sur cette données. Cela évite les conflits entre session.
> - Si la somme des deux achats était supérieure aux coins obtenus, le total de coins pourrait être négatif.

**Q11.** Qui a refusé l'achat d'Ombre : votre code C# ou la base ? Pourquoi le SELECT suivant a-t-il échoué lui aussi ? Après le ROLLBACK, la contrainte existe-t-elle toujours, et qu'est-ce que ça vous apprend sur PostgreSQL ?

> - C'est la base qui a refusé l'achat d'Ombre car la contrainte ```coins_positifs``` était configurée sur la base de données Postgres.
> - Le ```SELECT``` suivant a échoué car lorsqu'une contrainte n'est pas respectée dans le ```BEGIN``` Postgres bloque toutes les autres transactions jusqu'à un ```COMMIT``` ou à un ```ROLLBACK```.
> - Ce que ça m'a appris: 
>   - Que Postgres annule absolument toutes les transactions et manipulations après un BEGIN tant qu'un COMMIT n'a pas été fait
>   - Que Postgres configuré de sorte à pouvoir sécuriser et organiser les données durant les transactions.

## Étape 6 — Premiers pas dans Redis, MongoDB et Neo4j (25 min)

**Q12.** Que renvoie TTL temporaire au fil des secondes, puis une fois les 10 secondes écoulées ? Et GET temporaire ? Citez une donnée de PixelHub qui gagnerait à disparaître toute seule.

> - ```TTL temporaire``` renvoie le temps de vie restant de la variable ```temporaire```
> - ```GET temporaire``` renvoie la valeur de ```temporaire``` donc au début elle vaut ```"vite"``` et au bout de 10 secondes ```(nil)```
> - Une donnée qui gagnerait du temps à disparaître toute seule peut-être un token.

**Q13.**INCR lit, ajoute 1 et réécrit en une seule commande. Pourquoi est-ce plus sûr que de faire un GET, d'ajouter 1 dans le code C#, puis un SET, si 200 joueurs déclenchent le compteur au même moment ? (Pensez à ce que vous avez vu en 5.2.)

> ```INCR``` est plus sûr car si 200 utilisateurs changent une valeur en même temps avec le code C# cela peut faire ralentir et mettre les 199 autres utilisateurs en attente.

**Q14.** Les deux jeux sont dans la même collection, mais n'ont pas les mêmes champs (plateformes est une liste, multijoueur un objet imbriqué). Aurait-on pu ranger les deux dans une même table PostgreSQL ? À quel prix ? Rapprochez votre réponse de l'exercice du catalogue de jeux (section 2 du cours CM1_etudiants_Panorama.md).

> La manipulation est possible au prix de nombreuses données qui seront ```NULL```


**Q15.** La requête « ami d'un ami » se lit presque comme un dessin. Comment l'écririez-vous en SQL, avec une table `Amities(joueur_id, ami_id)` ? Combien de jointures faudrait-il pour « ami d'un ami d'un ami » ?

>Pour écrire cette requête en SQL il faudrait faire une seule jointure entre deux même tables amitiés.

**Q16.** Pourquoi `DELETE` seul a-t-il été refusé ? Que fait `DETACH` de plus ?

> `DELETE` a été refusé car les noeuds avait des relation entre eux. Pour remedier à ce problème on utilise `DETACHE` pour spécifier qu'il faut supprimer toute relation des noeuds avant de les supprimer.

---

## Étape 7 — Les données survivent-elles ? Arrêter proprement (10 min)

**Q17.** La clé `survivant` a-t-elle survécu au `stop` ? Au `down` ? Et les joueurs de PostgreSQL ? Expliquez la différence avec le mot **volume**.

> La donnée `survivant` a survecu au `docker compose stop` car seules le conteneur est arreté mais les volumes sont conservé. Tandis que le `docker compose down` a supprimé les conteneur et les données car `Redis` n'a pas de volume