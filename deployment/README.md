# Inbest deployment and data refresh

Our fork of `ideal-postcodes/postcodes.io`. The only Inbest changes are the files in
this folder; everything else follows upstream. It serves welfare, cmsApi and
councilTaxBands inside the cluster (`http://postcodesio.production.svc.cluster.local`)
and publicly as `https://postcodes.inbest.ai`.

## Where the data lives

- The API runs in namespace `production` and reads the database named by the
  `postgres-env-file` configmap, key `POSTGRES_DB` (one database per ONSPD release,
  `postcodesiodb_MMYY`, e.g. `postcodesiodb_1225` = loaded Dec 2025 from the Nov 2025
  ONSPD). The app reads `POSTGRES_DATABASE`; `deployment.yml` maps it from the same key.
- The database sits on the shared postgres pod in namespace `default`, next to the
  `postcodes` database that councilTaxBands uses (`water_providers`, `scotland`).
- Upstream publishes a ready-made dump per ONSPD release; its URL is in `latest`.
  ONS releases quarterly (Feb, May, Aug, Nov); upstream follows a few weeks later.

## Refreshing to a new ONSPD release

1. **Code.** Merge the upstream release tag that carries the new data into a branch,
   keep `deployment/`, open a PR to `master`. Build and push the image with a new tag
   and set it in `deployment.yml`.
2. **Load the dump into a NEW database** (never overwrite the live one):

   ```sh
   POD=$(kubectl get pods -n default -l app=postgres -o jsonpath='{.items[0].metadata.name}')
   NEW=postcodesiodb_MMYY
   PGUSER_APP=$(kubectl get configmap postgres-env-file -n production -o jsonpath='{.data.POSTGRES_USER}')
   kubectl exec -n default $POD -- psql -U postgres -c "CREATE DATABASE $NEW"
   kubectl exec -n default $POD -- psql -U postgres -d $NEW -c "CREATE EXTENSION IF NOT EXISTS postgis"
   curl -sS "$(cat latest)" | gunzip \
     | sed -e '/^\\restrict /d' -e '/^\\unrestrict /d' -e '/^SET transaction_timeout/d' \
     | kubectl exec -i -n default $POD -- psql -U postgres -d $NEW -v ON_ERROR_STOP=1 -q
   kubectl exec -n default $POD -- psql -U postgres -d $NEW -c "
     GRANT USAGE ON SCHEMA public TO \"$PGUSER_APP\";
     GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"$PGUSER_APP\";
     ANALYZE;"
   ```

   The `sed` drops lines an older `psql`/server rejects (`\restrict` comes from
   pg_dump 17.6+, `transaction_timeout` needs PG 17). The dump needs PostGIS
   (`geography` columns). Expect ~1.8M live + ~0.9M terminated postcodes in
   `public.postcodes` (Aug 2026: 1,810,364 live, 918,726 terminated).
3. **Check the new database** before switching:

   ```sql
   SELECT count(*) FILTER (WHERE date_of_termination IS NULL) AS live,
          count(*) FILTER (WHERE date_of_termination IS NOT NULL) AS terminated
     FROM public.postcodes;
   SELECT postcode, admin_district, region FROM public.postcodes WHERE postcode = 'SW1A 1AA';
   ```
4. **Rebuild councilTaxBands `water_providers` from the new database in the same
   change** — `councilTaxBands/docs/water-providers.md`, "Refresh runbook", steps 3–6.
   Skipping it makes `/waterProvider` return 422 for every new postcode.
5. **Switch:** set `POSTGRES_DB` in the `postgres-env-file` configmap (namespace
   `production`) to the new name, deploy the new image, check
   `curl https://postcodes.inbest.ai/postcodes/SW1A1AA`, then swap `water_providers`.
   Rollback = put the old name back and restart the deployment. Keep the old database
   until a week of clean traffic, then drop it.
6. **Dependent datasets in welfare** (`constants/adminDistrictToCode.json`,
   `outcodeToRegion.json`, `outcodeToBrma.json`): compare the new release's
   `admin_district` names and live outcodes against them; a name missing from
   `adminDistrictToCode` sends `code: undefined` for that council.
