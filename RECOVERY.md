# Nexus H2 Corruption Recovery

For `nexus_aoc` (`sonatype/nexus3:3.72.0`, data at `./data/nexus-data-aoc`).

## Symptom

Container won't start. Logs show `MyBatisDataStore` schema errors, or:

```
org.h2.mvstore.MVStoreException: Invalid chunk id 0
```

This is a known bug in the H2 build Nexus 3.72.0 bundles (2.2.224), usually
triggered by an unclean shutdown (container killed before the JVM finished
flushing H2, or an OOM).

## Prevent it

In `docker-compose.yml`:

```yaml
    stop_grace_period: 120s   # let the JVM shut down cleanly
    mem_limit: 6g             # headroom above Xmx2703m + MaxDirectMemorySize2703m
```

## Recovery steps

**0. Stop the container and back everything up first.** Copy the whole
`db/` directory before touching anything. Keep every intermediate copy
until the instance is confirmed healthy.

**1. Get a current H2 jar** (the bundled 2.2.224 Recover tool has its own
NPE bug on corrupted chunks — issue h2database/h2database#3931):

```bash
curl -L -o h2-2.4.240.jar https://repo1.maven.org/maven2/com/h2database/h2/2.4.240/h2-2.4.240.jar
```

**2. Dump the corrupted database to SQL:**

```bash
docker run --rm \
  -v "$(pwd)/data/nexus-data-aoc/db:/db" \
  -v "$(pwd)/h2-2.4.240.jar:/tmp/h2.jar:ro" \
  --entrypoint java sonatype/nexus3:3.72.0 \
  -cp /tmp/h2.jar org.h2.tools.Recover -dir /db -db nexus
```

Check `nexus.h2.sql` isn't tiny/empty before continuing.

**3. Move the broken file aside (don't delete), then rebuild:**

```bash
mv ./data/nexus-data-aoc/db/nexus.mv.db ./data/nexus-data-aoc/db/nexus.mv.db.broken

docker run --rm \
  -v "$(pwd)/data/nexus-data-aoc/db:/db" \
  -v "$(pwd)/h2-2.4.240.jar:/tmp/h2.jar:ro" \
  --entrypoint java sonatype/nexus3:3.72.0 \
  -cp /tmp/h2.jar org.h2.tools.RunScript -url "jdbc:h2:/db/nexus" -user sa -password "" -script /db/nexus.h2.sql
```

(Nexus's H2 default admin login is blank `sa` / blank password.)

**4. Start Nexus and watch the logs.**

## Known follow-up issues after "successful" recovery

The dump-and-reload doesn't always round-trip every column cleanly. Seen so far:

- **A specific table throws a query error on startup** (e.g. a JSON column
  in `user_role_mapping`) — fix that one row/table rather than redoing the
  whole recovery.
- **Login fails with a keystore error** (`KeystoreException`, `Invalid
  keystore format`) — the SSL plugin's keystore bytes live in the
  `KEY_STORE_DATA` table, not on disk. Fix:

  ```bash
  docker run --rm -it \
    -v "$(pwd)/data/nexus-data-aoc/db:/db" \
    -v "$(pwd)/h2-2.4.240.jar:/tmp/h2.jar:ro" \
    --entrypoint java sonatype/nexus3:3.72.0 \
    -cp /tmp/h2.jar org.h2.tools.Shell -url "jdbc:h2:/db/nexus" -user sa -password ""
  ```

  ```sql
  DELETE FROM KEY_STORE_DATA;
  ```

  Restart — Nexus regenerates a fresh default certificate. Any custom
  trusted certs added via Administration → SSL Certificates will need
  re-adding afterwards.

## Upgrading off 3.72.0

3.72.0 is a version specifically named in reports of this same H2 bug.
Target a recent-but-settled line (a few months old, not the newest release)
rather than 3.72.0 or the latest release. Direct upgrade from 3.72.0 is
supported (no OrientDB/legacy-H2 stepping stone needed). Check for your
target version:

- **3.85.0+**: run "Repair – Rebuild repository search" after upgrading.
- **3.87.0+**: bundles its own Java 21 runtime; old truststore isn't
  reused — reconfigure any custom certs (LDAP, proxies).
