# Phase 7 — Part 1: Docker Infrastructure (TheHive + Cortex)
 
## Overview
 
Phases 1 through 6 built a detection capability: telemetry sources, custom rules, and an operational dashboard that turns raw events into alerts. But detection is only the first half of a SOC's job. An alert that fires on a dashboard is not an outcome — it is the start of a process. Someone has to open it, investigate it, enrich the indicators it contains, decide what it means, and act. Phase 7 builds the half of the SOC that begins where the alert ends.
 
This part deploys the foundation of the response layer: a dedicated host running **TheHive** (case management) and **Cortex** (analysis and enrichment) as containers. The design question here is not "how do I detect?" but "once something has been detected, what does an analyst actually do with it?". The answer is a workflow, alert becomes case, case accumulates observables, observables get enriched with context, context drives a decision, and that workflow needs tooling that Wazuh alone does not provide.
 
The response platform was deliberately placed on a new, separate VM rather than co-located with the Wazuh Manager. The two roles have different failure modes: a runaway container or an out-of-memory event on the response stack must never be able to take down the SIEM that is the lab's primary source of truth. Isolation on a dedicated host, within the same out-of-band SOC VLAN, keeps the blast radius contained while preserving low-latency communication between the two.
 
---
 
## Architecture
 
```mermaid
flowchart TB
    subgraph vlan99["VLAN 99 · SOC · 10.10.99.0/24"]
        wazuh["wazuh-srv<br/>10.10.99.10<br/>Wazuh Manager + Indexer + Dashboard"]
        subgraph socplat["soc-platform · 10.10.99.20 · Docker host"]
            thehive["TheHive 5.2<br/>:9000"]
            cortex["Cortex 3.1.7<br/>:9001"]
            cassandra["Cassandra<br/>datastore"]
            elastic["Elasticsearch<br/>index"]
            minio["MinIO<br/>attachments"]
        end
    end
    wazuh -.->|"webhook · level ≥ 10<br/>"| thehive
    thehive -->|"analyzer requests<br/>API + bearer key"| cortex
    thehive --> cassandra
    thehive --> elastic
    thehive --> minio
    cortex --> elastic
    cortex -.->|"runs analyzers<br/>as containers"| dockersock["/var/run/docker.sock"]
```
 
All five containers share a single Docker bridge network, so services address each other by name (`cortex`, `cassandra`, `elasticsearch`) rather than by IP. TheHive is the operational entry point on port 9000; Cortex sits behind it on 9001, invoked over its API. Cassandra, Elasticsearch, and MinIO are backing stores that TheHive requires and that an analyst never touches directly. The dashed webhook from Wazuh is the integration that closes the loop between detection and response, it is documented and deployed in a later part; here the platform is built and validated in isolation so that any fault is attributable to the platform itself, not to the integration.
 
---
 
## Host preparation
 
### VM specifications
 
![soc-platform network validation](../../screenshots/07-soc-platform/01-vm-preparation.png)
 
The VM was placed on VLAN 99 alongside the existing Wazuh host. Before any container was deployed, connectivity was validated in both directions that matter: reachability to the Wazuh Manager (`10.10.99.10`) and reachability to external networks (required to pull container images and to let Cortex analyzers query online services).
 
![soc-platform network validation](../../screenshots/07-soc-platform/02-network-validation.png)

---
 
## The stack
 
### Services
 
The `thehive-cortex` stack comprises five services on a single bridge network:
 
| Service | Image | Role |
| ------- | ----- | ---- |
| cassandra | cassandra:4.1 | Primary datastore for TheHive |
| elasticsearch | elasticsearch:7.17.9 | Index backend for TheHive |
| minio | minio (latest) | S3-compatible object storage for case attachments |
| cortex | thehiveproject/cortex:3.1.7 | Analysis and enrichment engine |
| thehive | strangebee/thehive:5.2 | Case management platform |

---
 
## Deployment
 
### Bringing up the stack
 
With the compose file in place, the stack was launched detached:
 
```bash
cd ~/soc-platform/thehive-cortex
docker compose up -d
```
 
The first launch pulls several multi-gigabyte images. Startup is **not** instantaneous once they land: Cassandra and Elasticsearch take one to two minutes to become ready, and TheHive retries its backend connections during that window. Transient "connection refused" lines in the TheHive log during the first minutes are expected and clear on their own once the backends report healthy. Startup was followed with:

 
Once all five containers reported `Up` — with Cassandra showing `healthy` — the stack was considered live.
 
![docker compose ps — all services Up](../../screenshots/07-soc-platform/03-docker-compose-ps.png)

---
 
## Initial configuration
 
### TheHive
 
TheHive 5 separates platform administration from operational work, and the configuration follows that split:
 
1. The platform superadmin (`admin@thehive.local`) had its default password changed on first login.
2. An organisation, `SOC-HomeLab`, was created to hold all case work.
3. An operational analyst user (`daniel@soc-homelab.local`) was created inside that organisation with the `org-admin` profile.
The superadmin exists only to manage organisations and platform-level connectors; every case is worked as the analyst. This mirrors how TheHive is run in production, where the person triaging alerts is never the same identity that administers the platform.
 
![TheHive organisation and analyst user](../../screenshots/07-soc-platform/04-thehive-org-user.png)
 
### Cortex
 
Cortex has its own first-start sequence:
 
1. The database index was initialised through the "Update Database" migration on first access.
2. A platform superadmin was created.
3. An organisation, `SOC-HomeLab`, was created and set to Active.
4. The dedicated `thehive-integration` user was created with `read`, `analyze`, and `orgadmin` roles, and an API key was generated for it.

![Cortex organization](../../screenshots/07-soc-platform/05-cortex-org-user.png)
![Cortex integration user](../../screenshots/07-soc-platform/06-cortex-org-user-2.png)





