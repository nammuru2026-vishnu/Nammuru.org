# NammUru --- Minimal Solution Architecture

**Document purpose:** Provide a concise view of NammUru's application
services, runtime platform, infrastructure provisioning, and delivery
model.

**Architecture baseline:** This document reflects the repository and
deployment notes currently available for the project. Where
infrastructure documentation describes a Terraform skeleton or where
deployment details conflict, the distinction is called out rather than
assumed.

------------------------------------------------------------------------

## 1. Architecture at a glance

NammUru is a geospatial civic-governance platform. Residents and other
users interact through the web experience and supported channels.
Requests enter through the Kong edge, are routed to domain services, and
are supported by shared data and messaging infrastructure. The platform
runs on Kubernetes (OCI OKE for the current OCI deployment), with Helm
packaging and Argo CD GitOps delivery.

``` mermaid
flowchart TB
    R[Residents] --> UI[Resident Web / PWA]
    S[Staff] --> UI
    P[CSR / Partners] --> UI
    WA[WhatsApp] --> EDGE

    UI --> EDGE[Kong Edge]
    EDGE --> API[Domain Services]

    subgraph APP["NammUru application services"]
      ID[Identity]
      GEO[Geography]
      ASSET[Asset]
      ISSUE[Issue]
      BUD[Budget]
      SAH[Sahayoga]
      ML[ML]
      NOTIF[Notification]
      REP[Reporting]
      PET[Petition]
      HS[Ward Health Score]
      LIVE[Liveability]
      XRAY[X-Ray]
      BOT[Bot]
    end

    API --> ID
    API --> GEO
    API --> ASSET
    API --> ISSUE
    API --> BUD
    API --> SAH
    API --> ML
    API --> NOTIF
    API --> REP
    API --> PET
    API --> HS
    API --> LIVE
    API --> XRAY
    API --> BOT

    APP --> DATA[(Shared data and platform services)]
```

The diagram is a logical view. It does not imply that every service is
called on every request.

## 2. Key application services

  -----------------------------------------------------------------------
  Service / domain                    Responsibility
  ----------------------------------- -----------------------------------
  Identity                            User identity, authentication,
                                      roles, and access control

  Geography                           Administrative boundaries, wards,
                                      zones, and geospatial reference
                                      data

  Asset                               Civic asset registry and
                                      asset-related information

  Issue                               Citizen issue intake, lifecycle,
                                      assignment, and resolution

  Budget                              Budget and civic funding workflows

  Sahayoga                            CSR / partner participation and
                                      funding workflows

  ML                                  Image/video intelligence and
                                      model-serving workflows

  Notification                        User and workflow notifications

  Reporting                           Operational and governance reports

  Petition                            Petition creation and lifecycle

  Healthscore                         Ward Health Score and
                                      civic-performance signals

  Liveability                         Ward and neighbourhood liveability
                                      information

  X-Ray                               Civic data exploration and
                                      analytical views

  Bot                                 Conversational and channel
                                      integration workflows

  Web / edge                          Web experience and Kong edge
                                      workload, as configured in the
                                      deployment
  -----------------------------------------------------------------------

Resident, Staff, and CSR / Partner experiences are distinct user
journeys that use the shared Identity Service. They should not be
interpreted as separate identity systems.

## 3. Core request and data flows

### 3.1 Citizen issue reporting

``` mermaid
sequenceDiagram
    participant C as Resident
    participant W as Web / WhatsApp
    participant K as Kong Edge
    participant I as Identity
    participant IS as Issue Service
    participant G as Geography
    participant M as ML
    participant N as Notification
    participant D as Data Services

    C->>W: Submit issue, image, location or message
    W->>K: Send request
    K->>I: Validate identity / access where required
    K->>IS: Create or update issue
    IS->>G: Resolve ward and geographic context
    IS->>M: Request image/video classification when applicable
    IS->>D: Persist issue and related records
    IS->>N: Trigger workflow notification
    N-->>C: Acknowledgement / status update
```

### 3.2 Sahayoga funding workflow

``` mermaid
flowchart LR
    A[CSR / Partner] --> B[Sahayoga]
    B --> C[Budget and opportunity details]
    C --> D[Review / approval workflow]
    D --> E[Funding or participation record]
    E --> F[Execution and progress tracking]
    F --> G[Reporting and evidence]
```

### 3.3 Ward Health Score

``` mermaid
flowchart LR
    I[Issue data] --> H[Ward Health Score]
    R[Resolution time] --> H
    SLA[SLA compliance] --> H
    O[Reopen rate] --> H
    P[Population / ward context] --> H
    H --> V[Ward-level score and trends]
```

The score is intended to summarize civic performance from defined
signals. Its exact formula, weighting, and refresh cadence should be
taken from the current implementation specification.

## 4. Runtime and infrastructure architecture

The current OCI deployment uses Oracle Kubernetes Engine (OKE) as the
Kubernetes runtime. Application workloads are deployed in the `nammuru`
namespace; Argo CD runs in the `argocd` namespace.

``` mermaid
flowchart TB
    U[Users / WhatsApp] --> LB[OCI Load Balancer]
    LB --> K[Kong Edge]

    subgraph OKE["OCI Kubernetes Engine"]
      subgraph NS["Namespace: nammuru"]
        K --> WEB[Web experience]
        K --> SVC[Domain service Deployments]
        SVC --> JOB[Workers / scheduled workloads]
        SVC --> PG[PostgreSQL]
        SVC --> REDIS[Redis]
        SVC --> MQ[RabbitMQ]
        SVC --> OS[OpenSearch, where enabled]
      end
      subgraph GITOPS["Namespace: argocd"]
        ARGO[Argo CD]
      end
    end

    PG --> PVC[Persistent Volumes]
    OBJ[OCI Object Storage] --- SVC
```

### 4.1 Kubernetes / OKE

-   **Kubernetes runtime:** OCI OKE, in OCI region `ap-hyderabad-1`.
-   **Namespaces:** `nammuru` for application workloads and `argocd` for
    Argo CD.
-   **Workload types:** Kubernetes Deployments and Services for
    application components, with supporting jobs/workers where
    configured.
-   **Networking:** Application services are internal `ClusterIP`
    services. External application traffic enters through the OCI Load
    Balancer and Kong edge.
-   **Persistence:** Stateful data services use persistent storage as
    configured by their Helm charts and storage classes.
-   **Object storage:** OCI Object Storage is used for object/file
    storage integration where configured.

The current edge design uses Kong as the single application edge. The
web workload includes the Kong edge deployment/service configuration;
the OCI Load Balancer fronts that edge. Do not add a separate Nginx or
second Kong tier to this diagram unless the repository architecture
changes.

### 4.2 Shared platform services

The repository deployment notes identify the following shared platform
components:

-   PostgreSQL for relational application data.
-   PgBouncer for PostgreSQL connection pooling.
-   Redis for caching and related low-latency workloads.
-   RabbitMQ for asynchronous messaging.
-   OpenSearch where enabled by the active deployment configuration.
-   OCI Object Storage for object/file storage.

Treat the active Helm values and current deployment state as
authoritative for whether an optional component such as OpenSearch is
enabled in a given environment.

## 5. CI/CD and GitOps

GitHub Actions builds and publishes container images. It then updates
the image tag in the Git-managed Helm configuration. Argo CD observes
the Git repository and reconciles the Kubernetes environment to the
declared state.

``` mermaid
flowchart LR
    DEV[Developer commit] --> GIT[GitHub repository]
    GIT --> CI[GitHub Actions]
    CI --> TEST[Build and test]
    TEST --> IMG[Build container image]
    IMG --> GHCR[GitHub Container Registry]
    CI --> BUMP[Update image tag in Helm values]
    BUMP --> GIT
    GIT --> ARGO[Argo CD]
    ARGO --> HELM[Helm chart reconciliation]
    HELM --> OKE[OKE workloads]
    OKE --> VERIFY[Deployment / health verification]
```

### 5.1 GitHub Actions

The deployment workflow is defined under `.github/workflows/`. The
documented pipeline:

1.  Detects changed services or deployment targets.
2.  Builds and tests the relevant container images.
3.  Tags images with an immutable Git commit SHA.
4.  Publishes images to GitHub Container Registry (GHCR), under the
    NammUru image namespace.
5.  Updates the corresponding Helm `image.tag` value in Git.
6.  Leaves deployment reconciliation to Argo CD.

### 5.2 Helm

Helm charts are maintained under `infra/helm/charts/`.

Helm is the packaging and configuration layer for Kubernetes workloads.
Charts define, as applicable:

-   Deployments, Services, and workload configuration.
-   Container image repository and immutable image tag.
-   Environment-specific values and resource settings.
-   Persistent volume claims and storage settings.
-   Secrets/configuration references.
-   Worker and migration jobs.

Cloud-specific values files provide environment overlays where required.
Keep environment-specific configuration in values files rather than
duplicating chart templates.

### 5.3 Argo CD

Argo CD provides continuous delivery through GitOps.

-   The root application (`nammuru-root`) follows the app-of-apps
    pattern.
-   The root application manages the child applications for application
    services and infrastructure components.
-   The documented configuration enables automated synchronization,
    pruning, and self-healing.
-   Git is the desired-state source of truth; Argo CD reconciles the
    cluster to the Helm configuration committed there.

In practical terms, a successful image build alone does not deploy an
application. The image must be published and its tag reflected in the
Git-managed Helm configuration; Argo CD then applies the change to OKE.

## 6. Infrastructure as Code: Terraform and CDK

Terraform and AWS CDK serve the infrastructure-provisioning layer, not
the application deployment layer.

  -----------------------------------------------------------------------
  Technology                          Role
  ----------------------------------- -----------------------------------
  Terraform                           OCI infrastructure provisioning
                                      path under `infra/oci/terraform/`

  AWS CDK                             AWS infrastructure code under
                                      `infra/cdk/`

  Kubernetes / OKE                    Runs application and platform
                                      workloads

  Helm                                Packages and configures Kubernetes
                                      workloads

  Argo CD                             Reconciles Git-managed desired
                                      state into Kubernetes

  GitHub Actions                      Builds, tests, publishes images,
                                      and updates Git-managed image tags
  -----------------------------------------------------------------------

### 6.1 Terraform status and scope

The OCI deployment documentation describes `infra/oci/terraform/` as a
Terraform skeleton that must be reviewed before use. It is intended for
OCI infrastructure provisioning and documents outputs such as the OKE
cluster OCID, Object Storage namespace, and region.

**Important:** The presence of this Terraform skeleton does not prove
that every currently running OCI resource was created or is managed by
Terraform. Confirm the module contents, state backend, and ownership of
existing resources before running `terraform apply` against a live
environment.

Typical reviewed Terraform workflow:

``` bash
cd infra/oci/terraform
cp terraform.tfvars.example terraform.tfvars
terraform init
terraform plan
# Review the plan carefully before applying
terraform apply
```

Terraform should manage cloud infrastructure. It should not replace Helm
or Argo CD for deploying application workloads into Kubernetes.

## 7. Security and configuration boundaries

-   External traffic is terminated/routed through the OCI Load Balancer
    and Kong edge.
-   Domain services are not intended to be exposed individually as
    public endpoints; they are reached through Kubernetes service
    networking and the edge routing layer.
-   Identity Service is the central identity and access-control
    authority for Resident, Staff, and CSR / Partner experiences.
-   Secrets should be supplied through the environment's approved
    secret-management mechanism and referenced by workloads; do not
    commit secret values to Git.
-   Keep environment-specific configuration separate across local,
    staging, and production deployments.
-   Use immutable image tags (Git commit SHA) to make deployments
    traceable and repeatable.

## 8. Architecture ownership by layer

``` mermaid
flowchart TB
    subgraph PRODUCT["Product layer"]
      UX[Resident / Staff / CSR experiences]
      DOM[Domain services and civic workflows]
    end

    subgraph PLATFORM["Application platform layer"]
      K8S[Kubernetes / OKE]
      H[Helm]
      A[Argo CD]
      D[PostgreSQL / Redis / RabbitMQ / other shared services]
    end

    subgraph DELIVERY["Build and delivery layer"]
      GH[GitHub]
      GA[GitHub Actions]
      REG[GHCR]
    end

    subgraph INFRA["Cloud infrastructure layer"]
      TF[Terraform for OCI, where adopted]
      OCI[OCI networking / OKE / storage resources]
    end

    UX --> DOM
    DOM --> K8S
    K8S --> D
    GH --> GA
    GA --> REG
    GA --> GH
    GH --> A
    A --> H
    H --> K8S
    TF --> OCI
    OCI --> K8S
```

  -----------------------------------------------------------------------
  Layer                               Primary responsibility
  ----------------------------------- -----------------------------------
  Product                             User journeys and civic workflows

  Application                         Domain APIs, workers, and business
                                      logic

  Runtime                             Kubernetes scheduling, networking,
                                      scaling, and workload lifecycle

  Packaging                           Helm chart templates and
                                      environment values

  GitOps delivery                     Argo CD reconciliation from Git

  CI                                  GitHub Actions build/test/image
                                      publishing and GitOps tag updates

  Registry                            GHCR container image storage

  Cloud infrastructure                OCI resources, provisioned through
                                      Terraform where Terraform is
                                      adopted
  -----------------------------------------------------------------------

## 9. Operational notes and validation items

Before treating this document as a production topology specification,
validate these items against the active repository and cluster:

1.  **OKE node topology:** Deployment notes have contained differing
    node-count descriptions. Use the current OKE cluster and node-pool
    configuration as the source of truth; this document intentionally
    does not state a node count.
2.  **OpenSearch status:** Confirm whether OpenSearch is enabled in the
    target environment before treating it as an active runtime
    dependency.
3.  **Terraform ownership:** Confirm which OCI resources are actually
    managed by Terraform and whether the Terraform state is configured
    for the target environment.
4.  **Observability:** Confirm which monitoring, logging, dashboards,
    and alerting components are currently deployed. Do not assume that a
    component mentioned in an infrastructure plan is active in
    production.
5.  **Service inventory:** Keep the service list synchronized with the
    current Helm charts and repository service directories.

------------------------------------------------------------------------

## 10. Key architectural decisions

-   One shared Identity Service supports separate Resident, Staff, and
    CSR / Partner experiences.
-   Kong is the single application edge behind the OCI Load Balancer in
    the current OCI deployment.
-   Domain services run inside Kubernetes and are exposed internally
    through Kubernetes Services.
-   Helm defines the Kubernetes workload configuration.
-   Argo CD performs GitOps-based deployment reconciliation from Git.
-   GitHub Actions builds and publishes immutable container images and
    updates Git-managed image tags.
-   Terraform is the OCI infrastructure-provisioning path documented in
    the repository, but its skeleton and live-resource ownership must be
    reviewed before use.
-   Kubernetes application deployment and cloud infrastructure
    provisioning remain separate responsibilities.
