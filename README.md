# \# deployment-cd — Déploiement de SunuSanté

# 

# Dépôt de \*\*déploiement continu (CD)\*\* de \*\*SunuSanté\*\*, plateforme de prise de rendez-vous médicaux pour plusieurs établissements de santé au Sénégal (Master 2 SI-SR, UADB).

# 

# Ce dépôt ne contient \*\*aucun code applicatif\*\* : il décrit comment \*\*assembler et faire tourner\*\* l'application à partir des images publiées sur Docker Hub par les pipelines CI.

# 

# \## Architecture

# 

# ```

# &#x20;             Navigateur

# &#x20;                 │  http://localhost  (port 80)

# &#x20;                 ▼

# &#x20;       ┌───────────────────┐

# &#x20;       │     frontend      │  Nginx + Angular

# &#x20;       │     (port 80)     │  /api/\*  ──►  backend

# &#x20;       └─────────┬─────────┘

# &#x20;                 │  réseau interne Docker

# &#x20;       ┌─────────▼─────────┐

# &#x20;       │    backend  x2    │  Spring Boot (port 8080, interne)

# &#x20;       └─────────┬─────────┘

# &#x20;                 │

# &#x20;       ┌─────────▼─────────┐

# &#x20;       │        db         │  PostgreSQL 16 (port 5432, interne)

# &#x20;       │   volume pgdata   │

# &#x20;       └───────────────────┘

# ```

# 

# Seul le \*\*port 80\*\* (Nginx) est ouvert vers l'extérieur. Le backend et la base de données restent accessibles uniquement à l'intérieur du réseau Docker.

# 

# \## Dépôts du projet

# 

# | Dépôt | Rôle | Image Docker Hub |

# | --- | --- | --- |

# | \[backend-ci](https://github.com/Thierno-Souleymane-diallo/backend-ci) | API Spring Boot + pipeline CI | \[sunusante-backend](https://hub.docker.com/r/thiernosouleymanediallo/sunusante-backend) |

# | \[frontend-ci](https://github.com/Thierno-Souleymane-diallo/frontend-ci) | Application Angular + pipeline CI | \[sunusante-frontend](https://hub.docker.com/r/thiernosouleymanediallo/sunusante-frontend) |

# | deployment-cd (ce dépôt) | Déploiement Docker Compose + pipeline CD | — |

# 

# \## Contenu du dépôt

# 

# | Fichier | Rôle |

# | --- | --- |

# | `docker-compose.yml` | Décrit les 3 services, leur réseau, le volume de la base et les vérifications de santé |

# | `.env` | Variables \*\*non secrètes\*\* : nom de la base, utilisateur, versions des images |

# | `Jenkinsfile` | Pipeline CD : récupération des images, déploiement sans coupure, vérification |

# | `README.md` | Cette documentation |

# 

# \## Configuration

# 

# | Variable | Où elle est définie | Valeur par défaut |

# | --- | --- | --- |

# | `DB\_NAME` | `.env` | `sunusante` |

# | `DB\_USER` | `.env` | `sunu` |

# | `BACKEND\_TAG` | `.env`, ou paramètre Jenkins | `latest` |

# | `FRONTEND\_TAG` | `.env`, ou paramètre Jenkins | `latest` |

# | `DB\_PASSWORD` | \*\*coffre-fort de Jenkins\*\* (identifiant `db-password`) | aucune : obligatoire |

# 

# \*\*Aucun secret n'est stocké dans ce dépôt.\*\* Le mot de passe de la base est fourni par Jenkins au moment du déploiement. S'il manque, Docker Compose refuse de démarrer (`${DB\_PASSWORD:?...}`).

# 

# \## Le pipeline CD

# 

# \### Déclenchement

# 

# Le pipeline `sunusante-cd` se lance de trois façons :

# 1\. \*\*Par les pipelines CI\*\* : à la fin de `backend-ci` ou `frontend-ci`, avec la version de l'image qui vient d'être publiée (`BACKEND\_TAG` ou `FRONTEND\_TAG`).

# 2\. \*\*Par un `git push` sur ce dépôt\*\* : Jenkins vérifie le dépôt toutes les 2 minutes (\*polling\*).

# 3\. \*\*À la main\*\* dans Jenkins, avec le bouton \*Lancer un build avec des paramètres\*.

# 

# \### Étapes

# 

# | Étape | Rôle |

# | --- | --- |

# | Vérifier la configuration | `docker compose config` : le fichier est valide et toutes les variables sont fournies |

# | Récupérer les images | `docker compose pull` : téléchargement des versions demandées depuis Docker Hub |

# | Base de données | démarrage de PostgreSQL et attente de son état \*healthy\* |

# | Backend sans coupure | mise à jour progressive du backend (voir ci-dessous) |

# | Frontend | mise à jour de Nginx et attente de son état \*healthy\* |

# | Vérifier de bout en bout | appel de `/api/etablissements` à travers Nginx, le backend et la base |

# 

# \## Mise à jour sans coupure (rolling update)

# 

# Le backend tourne en \*\*2 copies\*\*. Lors d'un déploiement :

# 1\. Deux \*\*nouvelles\*\* copies démarrent \*\*à côté\*\* des anciennes.

# 2\. Le pipeline attend qu'elles soient \*healthy\* (`/health` répond).

# 3\. Il attend 15 secondes que Nginx les découvre.

# 4\. Les \*\*anciennes\*\* copies sont arrêtées.

# 

# À tout moment, au moins deux copies répondent : les utilisateurs ne voient aucune interruption. Si les nouvelles copies ne deviennent jamais \*healthy\*, elles sont supprimées et \*\*les anciennes continuent de servir\*\*.

# 

# \*\*Limite connue :\*\* le frontend n'a qu'une copie (il occupe le port 80). Une mise à jour de son image provoque une interruption d'environ une seconde.

# 

# \## Déployer à la main (sans Jenkins)

# 

# Dans PowerShell, depuis ce dossier :

# 

# ```powershell

# $env:DB\_PASSWORD = "mot\_de\_passe\_de\_la\_base"

# $env:COMPOSE\_PROJECT\_NAME = "sunusante"

# docker compose pull

# docker compose up -d

# docker compose ps

# ```

# 

# L'application est alors accessible sur \*\*http://localhost\*\*. Ce déploiement manuel ne fait pas de mise à jour sans coupure : il est réservé aux tests.

# 

# \## Revenir à une version précédente

# 

# Dans Jenkins, ouvrir `sunusante-cd`, cliquer sur \*\*Lancer un build avec des paramètres\*\*, puis saisir le numéro de la version voulue dans `BACKEND\_TAG` ou `FRONTEND\_TAG`. Toutes les versions restent disponibles sur Docker Hub.

# 

# \## Persistance des données

# 

# Les données de PostgreSQL sont rangées dans le volume \*\*`sunusante\_pgdata`\*\*. Elles survivent aux redéploiements et aux redémarrages : le conteneur de la base n'est jamais recréé tant que sa configuration ne change pas.

# 

# ⚠️ Ne jamais utiliser `docker compose down -v` : l'option `-v` supprime le volume, et donc toutes les données.

