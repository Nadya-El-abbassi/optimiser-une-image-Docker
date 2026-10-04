# adult-serve — Optimisation d'une image Docker

Projet d'optimisation d'une image Docker pour une application de prédiction Python.

## Objectifs

* Réduire la taille de l'image Docker.
* Améliorer le temps de reconstruction.
* Optimiser l'utilisation du cache Docker.
* Limiter les fichiers inutiles.
* Utiliser un utilisateur non-root.
* Ajouter un `HEALTHCHECK`.
* Analyser les couches avec `dive`.

## Optimisations et résultats

| # | Optimisation                              |        Taille | Build à froid | Rebuild après modification |
| - | ----------------------------------------- | ------------: | ------------: | -------------------------: |
| 0 | Version naïve (référence)                 |        644 Mo |         329 s |                      348 s |
| 1 | Base `python:3.11-slim`                   |        283 Mo |        1386 s |                      442 s |
| 2 | `COPY requirements.txt` avant `COPY src/` |        172 Mo |         219 s |                       15 s |
| 3 | `.dockerignore` complet                   |        172 Mo |         149 s |                        4 s |
| 4 | `pip install --no-cache-dir`              | Déjà appliqué |             — |                          — |
| 5 | Construction multi-étapes                 |        168 Mo |         212 s |                        4 s |
| 6 | Non-root + `HEALTHCHECK`                  |        168 Mo |         143 s |                        6 s |

## Bilan

| Mesure                     | Départ | Arrivée | Variation |
| -------------------------- | -----: | ------: | --------: |
| Taille                     | 644 Mo |  168 Mo | **−74 %** |
| Rebuild après modification |  348 s |     6 s | **−98 %** |
| Build à froid              |  329 s |   143 s | **−57 %** |

Le plus gros gain de taille vient de `python:3.11-slim`.

Le plus gros gain de temps de reconstruction vient du placement de `COPY requirements.txt` avant `COPY src/`, ce qui permet de conserver l'installation des dépendances dans le cache Docker.

> Les temps de build à froid peuvent varier selon le réseau et le téléchargement des dépendances.

## Analyse avec Dive

L'analyse de l'image finale `adult-serve:opt` avec `dive` donne :

* **Score d'efficacité : 97 %**
* **Espace potentiellement gaspillé : environ 22 Mo**
* **Taille décompressée analysée : environ 519 Mo**

La couche la plus lourde est :

```text
COPY /install /usr/local
```

avec environ **386 Mo**.

Cette couche contient principalement les dépendances Python nécessaires à l'application. Son poids est donc principalement utile.

Le principal gaspillage identifié concerne `libcrypto.so.3`, avec environ **13 Mo**, provenant de l'image de base.

## Benchmark

Les mesures sont automatisées avec le script `benchmark.sh`.

Exécution :

```bash
bash benchmark.sh opt
```

Le script mesure automatiquement :

* la taille de l'image ;
* le temps de build à froid ;
* le temps de rebuild après modification du code.

## Structure du projet

```text
adult-serve/
├── Dockerfile.serve
├── benchmark.sh
├── requirements.txt
├── src/
│   └── predict.py
└── README.md
```

## Image Docker

L'image optimisée est publiée sur **GitHub Container Registry (GHCR)**.

Package Docker :

https://github.com/users/Nadya-El-abbassi/packages/container/package/adult-serve

Pour récupérer l'image :

```bash
docker pull ghcr.io/nadya-el-abbassi/adult-serve:latest
```

Pour l'exécuter :

```bash
docker run ghcr.io/nadya-el-abbassi/adult-serve:latest
```

## Conclusion

Les optimisations permettent de réduire fortement la taille de l'image et le temps de reconstruction.

La taille passe de **644 Mo à 168 Mo**, tandis que le temps de rebuild passe de **348 s à 6 s**.
