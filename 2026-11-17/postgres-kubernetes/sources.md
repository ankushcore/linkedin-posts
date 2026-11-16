# Sources

- CNCF milestone, 10 November 2026: https://www.cncf.io/blog/2026/11/10/cloudnativepg-becomes-a-cncf-incubating-project/
- Operator architecture, primary/write service, replication, failure domains and cross-cluster limitations: https://cloudnative-pg.io/docs/current/architecture/
- Replication durability tradeoffs: https://cloudnative-pg.io/docs/current/replication/
- Persistent storage considerations: https://cloudnative-pg.io/docs/current/storage/
- Base backup and continuous WAL requirements for PITR: https://www.postgresql.org/docs/current/continuous-archiving.html
- Backup plugin usage: https://cloudnative-pg.io/plugin-barman-cloud/docs/usage/
- Recovery creates a new cluster: https://github.com/cloudnative-pg/cloudnative-pg/blob/main/docs/src/recovery.md
- Managed-service shared security responsibilities: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.html

Research used primary-source search extracts where direct documentation fetches returned empty pages/cache misses. Version-specific deployment instructions are intentionally excluded; verify against the installed operator/PostgreSQL versions before implementation.
