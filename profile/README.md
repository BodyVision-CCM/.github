<h1 align="center">BodyVision</h1>

<p align="center">
  <strong>Analyse biomécanique Edge &amp; Cloud</strong><br/>
  Projet thématique — Master 2 Cloud Computing &amp; Mobility, INSSET
</p>

---

BodyVision est une application Android qui analyse la posture pendant les exercices de
musculation et prévient l'utilisateur lorsque son mouvement l'expose à une blessure.

La détection de pose s'exécute **sur le téléphone** : l'alerte s'affiche pendant la série,
sans réseau et sans latence. Seules les coordonnées articulaires partent ensuite vers le
Cloud, en différé, pour construire l'historique et la progression. **Aucune vidéo ne quitte
l'appareil.**

L'intérêt du sujet est qu'il force les deux moitiés du master à coexister : du *Machine
Learning* embarqué sur mobile d'un côté, une architecture microservices sur Kubernetes de
l'autre, avec une contrainte de bande passante qui conditionne les deux.

|  |  |
|---|---|
| **Formation** | Master 2 Cloud Computing & Mobility — INSSET, Saint-Quentin |
| **Période** | septembre 2026 → juin 2027 |
| **Méthode** | Scrum, sprints de 2 semaines, approche API-First |
| **Soutenance** | juin 2027 |

## L'équipe

| Membre | Rôle |
|---|---|
| **Jayson Poncey-Valdemar** | Lead architecte, application mobile Edge, gestion du projet |
| **Raphaël Verchain** | DevSecOps, infrastructure et SRE |
| **Brice Ngongang** | Backend et données |

Rituels le dimanche : point d'étape à mi-sprint, rétrospective et planification en fin de
sprint. Le reste de la collaboration est asynchrone, sans blocage mutuel.

## Les dépôts

| Dépôt | Contenu |
|---|---|
| [`bodyvision-contracts`](https://github.com/BodyVision-CCM/bodyvision-contracts) | Contrat OpenAPI — la source de vérité des interfaces |
| [`bodyvision-mobile`](https://github.com/BodyVision-CCM/bodyvision-mobile) | Application Android — Kotlin, CameraX, ML Kit |
| [`bodyvision-user-service`](https://github.com/BodyVision-CCM/bodyvision-user-service) | Service Utilisateurs — Python, FastAPI |
| [`bodyvision-analytics-service`](https://github.com/BodyVision-CCM/bodyvision-analytics-service) | Service Analytique — Python, API et worker |
| [`bodyvision-infrastructure`](https://github.com/BodyVision-CCM/bodyvision-infrastructure) | Terraform — VPC, GKE, IAM, Artifact Registry |

Le service Ingestion, écrit en Go, aura son dépôt à l'ouverture de son développement.

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
        gw["<b>Gateway</b><br/>Gateway API — gérée par GKE<br/><br/>Entrée unique, TLS, routage"]

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

## Planning prévisionnel

Dix-sept sprints de deux semaines, de septembre 2026 à juin 2027. Les sprints en rouge
portent un jalon : ce sont les quatre échéances qui ne peuvent pas glisser. La barre
verticale marque la date du jour, et l'interruption entre décembre et janvier correspond
aux congés universitaires.

Ce planning est le reflet du tableau Jira de l'équipe, qui reste la référence.

```mermaid
gantt
    title BodyVision — planning prévisionnel (sprints de 2 semaines)
    dateFormat YYYY-MM-DD
    axisFormat %b
    tickInterval 1month
    todayMarker stroke:#186DB8,stroke-width:3px,opacity:0.6

    section Socle
    S1 Cadrage et contrats            :done,   s1,  2026-09-22, 2026-10-05
    S2 Socle doc et reseau            :done,   s2,  2026-10-05, 2026-10-18
    S3 Captation mobile et GKE        :active, s3,  2026-10-20, 2026-11-02

    section Jalon 1
    S4 Squelette temps reel           :crit,   s4,  2026-11-03, 2026-11-16
    Jalon 1 - PoC technique           :milestone, j1, 2026-11-16, 0d

    section Industrialisation
    S5 CI et perf 30 FPS              :        s5,  2026-11-17, 2026-11-30
    S6 CD, RabbitMQ et ingestion      :        s6,  2026-12-01, 2026-12-14
    S7 Gateway et envoi des seances   :        s7,  2027-01-04, 2027-01-17

    section Jalon 2
    S8 Observabilite                  :crit,   s8,  2027-01-18, 2027-01-31
    Jalon 2 - Ingestion bout en bout  :milestone, j2, 2027-01-31, 0d

    section Coeur metier
    S9 Angles et alerte posturale     :        s9,  2027-02-01, 2027-02-14
    S10 Historique et dashboards      :        s10, 2027-02-15, 2027-02-28
    S11 Erreurs de posture et SLO     :        s11, 2027-03-01, 2027-03-14

    section Jalon 3
    S12 Statistiques                  :crit,   s12, 2027-03-15, 2027-03-28
    Jalon 3 - MVP metier complet      :milestone, j3, 2027-03-28, 0d

    section Robustesse
    S13 Charge et trajectoires        :        s13, 2027-03-29, 2027-04-11
    S14 Chaos - preparation           :        s14, 2027-04-12, 2027-04-25
    S15 Chaos - execution             :        s15, 2027-04-26, 2027-05-09

    section Rendu
    S16 Stabilisation et doc          :        s16, 2027-05-10, 2027-05-23
    S17 Soutenance                    :crit,   s17, 2027-05-24, 2027-06-06
    Soutenance                        :milestone, sout, 2027-06-06, 0d
```

### Les quatre jalons

| Jalon | Échéance | Ce qui doit être démontrable |
|---|---|---|
| **Jalon 1** — PoC technique | mi-novembre 2026 | Squelette filaire affiché en temps réel sur le téléphone, et extraction du JSON de télémétrie |
| **Jalon 2** — Ingestion bout en bout | fin janvier 2027 | Une séance filmée traverse toute la chaîne jusqu'à la base, avec OpenTelemetry et Grafana opérationnels |
| **Jalon 3** — MVP métier | fin mars 2027 | Détection des erreurs de posture, tableau de bord de progression et alerting |
| **Soutenance** | juin 2027 | Démonstration en direct d'une panne et de l'auto-guérison |

Les trois sprints qui suivent le jalon 3 n'ajoutent aucune fonctionnalité : ils servent à
prouver que le système tient. Tests de charge et validation de l'autoscaling, puis pannes
volontaires pour vérifier l'auto-guérison et l'idempotence du consumer.

## Comment on travaille

**API-First.** Le contrat OpenAPI vit dans son propre dépôt et sa branche `main` fait foi.
Il précède l'implémentation, ce qui permet au mobile de développer contre un serveur de
simulation avant que le backend existe. Il est validé automatiquement par Spectral.

**Trunk-Based Development.** Branche `main` protégée, revue croisée obligatoire avant toute
fusion. Aucune branche de longue durée.

**Tout en Infrastructure as Code.** L'environnement GCP est décrit en Terraform et
entièrement recréable. Il est volontairement éphémère : créé avant une session de test,
détruit après, ce qui est la principale mesure de maîtrise des coûts.

**Sécurité dans la chaîne de build.** Analyse statique et scan de vulnérabilités des images
avec Trivy, directement dans les pipelines.

---

<p align="center"><sub>Projet académique — UE Projet Thématique, M2 CCM, INSSET (Université de Picardie Jules Verne)</sub></p>
