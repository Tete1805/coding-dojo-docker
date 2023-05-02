# Docker 101

## 1. Docker kesako?

En quelques mots, Docker permet de construire un environnement isolé du reste de la machine (on peut presque voir ça comme une machine virtuelle).
On appelle cet environnement un conteneur (container).
L’avantage, c’est qu’on s’assure que le logiciel qu’on fera tourner dans cet environnement tournera de la même manière sur toutes les machines.
On définit ce qui doit constituer cet environnement avec un simple fichier texte : le Dockerfile.

```docker
FROM alpine
CMD echo "hello world" # ou CMD ["echo", "hello world"]
```

Ensuite, on “build” l’image, le template dont on se servira plus tard :

```bash
docker build -t test-echo .
```

Ensuite, on lance un container qui va “jouer” cette image.

```bash
docker run test-echo
```

## 2. Image vs container

Il faut donc bien dissocier le dockerfile (la définition de ce que vous voulez faire), l’image (qui représente une version préfabriquée de l’environnement) et enfin le container (l’instance même de l’environnement).

- Modifier le dockerfile ne change rien d’autre sans build
- On peut builder plusieurs images sans jamais les jouer dans un container
- On peut lancer plusieurs containers avec la même image

Par exemple, on peut lancer plusieurs fois la commande au-dessus (`docker run test-echo`) et on constate que pour chacun des runs, docker a initié et stoppé un nouveau container. On voit les containers inactifs qui traînent avec `docker container ls -a` (`-a` permet de lister aussi les conteneurs inactifs).

```bash
CONTAINER ID   IMAGE       COMMAND                  CREATED         STATUS                     PORTS     NAMES
f40e76f3bac8   test-echo   "docker-entrypoint.s…"   5 seconds ago   Exited (0) 3 seconds ago             reverent_wilson
2df23a934e60   test-echo   "docker-entrypoint.s…"   6 seconds ago   Exited (0) 5 seconds ago             quirky_swartz
```

Docker assigne un nom aléatoire à chaque container, on peut forcer un nom particulier avec `docker run --name name image`

On peut supprimer un container avec `docker container rm id` ou directement avec `docker rm <id ou name>` (Docker offre quelques raccourcis.)

_NB _: On ne peut pas faire `docker run --name testing-echo test-echo` plusieurs fois de suite, car on essaierait de créer un container qui existe déjà.

```jsx
docker: Error response from daemon: Conflict.
The container name "/testing-echo" is already in use
by container "4f51ba9394dcc00c3baf6ca85aea345029d5e60cd9e8d6f0aefc2a563d2b3813".
You have to remove (or rename) that container to be able to reuse that name.
```

Pour le lancer de nouveau, on utilisera `docker container start <id ou name>` ou `docker start <id ou name>`.

_NB :_ On ne voit plus que le nom du container dans la console. (`-a` pour “s’attacher” et voir l’output de nouveau.)

## 3. `CMD` vs `RUN`

Créons le fichier suivant

```docker
FROM alpine
RUN echo foo > baz.txt
RUN echo bar >> baz.txt
CMD cat baz.txt
```

Si on build l’image, le fichier `baz.txt` sera créé mais le `cat` ne sera effectué qu’au moment de l’instanciation du container.
_NB :_ On peut remplacer `CMD` par `ENTRYPOINT`.

## 4. Comprendre l’exécution

Créons le fichier suivant et rendons-le exécutable

```bash
# sleep.sh => chmod +x sleep.sh
echo "Going to bed for now…"
sleep 30
echo "Nap over, bye!"
```

Modifions l’image docker comme ceci

```docker
# dockerfile
FROM alpine
WORKDIR /app
COPY sleep.sh sleep.sh
CMD /app/sleep.sh
```

Après le build de l’image, lançons le container et connectons-nous dans une autre console au container !

`docker exec --rm -it <container name> sh`

Si on attend une trentaine de secondes, les deux terminaux se fermeront en même temps.
Le container s’arrête quand il n’y a plus rien à faire.

## 5. Quelques commandes dans le CLI

```bash
> docker image ls

REPOSITORY   TAG      IMAGE ID       CREATED         SIZE
test-node    latest   b21247d8113d   2 minutes ago   991MB

> docker container ls
> docker inspect id
> docker volume prune
...
```

## 6. Le network

Tout ça, c’est bien, mais on voudrait faire communiquer des containers entre eux.

Créons d’abord un micro serveur Node

```jsx
// index.js
const http = require('http');

const host = '0.0.0.0';
const port = 9001;

const requestListener = function (req, res) {
  console.log('Received a request');
  res.writeHead(200);
  res.end('My first server!\n');
};

const server = http.createServer(requestListener);
server.listen(port, host, () => {
  console.log(`Server is running on http://${host}:${port}`);
});
```

```docker
# dockerfile.server
FROM node:18.12.1-buster-slim
WORKDIR /app
COPY . .
USER node
ENTRYPOINT ["node", "index.js"]
```

Construisons l’image du serveur avec `docker build -t server-node -f dockerfile.server .`

Lançons un container avec `docker run --name test-server-node --rm server-node`

Plusieurs points :

1.  le container s’exécute mais ne rend pas la main (`-d` pour “detached” permet de rendre la main immédiatement)
2.  CTRL+C ne termine pas le container (`docker container stop testing-node -t 1` pour forcer l’arrêt) ⇒ C’est une petite subtilité de node et on peut lancer le container plutôt comme ceci pour s’éviter ça `docker run --name testing-node --rm —init server-node` (cf. https://github.com/Yelp/dumb-init pour des explications détaillées)
3.  `curl [localhost:9001](http://localhost:9001)` ne marche pas ⇒ Il faut préciser à docker de router un port de notre machine vers un port du container

Arrêtons le container puis relançons-le de façon détachée en précisant le mapping des ports

`docker run --name test-server-node --init --rm -d -p 9001:9001 server-node` (_NB :_ ou `--expose 9001`)

On peut maintenant vérifier que le curl fonctionne bien.

Créons maintenant une deuxième application simple qui va appeler le serveur toutes les 3 secondes :

```docker
# dockerfile.pinger
FROM alpine
RUN apk add curl
CMD while true; do curl -s http://localhost:9001; sleep 3; done;
```

Construisons l’image `docker build -t pinger -f dockerfile.pinger .`

Lançons la deuxième application `docker run --name test-pinger --rm pinger`

Il ne se passe rien, la communication entre les deux applications ne fonctionne pas.

On va donc créer un network : `docker network create mynetwork`

Et adapter un peu le dockerfile du pinger

```docker
# dockerfile.pinger
FROM alpine
RUN apk add curl
CMD while true; do curl -s http://test-server-node:9001; sleep 3; done;
```

_NB :_ C’est l’adresse qui a changé, on appelle l’autre application avec le nom du container, c’est pratique.

On peut arrêter les deux containers (avec `docker stop test-server-node` pour le serveur)

Reconstruire l’image du pinger `docker build -t pinger -f dockerfile.pinger .`

Puis lancer chaque container dans le network :

- `docker run --name test-server-node --init --rm -d --network mynetwork server-node` (on oublie volontairement le port ici)
- `docker run --rm --network mynetwork --name test-pinger pinger`

Maintenant, ça fonctionne.

Une autre force des networks, c’est l’isolation des containers par network.

Créons un deuxième network : `docker network create mynetwork2`

Modifions un tout petit peu le fichier `index.js`

```jsx
// ...
res.end('My second server!\n');
// ...
```

Créons une deuxième image : `docker build -t server-node-2 -f dockerfile.server .`

_NB :_ On ne peut pas lancer deux containers avec le même nom mais on a vu que l’interaction entre les deux dépendait du nom, donc on va devoir recourir à une astuce

`docker run --rm --network mynetwork -d --network-alias test-server-node server-node`

`docker run --rm --network mynetwork2 -d --network-alias test-server-node server-node-2`

`docker run --rm --network mynetwork pinger`

`docker run --rm --network mynetwork2 pinger` (dans une autre console)

Les deux serveurs sont accessibles dans leurs réseaux sur `test-server-node:9001`

Les deux pingers appellent bien chacun leur serveur, c’est magique.

Arrêtons tous nos containers.

## 7. Les volumes

**Où sont stockées les données ?**

Connectons-nous à un container en le lançant

`docker run -it pinger sh`

Écrivons de la donnée quelque part

`echo test > test.txt`

Arrêtons le container et relançons-le

`docker start confident_feistel`

reconnectons-nous avec

`docker exec -it confident_feistel sh`

Le fichier a disparu.

Arrêtons le container.

Créons un volume nommé à la volée

`docker run -it -v my-volume:/my-data pinger sh`

Si on fait `ls -al` à la racine, on découvre un nouveau répertoire qui s’appelle `my-data`

Écrivons de nouveau dans un fichier mais dans le répertoire `echo test > /my-data/test.txt` puis quittons le terminal (ctrl + d)

Relançons notre container : `docker run -it -v my-volume:/my-data pinger sh`

On peut voir que la donnée est gardée : `cat /my-data/test.txt`

On peut créer le volume en avance de phase avec `docker volume create my-volume`

Aujourd’hui les données de ce volume sont à un endroit géré par docker (`docker inspect my-volume`) mais on peut décider de l’endroit où sont stockées ces informations, on parle de dossier mappé.

## 8. Les registries

Ils servent à héberger les images (pas les dockerfiles, ni les containers)

On utilise Gitlab chez nous.

Pour se connecter, il faut créer un token Gitlab qui autorise les actions sur les registries au préalable.

_Pour se login :_ `docker login [registry.gitlab.com](http://registry.gitlab.com/) -u <username> -p <token>`

Une fois loggué, on peut builder une image avec un tag sous la forme `<registry-url>/[…folders]/<image-name>:<version>`

_Exemple :_ `docker build -t [registry.gitlab.com/<votre organisation>/docker-images/pinger:latest](http://registry.gitlab.com/<votre organisation>/docker-images/node-e2e-cypress:latest) -f dockerfile.pinger .`

On push l’image avec le tag créé à l’instant `docker push [registry.gitlab.com/<votre organisation>/docker-images/pinger:latest](http://registry.gitlab.com/<votre organisation>/docker-images/node-e2e-cypress:latest)` (docker se base sur le nom du tag pour savoir où push)

## 9. Le multi-stage

Permet de créer des contextes isolés (test puis build par exemple) . En gros, on peut voir ça comme plusieurs images successives dans un même dockerfile.

Créons un nouveau dockerfile :

```docker
# dockerfile.multi
FROM alpine as first

WORKDIR /app
RUN echo foo > /app/foo.txt
RUN echo bar > /app/bar.txt

FROM alpine

WORKDIR /app
COPY --from=first /app/bar.txt /app/baz.txt
CMD ls -al /app && \
cat /app/baz.txt
```

Construisons l’image et lançons un conteneur :

- `docker build -t test-multi -f dockerfile.multi .`
- `docker run --rm test-multi`

Comme on peut le voir, le résultat est

```bash
total 12
drwxr-xr-x    1 root     root          4096 Jan 20 08:32 .
drwxr-xr-x    1 root     root          4096 Jan 20 08:33 ..
-rw-r--r--    1 root     root             4 Jan 20 08:32 baz.txt
> bar
```

Docker a donc construit une première image `first` dans laquelle on a créé deux fichiers dans /app.

Petite astuce sur le multi-stage, on peut choisir la cible de build.

Essayons avec `docker build -t test-multi -f dockerfile.multi --target first .`

Lançons le container en mode interactif : `docker run --rm -it test-multi sh`

```bash
/app # ls -al
total 16
drwxr-xr-x    1 root     root          4096 Jan 20 08:32 .
drwxr-xr-x    1 root     root          4096 Jan 20 09:23 ..
-rw-r--r--    1 root     root             4 Jan 20 08:32 bar.txt
-rw-r--r--    1 root     root             4 Jan 20 08:31 foo.txt
```

On voit que l’image construite s’est arrêté au premier stage.

NB : Comme on peut chaîner les stages entre eux, on peut faire un genre de graph de liaison entre les stages, ça permet des choses un peu poussées (exemple dans Falco)

## 10. Docker-compose

On a pu voir qu’on finit par devoir passer pas mal d’arguments à docker dans certains cas et ça peut vite devenir beaucoup plus.

Au-delà du problème de lisibilité, du problème de typo possible, etc., on se rend compte aussi que ces arguments ne sont pas commités nulle part et qu’il peut être difficile de savoir avec quels arguments le container a été instancié.

Autre problème, il faut se rappeler de créer tous les éléments dans le bon ordre (network d’abord, puis le serveur, puis le pinger)

Pour répondre à un certain nombre de ces problématiques, il existe le `docker-compose.yml`

Créons un exemple :

```yaml
version: '3'

services:
  test-server-node-1:
    image: server-node
    networks:
      mynetwork:
        aliases:
          - test-server-node

  test-pinger-1:
    image: pinger
    depends_on:
      - test-server-node-1
    networks:
      mynetwork:
        aliases:
          - test-pinger

  test-server-node-2:
    image: server-node-2
    networks:
      mynetwork2:
        aliases:
          - test-server-node

  test-pinger-2:
    image: pinger
    depends_on:
      - test-server-node-2
    networks:
      mynetwork2:
        aliases:
          - test-pinger

networks:
  mynetwork:
  mynetwork2:
```

On peut lancer toutes les applications, dans leur network, dans le bon ordre et voir les logs des deux pingers en une seule commande : `docker-compose up`

On peut arrêter les containers avec CTRL+C et les relancer en mode détaché avec `-d` : `docker-compose up -d`

Pour retrouver les logs des applications, il suffit de taper `docker-compose logs -f`

Pour arrêter des containers lancés en mode détaché, il faut taper `docker-compose down`

On peut se limiter au build des images avec `docker-compose build`

Toutes ces commandes (sauf `down`) fonctionnent aussi en ciblant un ou plusieurs services :

`docker-compose up test-server-node-2` par exemple.

Pour supprimer un seul container, on peut utiliser `docker-compose rm -y test-server-node-2`

On peut passer quelques options à docker-compose comme `--build` pour forcer la reconstruction de l’image au préalable `docker-compose up --build test-server-node-2`

Par contre, il faudra pour cela préciser les options nécessaires au build (comme le lien vers le dockerfile par exemple) dans le `docker-compose.yml`. (Exemples dans Falco.)

_Une petite astuce :_ Par défaut, docker va préfixer les containers dans le compose par le nom du répertoire qui héberge le fichier `docker-compose.yml`, on peut spécifier un nom en passant l’option `--project` (ou `-p`) `docker-compose -p test-branch1 up`
