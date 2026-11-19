# Inbest postcodes.io: what runs and how to refresh the data

**Production does not run code from this repository.** Deployment `postcodes-api`
(namespace `production`, service `postcodesio`) runs upstream's published image
`idealpostcodes/postcodes.io:release-<version>`. The two files here are the live
manifests, exported from the cluster on 2026-11-19, so the repo matches production.

Callers: welfare and cmsApi (`http://postcodesio.production.svc.cluster.local`,
public `https://postcodes.inbest.ai`), councilTaxBands (water provider gate).

## Data

One database per ONS Postcode Directory release, `postcodesiodb_MMYY`, on the shared
`postgres` pod in namespace `default` (Postgres 15, PostGIS). The live one is
`postgres-env-file` (production) key `POSTGRES_DB`, which the app reads as
`POSTGRES_DATABASE`. Live since 2026-11-19: `postcodesiodb_0826` (ONSPD Aug 2026,
upstream 20.1.0).

ONS publishes quarterly (Feb, May, Aug, Nov); upstream publishes a dump a few weeks
later. Its URL is the `latest` file of the matching upstream release; use the image
tag of that same release with it (the schema changes between majors).

## Refresh

1. **Room:** the pod's disk must hold the old and the new database together (a
   release is ~2.8 GB). Check `df -h /var/lib/postgresql/data`; the PVC
   `postgres-pv-claim` can be grown online.
2. **Load into a new database** (never the live one):

   ```sh
   POD=$(kubectl get pods -n default -l app=postgres -o jsonpath='{.items[0].metadata.name}')
   NEW=postcodesiodb_MMYY
   DUMP=https://postcodesio.s3.amazonaws.com/public/postcodesio-<date>.sql.gz   # upstream `latest`
   kubectl exec -n default $POD -- psql -U postgres -c "CREATE DATABASE $NEW"
   kubectl exec -n default $POD -- psql -U postgres -d $NEW -c "CREATE EXTENSION IF NOT EXISTS postgis"
   curl -sS "$DUMP" | gunzip \
     | sed -e '/^\\restrict /d' -e '/^\\unrestrict /d' -e '/^SET transaction_timeout/d' \
     | kubectl exec -i -n default $POD -- psql -U postgres -d $NEW -v ON_ERROR_STOP=1 -q
   ```

   The `sed` drops lines Postgres 15 rejects. The dump carries its own indexes.
   2026-11-19: ~15 minutes, 1,810,364 live + 918,726 terminated postcodes.
3. **Compare** a random sample of live lookups old vs new (run the new release
   locally against the new database through a port-forward). Expect only new
   postcodes and a few ward changes.
4. **Water providers:** rebuild councilTaxBands `water_providers` from the new
   database in the same change: `councilTaxBands/docs/water-providers.md` steps 3–6.
   Never run its validation on all postcodes against the shared pod (435 MiB limit).
5. **Switch, back to back:** swap the water tables; set `POSTGRES_DB` in
   `postgres-env-file` to the new name; `kubectl set image deploy/postcodes-api -n
   production '*=idealpostcodes/postcodes.io:release-<version>'`. Check
   `https://postcodes.inbest.ai/postcodes/SW1A1AA` and the ctbands logs.
6. **Rollback:** the same three steps reversed. Drop the old database and the old
   water table once you are happy (2026-11-19: dropped the same day).
7. **Welfare's bundled files** (`constants/adminDistrictToCode.json`,
   `outcodeToRegion.json`, `outcodeToBrma.json`): check every `admin_district` name
   of the new release maps in `adminDistrictToCode` (a missing name sends
   `code: undefined`). Aug 2026: all mapped.
