## Architecture

BodyVision analyse la posture **sur le téléphone** et n'envoie au Cloud que des
coordonnées articulaires, jamais de vidéo. Le Cloud conserve l'historique, calcule
la progression et absorbe les rafales d'envoi.

### Niveau 1 — Contexte

```mermaid
flowchart TB
    athlete["Athlète<br/>— Personne —<br/>Filme les séries qu'il choisit,<br/>lit ses alertes et sa progression"]

    bv["Système BodyVision<br/>— Système logiciel —<br/>Analyse biomécanique : détection de posture<br/>sur le téléphone, historique et progression<br/>dans le Cloud"]

    mlkit["Google ML Kit Pose Detection<br/>— Système externe —<br/>Modèle de détection de pose<br/>exécuté sur l'appareil, hors ligne"]

    gcp["Google Cloud Platform<br/>— Système externe —<br/>Hébergement : GKE, PostgreSQL,<br/>Cloud Storage, observabilité"]

    athlete -->|"Filme une série,<br/>consulte ses résultats"| bv
    bv -->|"Alerte posturale immédiate,<br/>rapport de série, progression"| athlete
    bv -->|"Demande les 33 points du corps,<br/>image par image, sans réseau"| mlkit
    bv -->|"S'exécute sur"| gcp

    classDef personne fill:#95DB3B,stroke:#5e8c22,stroke-width:2px,color:#13240a
    classDef systeme fill:#186DB8,stroke:#0d3f6b,stroke-width:2px,color:#ffffff
    classDef externe fill:#8899a6,stroke:#5a6b77,stroke-width:2px,color:#ffffff

    class athlete personne
    class bv systeme
    class mlkit,gcp externe
```

### Niveau 2 — Conteneurs

Trois images Docker, quatre déploiements : l'API et le worker du service Analytique
partagent la même image et ne diffèrent que par leur point d'entrée.

```mermaid
flowchart TB
    athlete["Athlète"]

    subgraph edge["EDGE — Téléphone Android"]
        app["<b>Application mobile</b><br/>Kotlin · CameraX · ML Kit<br/><br/>Pose à 30 FPS et alerte posturale<br/>calculées hors ligne.<br/>Sous-échantillonne à 10 FPS,<br/>envoie les séries en différé."]
    end

    subgraph cloud["CLOUD — GKE · europe-west9"]
        gw["<b>Gateway</b><br/>Nginx Ingress<br/><br/>Entrée unique, TLS, routage"]

        users["<b>Service Utilisateurs</b><br/>Python · FastAPI<br/><br/>Comptes, JWT RS256, JWKS,<br/>profil et morphologie<br/><i>HPA — CPU et mémoire</i>"]

        ing["<b>Service Ingestion</b><br/>Go<br/><br/>Un lot = une série. Répond 202,<br/>dépose le brut, publie dans la file.<br/>Idempotent par batchId<br/><i>HPA — CPU et mémoire</i>"]

        subgraph img3["Une seule image, deux déploiements"]
            api["<b>Analytique — API</b><br/>Python<br/><br/>Rapport de série,<br/>historique, progression<br/><i>HPA</i>"]
            wrk["<b>Analytique — Worker</b><br/>Python<br/><br/>Angles, anomalies, agrégats<br/><i>KEDA — profondeur de la file</i>"]
        end

        mq[("<b>RabbitMQ</b><br/>File des lots")]
        db[("<b>PostgreSQL</b><br/>Une instance, deux schémas,<br/>un rôle SQL par schéma")]
        obs["<b>Observabilité — transverse</b><br/>Managed Prometheus · Grafana · Cloud Logging<br/><br/><i>Collecte les métriques, logs et traces<br/>des quatre déploiements ci-dessus</i>"]
    end

    gcs[("<b>Cloud Storage</b><br/>Télémétrie brute<br/>rétention 30 j")]

    athlete --> app
    app -->|"HTTPS · JSON gzip · différé<br/>jamais de vidéo"| gw

    gw --> users
    gw --> ing
    gw --> api

    ing -->|"dépose le brut"| gcs
    ing -->|"publie"| mq
    mq -->|"consomme"| wrk

    wrk -->|"angles, anomalies, agrégats<br/>schéma analytique"| db
    api -->|"lit<br/>schéma analytique"| db
    users -->|"comptes et profils<br/>schéma utilisateurs"| db

    wrk -->|"morphologie par appel d'API —<br/>aucune jointure entre schémas"| users

    classDef personne fill:#95DB3B,stroke:#5e8c22,stroke-width:2px,color:#13240a
    classDef conteneur fill:#186DB8,stroke:#0d3f6b,stroke-width:2px,color:#ffffff
    classDef stockage fill:#1f5f91,stroke:#0d3f6b,stroke-width:2px,color:#ffffff
    classDef plateforme fill:#8899a6,stroke:#5a6b77,stroke-width:1px,color:#ffffff

    class athlete personne
    class app,gw,users,ing,api,wrk conteneur
    class mq,db,gcs stockage
    class obs plateforme
```

### Ce que le diagramme fige

| Décision | Où elle se lit |
|---|---|
| Aucune vidéo ne quitte le téléphone | Arête `Application mobile → Gateway` |
| L'alerte posturale est calculée hors ligne | Conteneur `Application mobile` |
| 30 FPS pour l'alerte, 10 FPS vers le Cloud | Conteneur `Application mobile` |
| Ingestion en Go, isolée du reste | Conteneur `Service Ingestion` |
| 3 images, 4 déploiements | Cadre `Une seule image, deux déploiements` |
| Ingestion asynchrone, réponse `202` | `Ingestion → RabbitMQ → Worker` |
| Rejeu sans doublon | `Idempotent par batchId` |
| La télémétrie brute n'est jamais en base | `Ingestion → Cloud Storage`, rétention 30 j |
| Une instance PostgreSQL, un schéma par service | Conteneur `PostgreSQL` |
| Aucune jointure entre schémas | Arête `Worker → Service Utilisateurs` |
| HPA sur les API, KEDA sur le worker | Mentions `HPA` et `KEDA` dans les conteneurs |
