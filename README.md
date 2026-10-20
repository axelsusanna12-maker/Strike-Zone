# Strike Zone – version en ligne gratuite

Un seul petit serveur Node.js (aucune dépendance) qui fait tout :
- il **héberge le jeu** (index.html) ;
- il gère la **recherche d'adversaire** (10 s, sinon bots), les **codes d'équipe** et le **classement**, pour **tout le monde**, sans compte.

## Fichiers
- `server.js` : le serveur
- `index.html` : le jeu
- `package.json` : indique à l'hébergeur comment démarrer
- `render.yaml` : (facultatif) réglages prêts pour Render

## Tester chez toi (facultatif)
Installe Node.js (nodejs.org), puis dans le dossier : `node server.js`
Ouvre http://localhost:3000 dans deux onglets : lance le même niveau dans les deux, ils se retrouvent.

## Mettre en ligne gratuitement (Render)
1. Crée un compte gratuit sur github.com, puis un nouveau dépôt (New repository).
2. Dans le dépôt : « Add file » > « Upload files » et envoie `server.js`, `index.html`, `package.json` (et `render.yaml`). Valide (Commit).
3. Crée un compte gratuit sur render.com (« Sign in with GitHub »).
4. « New + » > « Web Service » > choisis ton dépôt.
5. Réglages : Runtime = Node, Build Command = `npm install`, Start Command = `npm start`, Instance Type = **Free**.
6. « Deploy ». Au bout de 1–2 minutes tu obtiens une adresse `https://xxxx.onrender.com` : **c'est ton jeu, avec le multijoueur, les codes d'équipe et le classement.** Donne cette adresse à tes amis.

## À savoir sur l'offre gratuite de Render
(d'après la documentation de Render ; les conditions peuvent changer, vérifie sur render.com/free)
- Le service **s'endort après 15 min sans trafic** et met environ 1 minute à se réveiller. Le premier joueur peut donc attendre un peu, et la recherche d'adversaire bascule sur des bots tant que la connexion n'est pas prête.
- Tant que des joueurs sont connectés, il reste éveillé. Pour le garder éveillé en continu, un service de « ping » gratuit (par exemple UptimeRobot) peut appeler `https://xxxx.onrender.com/healthz` toutes les 5 minutes. Render offre 750 h gratuites par mois, ce qui couvre un seul service allumé en permanence.
- Les fichiers du serveur sont **effacés à chaque redémarrage** : le classement repart alors de zéro. Pour le garder, voir ci-dessous.

## Garder le classement pour de bon (facultatif, gratuit)
1. Crée une base Redis gratuite sur upstash.com.
2. Copie l'URL REST et le jeton (token) de la base.
3. Dans Render, onglet « Environment », ajoute deux variables :
   `UPSTASH_REDIS_REST_URL` et `UPSTASH_REDIS_REST_TOKEN`.
Le serveur les utilise tout seul au prochain démarrage.

## Héberger le jeu ailleurs que sur Render
Si `index.html` est posé sur un autre site (itch.io, Netlify…), ouvre-le avec l'adresse de ton serveur :
`https://ton-site/?server=wss://xxxx.onrender.com`

## Limites honnêtes
- Le jeu tourne dans le navigateur de chaque joueur : le classement n'a **aucune protection anti-triche** (quelqu'un de motivé peut s'écrire un faux score).
- Le code du panel admin est lisible dans `index.html` : il cache le panel aux joueurs, ce n'est pas une vraie sécurité.
- Les vies, pièces et la progression sont sauvegardées dans le navigateur de chaque joueur.
