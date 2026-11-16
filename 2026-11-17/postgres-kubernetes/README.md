# PostgreSQL on Kubernetes

> PostgreSQL on Kubernetes runs database instances as orchestrated workloads, with an operator automating database lifecycle operations.

A running database is easy to demonstrate. A recoverable database is harder to prove.

## What is it?

- **PostgreSQL** is the database engine. **Kubernetes** manages workload placement and infrastructure resources. An **operator** extends Kubernetes with database-specific reconciliation: it observes the desired configuration and acts to maintain it. CloudNativePG provides this through a `Cluster` custom resource. [CloudNativePG overview](https://cloudnative-pg.io/docs/)
- **Primary:** the instance serving writes. **Standby:** an instance replaying changes from the primary. **WAL:** the write-ahead log used for recovery and physical replication. A hot standby can serve read-only queries. [PostgreSQL replication overview](https://www.postgresql.org/docs/current/high-availability.html)
- **PVC:** a PersistentVolumeClaim requesting storage for an instance. CloudNativePG uses separate instance volumes; sharing a single data directory is not the replication model. [Storage](https://cloudnative-pg.io/docs/current/storage/)
- **HA** addresses continuity through some component failures. **Disaster recovery** addresses recovery after larger failures or data loss. They require different planning. [Architecture](https://cloudnative-pg.io/docs/current/architecture/)

## Why does it matter today?

- Platform teams can manage database infrastructure declaratively alongside application infrastructure. CNCF announced CloudNativePG's incubation on 10 November 2026; this is a project milestone, not a guarantee that a particular deployment meets its reliability targets. [CNCF announcement](https://www.cncf.io/blog/2026/11/10/cloudnativepg-becomes-a-cncf-incubating-project/)
- **Consider it when:** infrastructure control, on-premises operation or a consistent Kubernetes operating model justifies ownership of storage, upgrades and database incidents. This is a decision criterion, not a universal recommendation.
- **Prefer another option when:** the team cannot support both PostgreSQL and Kubernetes, or a managed service already meets the requirements with less operational work. Managed databases still leave customers responsible for access, data and application-level security. [Amazon RDS shared responsibility](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.html)
- Compare total operating cost: compute, storage, standby capacity, backup storage, data transfer and engineering effort. Portability does not eliminate storage-provider or operational differences. [Storage considerations](https://cloudnative-pg.io/docs/current/storage/)

## Where can I use it?

- **Application databases:** transactional services that need a primary and standby topology, provided their latency and recovery requirements can be met. [Supported architecture](https://cloudnative-pg.io/docs/current/architecture/)
- **Platform-provided databases:** standardize database provisioning through operator-managed configuration rather than separate manual procedures. [CloudNativePG capabilities](https://cloudnative-pg.io/docs/)
- **Documented integration:** CNCF describes OpenBao using CloudNativePG as a PostgreSQL backend. Treat this as a concrete integration reference, not proof of your own production readiness. [CNCF project references](https://www.cncf.io/projects/cloudnativepg/)
- **Implementation choices:** CloudNativePG on Kubernetes; provider-managed PostgreSQL; or PostgreSQL on VMs. The main tradeoff is where lifecycle automation and operational ownership sit. [Operator](https://cloudnative-pg.io/docs/) · [Managed-service responsibilities](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.html)

## How does it work?

1. **Declare the desired database:** instance count, storage and placement settings become inputs to the operator. [Operator architecture](https://cloudnative-pg.io/docs/current/architecture/)
2. **Provision instances and volumes:** each database instance receives persistent storage appropriate to the workload. [Storage configuration](https://cloudnative-pg.io/docs/current/storage/)
3. **Route writes:** the application connects through the read/write Service to the current primary. The operator manages the topology; it is not a SQL proxy. [Connection architecture](https://cloudnative-pg.io/docs/current/architecture/)
4. **Replicate changes:** PostgreSQL streams WAL to standbys. Commit durability and availability depend on the replication configuration. [Replication](https://cloudnative-pg.io/docs/current/replication/)
5. **Recover from instance failure:** eligible standbys can be promoted and the write endpoint redirected. Clients must reconnect; this is not a promise of interruption-free transactions. [PostgreSQL standby behavior](https://www.postgresql.org/docs/current/warm-standby.html)
6. **Recover earlier data:** restore a base backup and replay the required archived WAL to the target time. Outputs are a recovered database and a validated application—not merely a completed backup job. [PostgreSQL PITR](https://www.postgresql.org/docs/current/continuous-archiving.html)

```text
                         Operator
                    manages desired state
                             |
Application --> Write Service --> Primary [own volume]
                                    | WAL
                         +----------+----------+
                         v                     v
                   Standby B              Standby C
                  [own volume]           [own volume]

Base backup + continuous WAL archive
                 |
                 v
       Restore --> Validate --> Cut over
```

- The topology is illustrative. Configure node/zone placement and suitable storage; three instances alone do not establish three independent failure domains. [Placement considerations](https://cloudnative-pg.io/docs/current/architecture/)
- [Illustrated architecture](notes-page-1.png) · [Risk and recovery notes](notes-page-2.png)

## What can go wrong?

- **Data loss at failover:** asynchronous standbys can lag. Synchronous replication adds commit coordination and can delay writes when required standbys are unavailable. [PostgreSQL replication tradeoffs](https://www.postgresql.org/docs/current/warm-standby.html)
- **Logical mistakes:** replication also propagates unwanted data changes. Recovering a pre-error state requires usable backup history and WAL, not simply another replica. [Point-in-time recovery](https://www.postgresql.org/docs/current/continuous-archiving.html)
- **Storage exhaustion or latency:** slow disks affect database work; retained WAL and growing data consume capacity. Storage performance and lifecycle behavior must be assessed. [Storage](https://cloudnative-pg.io/docs/current/storage/) · [Replication slots](https://cloudnative-pg.io/docs/current/replication/)
- **Larger failure domains:** an entire Kubernetes-cluster outage needs a separate recovery plan. Cross-cluster recovery is not the same as local standby promotion. [Replica clusters](https://cloudnative-pg.io/documentation/current/replica_cluster/)
- **Security exposure:** excessive Kubernetes permissions can expose workloads and credentials; database credentials and network access need their own controls. [Kubernetes RBAC guidance](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)
- **Compatibility and ownership:** operator, PostgreSQL, backup plugins and storage capabilities must work together. Upgrade and recovery procedures are version-dependent. [Barman Cloud plugin usage](https://cloudnative-pg.io/plugin-barman-cloud/docs/usage/)

## How do we handle those risks?

- **Set measurable recovery requirements:** define acceptable data loss (RPO) and recovery time (RTO), then validate the chosen architecture against them. [Business continuity architecture](https://cloudnative-pg.io/docs/current/architecture/)
- **Protect recovery material:** keep base backups and continuous WAL archives in a separately protected destination; restrict deletion and test restoration into an isolated environment. PITR requires a complete WAL sequence covering recovery. [PITR requirements](https://www.postgresql.org/docs/current/continuous-archiving.html)
- **Run failure drills:** test primary loss, application reconnection and recovery from an accidental data change. Check row-level/application correctness as well as elapsed time. These are recommended acceptance tests, not built-in guarantees.
- **Monitor useful signals:** query activity, connection pressure, replication progress and archiver failures. PostgreSQL exposes views including `pg_stat_activity`, `pg_stat_replication` and `pg_stat_archiver`; add infrastructure disk and I/O monitoring. [Statistics reference](https://www.postgresql.org/docs/current/monitoring-stats.html)
- **Harden access:** use scoped identities, controlled network paths and encrypted connections; enable storage encryption where the infrastructure supports it. [RBAC](https://kubernetes.io/docs/concepts/security/rbac-good-practices/) · [Storage encryption](https://cloudnative-pg.io/docs/current/storage/)
- **Plan change recovery:** test supported upgrade paths with representative data, confirm backup compatibility and define a recovery procedure before production changes. Do not assume every database upgrade is reversible by changing an image tag. [PostgreSQL upgrades](https://www.postgresql.org/docs/current/upgrading.html)

## Where can I learn more?

- [CloudNativePG documentation](https://cloudnative-pg.io/docs/) — operator capabilities and implementation guidance.
- [PostgreSQL replication](https://www.postgresql.org/docs/current/warm-standby.html) — streaming, durability and standby behavior.
- [PostgreSQL PITR](https://www.postgresql.org/docs/current/continuous-archiving.html) — backup and WAL recovery requirements.
- [Barman Cloud plugin](https://cloudnative-pg.io/plugin-barman-cloud/docs/usage/) — practical backup integration.
- [Monitoring reference](https://www.postgresql.org/docs/current/monitoring-stats.html) — database and replication statistics.

## LinkedIn post

Draft for review; not scheduled or published. The prepared social caption is reproduced below, including its original formatting and hashtags. [Copy-ready caption](caption.txt)

```text
Should you run PostgreSQL inside Kubernetes? 🐘 ☸️

Your application runs on Kubernetes. Should its database move there too?

Sometimes—but the decision needs more than a working deployment.

📌 Why this matters now
CNCF announced CloudNativePG’s incubation on 10 November 2026. It is a timely reason to revisit database operations on Kubernetes, not a guarantee that every workload belongs there.

🔎 What does an operator do?
An operator such as CloudNativePG turns a declarative database configuration into managed PostgreSQL instances and automates lifecycle tasks, including failover.

Kubernetes handles workload orchestration. PostgreSQL handles data. Your team still owns reliability.

✅ When it can help
• Repeatable provisioning and GitOps workflows.
• Infrastructure control across cloud and on-premises environments.
• A common operating approach for teams already skilled in Kubernetes AND PostgreSQL.

A small team should also evaluate managed PostgreSQL. Compare the full cost: compute, storage, replicas, backups, engineering and on-call—not just the database bill.

⚠️ A real-world example
Imagine your checkout database has a primary and two standbys across three zones.

If the primary fails, the operator can promote an eligible standby and redirect the write service. Applications still need to reconnect and handle uncertain transactions safely.

But if someone accidentally deletes the orders table, replication can carry that mistake to the standbys.

High availability and recovery from bad data changes are different problems.

🛠️ How do we manage the risks?
• Configure failure-domain placement and test failover.
• Choose replication settings against acceptable data loss; asynchronous replication can lag, while synchronous replication can add latency or block writes.
• Protect backups outside the cluster and continuously archive WAL—the write-ahead log.
• Practise point-in-time recovery into an isolated cluster.
• Monitor disk space, query latency, connections, replication lag and backup health.
• Use least privilege, TLS, restricted networking and tested upgrade procedures.

Define your RPO (acceptable data loss) and RTO (acceptable recovery time), then measure whether recovery drills meet them.

The attached notes cover the architecture, operating-model choices, risks and safeguards.

What would make you choose PostgreSQL on Kubernetes over a managed database?

For more such posts and illustrated notes, visit: https://github.com/ankushcore/linkedin-posts

#DevOps #SRE #PostgreSQL #Kubernetes #CloudNativePG #Database #DatabaseReliability #DatabaseAdministration #CloudNative #PlatformEngineering #GitOps #HighAvailability #DisasterRecovery #Observability #DevSecOps
```
