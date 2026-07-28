# Runbook - Project 5 (apatel638)

Every command below has actually been run and tested against the real cluster. Passwords
shown are the one used in this project (`ClO835Pass!apatel638`) - swap in your own if you
rebuild with a different secret. Each section has a small diagram showing what's actually
happening when you run it, not just the commands themselves.

---

## 1. Bootstrap from nothing

What this does: rebuilds the entire system, in order, on a machine that has nothing but
Docker, kind, kubectl, and git.

```mermaid
graph LR
    A["Clean host<br/>Docker + kind + kubectl"] --> B["kind cluster<br/>created"]
    B --> C["Namespace, Secret,<br/>PVCs, MySQL applied"]
    C --> D["MySQL ready<br/>ping-retry loop"]
    D --> E["Schema + rows<br/>seeded"]
    E --> F["CronJob on,<br/>first backup taken"]

    style A fill:#fff,stroke:#333,color:#000
    style B fill:#fff,stroke:#333,color:#000
    style C fill:#fff,stroke:#333,color:#000
    style D fill:#fff,stroke:#333,color:#000
    style E fill:#fff,stroke:#333,color:#000
    style F fill:#fff,stroke:#333,color:#000
```

```bash
./bootstrap.sh
```

Verify the cluster, MySQL, and CronJob are healthy:

```bash
kubectl get pods,pvc,cronjob -n proj5-apatel638
```

---

## 2. List current backups, newest first

What this does: a short-lived pod mounts the same backup PVC the CronJob writes to, and
lists what's on it - no restore happens here, this is read-only.

```mermaid
graph LR
    A["Helper pod<br/>created"] --> B["Mounts PVC<br/>backup-apatel638"]
    B --> C["ls -1t /backups<br/>newest first"]
    C --> D["Helper pod<br/>deleted"]

    style A fill:#fff,stroke:#333,color:#000
    style B fill:#fff,stroke:#333,color:#000
    style C fill:#fff,stroke:#333,color:#000
    style D fill:#fff,stroke:#333,color:#000
```

```bash
kubectl apply -f manifests/backup-helper-pod.yaml
sleep 5
MSYS_NO_PATHCONV=1 kubectl exec -n proj5-apatel638 backup-helper-apatel638 -- ls -1t /backups
```

Clean up the helper pod when done:

```bash
kubectl delete pod backup-helper-apatel638 -n proj5-apatel638
```

---

## 3. Trigger an immediate backup

What this does: manually fires one run of the same job the CronJob would run on its own
schedule - dump, gzip, save, then rotate anything past the 7 newest.

```mermaid
graph LR
    A["kubectl create job<br/>--from=cronjob"] --> B["Pod runs<br/>mysqldump | gzip"]
    B --> C["File saved to<br/>backup-apatel638 PVC"]
    C --> D["Rotation:<br/>keep newest 7"]

    style A fill:#fff,stroke:#333,color:#000
    style B fill:#fff,stroke:#333,color:#000
    style C fill:#fff,stroke:#333,color:#000
    style D fill:#fff,stroke:#333,color:#000
```

```bash
kubectl create job --from=cronjob/backup-apatel638 manual-backup-<NAME> -n proj5-apatel638
kubectl get pods -n proj5-apatel638
kubectl logs job/manual-backup-<NAME> -n proj5-apatel638
```

Replace `<NAME>` with anything unique, e.g. `manual-backup-demo1`.

---

## 4. Inspect a backup's contents without restoring it

What this does: reads the rows inside a specific backup file directly, without touching
the live database at all. This is what makes predicting the restore outcome possible.

```mermaid
graph LR
    A["Chosen backup<br/>.sql.gz file"] --> B["gunzip -c<br/>decompress"]
    B --> C["grep INSERT<br/>show rows only"]
    C --> D["Live database<br/>untouched"]

    style A fill:#fff,stroke:#333,color:#000
    style B fill:#fff,stroke:#333,color:#000
    style C fill:#fff,stroke:#333,color:#000
    style D fill:#fff,stroke:#333,color:#000
```

```bash
kubectl apply -f manifests/backup-helper-pod.yaml
sleep 5
MSYS_NO_PATHCONV=1 kubectl exec -n proj5-apatel638 backup-helper-apatel638 -- sh -c "gunzip -c /backups/<FILENAME> | grep INSERT"
kubectl delete pod backup-helper-apatel638 -n proj5-apatel638
```

---

## 5. Insert a new dated row into the live database

What this does: adds one row directly to the running MySQL pod - this is how the project's
"data grows over time" requirement is satisfied.

```mermaid
graph LR
    A["kubectl exec<br/>into MySQL pod"] --> B["INSERT INTO<br/>clo835db.records"]
    B --> C["Row exists in<br/>live table now"]

    style A fill:#fff,stroke:#333,color:#000
    style B fill:#fff,stroke:#333,color:#000
    style C fill:#fff,stroke:#333,color:#000
```

```bash
kubectl exec -it -n proj5-apatel638 deploy/mysql-apatel638 -- mysql -uroot -p'ClO835Pass!apatel638' -e "INSERT INTO clo835db.records (student_id, note, added_on) VALUES ('apatel638','<NOTE>','<YYYY-MM-DD>');"
```

---

## 6. Run the restore Job against a named backup

What this does: reloads one specific backup file exactly - the table is dropped first, so
the result is that backup's exact state, not a merge with whatever was already there.

```mermaid
graph LR
    A["Edit BACKUP_FILE<br/>in restore-job.yaml"] --> B["Delete old<br/>restore Job"]
    B --> C["Apply new<br/>restore Job"]
    C --> D["DROP TABLE,<br/>then reload dump"]
    D --> E["Live DB now<br/>matches that backup"]

    style A fill:#fff,stroke:#333,color:#000
    style B fill:#fff,stroke:#333,color:#000
    style C fill:#fff,stroke:#333,color:#000
    style D fill:#fff,stroke:#333,color:#000
    style E fill:#fff,stroke:#333,color:#000
```

Jobs are immutable once created, so any previous restore Job needs to be deleted first:

```bash
kubectl delete job restore-job-apatel638 -n proj5-apatel638 --ignore-not-found
kubectl apply -f manifests/restore-job.yaml
kubectl logs -n proj5-apatel638 job/restore-job-apatel638
```

Verify the result:

```bash
kubectl exec -it -n proj5-apatel638 deploy/mysql-apatel638 -- mysql -uroot -p'ClO835Pass!apatel638' -e "SELECT * FROM clo835db.records ORDER BY added_on;"
```

---

## 7. Prove rotation (before / after)

What this does: shows the file count on the backup PVC never exceeds 7 - a new backup
pushes out the oldest one, it doesn't just accumulate forever.

```mermaid
graph LR
    A["Before: list<br/>backups, count 7"] --> B["Trigger one<br/>new backup"]
    B --> C["Rotation drops<br/>the oldest file"]
    C --> D["After: list<br/>backups, still 7"]

    style A fill:#fff,stroke:#333,color:#000
    style B fill:#fff,stroke:#333,color:#000
    style C fill:#fff,stroke:#333,color:#000
    style D fill:#fff,stroke:#333,color:#000
```

List backups before triggering a new one (see step 2), note the count and the oldest
filename. Trigger a new backup (see step 3). List backups again:

```bash
kubectl apply -f manifests/backup-helper-pod.yaml
sleep 5
MSYS_NO_PATHCONV=1 kubectl exec -n proj5-apatel638 backup-helper-apatel638 -- ls -1t /backups
kubectl delete pod backup-helper-apatel638 -n proj5-apatel638
```

Confirm the count is still capped at 7, the oldest file from the "before" list is gone,
and the new file is present.

---

## 8. Tear down and rebuild if something goes wrong mid-demo

What this does: if anything breaks live during the demo, this wipes the cluster completely
and rebuilds it from the same repo, using the same script that was already tested.

```mermaid
graph LR
    A["Something<br/>breaks mid-demo"] --> B["kind delete<br/>cluster"]
    B --> C["./bootstrap.sh<br/>runs again"]
    C --> D["Working system<br/>restored, ~2 min"]

    style A fill:#fff,stroke:#333,color:#000
    style B fill:#fff,stroke:#333,color:#000
    style C fill:#fff,stroke:#333,color:#000
    style D fill:#fff,stroke:#333,color:#000
```

```bash
kind delete cluster --name backup-apatel638
./bootstrap.sh
```
