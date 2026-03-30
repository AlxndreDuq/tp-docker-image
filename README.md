# Questions

**1. Quel est le poids de tp-docker:classic ?**

 1.12GB

**2. Qu’est-ce qui “gonfle” potentiellement l’image ici ?**
* Les dépendances complètes (node_modules) : avec npm ci, toutes les dépendances (y compris celles de développement) sont installées et conservées dans l’image.
* Les outils de build et l’environnement complet : l’image contient tout ce qui est nécessaire pour construire l’application (Node.js complet, outils système, etc.), alors que ces éléments ne sont pas utiles à l’exécution.
* Le COPY . . : cette instruction copie l’intégralité du projet dans l’image, y compris des fichiers inutiles (logs, .git, etc.) si aucun .dockerignore n’est utilisé.

**3. Pourquoi node_modules dans le .dockerignore est une bonne idée ?**

Exclure le dossier node_modules via le fichier .dockerignore est une bonne pratique car il est souvent très volumineux et dépend de l’environnement local (système d’exploitation, architecture). Le copier dans l’image ralentirait fortement le build et pourrait entraîner des incompatibilités. En le supprimant du contexte de build, on force Docker à réinstaller proprement les dépendances avec npm ci, garantissant ainsi un environnement reproductible, plus fiable et optimisé.

**4. Quel est l’écart de poids entre classic et multistage ?**

L’image classic fait 1.12 Go et l’image multistage 202 Mo. L’écart est donc d’environ 918 Mo

**5. Qu’est-ce qui a été supprimé (ou non copié) dans le stage runtime ?**

Dans le stage runtime, seuls les éléments nécessaires à l’exécution sont conservés. Ont été supprimés (ou non copiés) :

* Les outils de build (npm complet, outils système, etc.)
* Les fichiers sources inutiles (tout le projet via COPY . .)
* Les dépendances de développement
* Les fichiers temporaires et de configuration (.git, logs, etc.)

Seuls des éléments essentiels comme server.js, public et les dépendances de production sont gardés.

**6. Est-ce que ça réduit aussi la surface d’attaque ? Pourquoi ?**

Oui, le multi-stage réduit la surface d’attaque.

Moins il y a de logiciels et de fichiers dans une image, moins il y a de potentielles vulnérabilités exploitables. En supprimant les outils de build, les dépendances inutiles et les fichiers superflus, on réduit le nombre de composants susceptibles de contenir des failles.

De plus, une image plus minimaliste limite les possibilités pour un attaquant (moins de commandes disponibles, moins de bibliothèques exploitables), ce qui améliore globalement la sécurité.

**7. Quelles étapes créent les plus gros layers ?**

D’après le docker history, les couches les plus lourdes proviennent principalement :

Des couches système de l’image de base (Debian) avec plusieurs apt-get -> jusqu’à 588 MB, 177 MB, 117 MB
De l’installation de Node.js et ses dépendances -> environ 160 MB
De l’installation des dépendances npm (npm ci)
-> plusieurs dizaines de MB

Ces étapes “gonflent” fortement l’image car elles ajoutent :

des paquets système
des binaires
des bibliothèques complètes

À l’inverse, les étapes comme COPY du code ou npm run build sont relativement légères.

**8. Comment le fait de copier package\*.json avant le reste aide le cache Docker ?**

Copier package*.json avant le reste permet d’optimiser le cache Docker.

Docker fonctionne par couches : chaque instruction est mise en cache tant que ses entrées ne changent pas.
En copiant uniquement package.json et package-lock.json en premier, puis en exécutant npm ci :

Si les dépendances ne changent pas, cette couche est réutilisée depuis le cache
Même si le code source change ensuite (COPY . .), Docker n’a pas besoin de réinstaller les dépendances.


# Compte rendu TP

L’image construite avec le Dockerfile classique a une taille de 1.12 Go, tandis que l’image en multi-stage ne fait que 202 Mo, soit une réduction très significative (environ 80 % plus légère).

Cette différence s’explique d’abord par le fait que le multi-stage n’inclut pas les outils de build dans l’image finale. Dans le Dockerfile classique, on retrouve toutes les dépendances nécessaires à la compilation (npm, dépendances complètes, etc.), ce qui alourdit fortement l’image. À l’inverse, le multi-stage construit l’application dans une première étape puis ne copie que les fichiers nécessaires à l’exécution (comme server.js et le dossier public).

Le fichier .dockerignore permet d’exclure certains fichiers du contexte de build Docker (comme node_modules, .git ou les logs). Cela améliore les performances en réduisant la quantité de données envoyées au daemon Docker lors du build, ce qui accélère la construction de l’image.

De plus, il évite d’inclure accidentellement des fichiers inutiles ou volumineux dans l’image (notamment via un COPY . .), ce qui peut indirectement contribuer à réduire sa taille et à améliorer sa sécurité.

Ensuite, le multi-stage permet de ne conserver que les dépendances de production. On le voit avec la commande npm ci --omit=dev, qui évite d’embarquer les dépendances de développement inutiles en production, réduisant encore la taille finale.

En production, une amélioration pertinente serait d’utiliser une image de base plus légère, comme une version Alpine de Node.js (par exemple node:alpine), afin de réduire encore davantage la taille de l’image et la surface d’attaque.