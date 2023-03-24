#### Following query returns top 5 biggest tables in the database

```sql
SELECT
    relname AS "relation",
    pg_size_pretty (pg_total_relation_size (C .oid)) AS "total_size"
FROM
    pg_class C
LEFT JOIN pg_namespace N ON (N.oid = C .relnamespace)
WHERE
    nspname NOT IN ('pg_catalog', 'information_schema')
AND C .relkind <> 'i'
AND nspname !~ '^pg_toast'
ORDER BY
    pg_total_relation_size (C .oid) DESC
LIMIT 5;
```

# Clear DB 

```bash
docker cp /root/sentry/clear_db.sql sentry-self-hosted-postgres-1:/clear_db.sql
docker exec -i sentry-self-hosted-postgres-1 /bin/bash -c "psql -U postgres postgres < clear_db.sql"
```
