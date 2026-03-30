# Compte rendu TP

L’image construite avec le Dockerfile classique a une taille de 1.12 Go, tandis que l’image en multi-stage ne fait que 202 Mo, soit une réduction très significative (environ 80 % plus légère).

Cette différence s’explique d’abord par le fait que le multi-stage n’inclut pas les outils de build dans l’image finale. Dans le Dockerfile classique, on retrouve toutes les dépendances nécessaires à la compilation (npm, dépendances complètes, etc.), ce qui alourdit fortement l’image. À l’inverse, le multi-stage construit l’application dans une première étape puis ne copie que les fichiers nécessaires à l’exécution (comme server.js et le dossier public).

Le fichier .dockerignore permet d’exclure certains fichiers du contexte de build Docker (comme node_modules, .git ou les logs). Cela améliore les performances en réduisant la quantité de données envoyées au daemon Docker lors du build, ce qui accélère la construction de l’image.

De plus, il évite d’inclure accidentellement des fichiers inutiles ou volumineux dans l’image (notamment via un COPY . .), ce qui peut indirectement contribuer à réduire sa taille et à améliorer sa sécurité.

Ensuite, le multi-stage permet de ne conserver que les dépendances de production. On le voit avec la commande npm ci --omit=dev, qui évite d’embarquer les dépendances de développement inutiles en production, réduisant encore la taille finale.

En production, une amélioration pertinente serait d’utiliser une image de base plus légère, comme une version Alpine de Node.js (par exemple node:alpine), afin de réduire encore davantage la taille de l’image et la surface d’attaque.