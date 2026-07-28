# Runbook - Project 5 (apatel638)

All commands below have been run and tested. Passwords shown are the one used in this
project (`ClO835Pass!apatel638`) - swap in your own if you rebuild with a different secret.

## 1. Bootstrap from nothing

```bash
./bootstrap.sh
```

Verify the cluster, MySQL, and CronJob are healthy:

```bash
kubectl get pods,pvc,cronjob -n proj5-apatel638
```

## 2. List current backups, newest first

```bash
kubectl apply -f manifests/backup-helper-pod.yaml
sleep 5
MSYS_NO_PATHCONV=1 kubectl exec -n proj5-apatel638 backup-helper-apatel638 -- ls -1t /backups
```

Clean up the helper pod when done:

```bash
kubectl delete pod backup-helper-apatel638 -n proj5-apatel638
```

## 3. Trigger an immediate backup

```bash
kubectl create job --from=cronjob/backup-apatel638 manual-backup-<NAME> -n proj5-apatel638
kubectl get pods -n proj5-apatel638
kubectl logs job/manual-backup-<NAME> -n proj5-apatel638
```

Replace `<NAME>` with anything unique, e.g. `manual-backup-demo1`.

## 4. Inspect a backup's contents without restoring it

```bash
kubectl apply -f manifests/backup-helper-pod.yaml
sleep 5
MSYS_NO_PATHCONV=1 kubectl exec -n proj5-apatel638 backup-helper-apatel638 -- sh -c "gunzip -c /backups/<FILENAME> | grep INSERT"
kubectl delete pod backup-helper-apatel638 -n proj5-apatel638
```

Replace `<FILENAME>` with the exact backup file name from step 2. This is what makes the
prediction possible - you can see exactly which rows a backup contains before touching
the restore Job.

## 5. Insert a new dated row into the live database

```bash
kubectl exec -it -n proj5-apatel638 deploy/mysql-apatel638 -- mysql -uroot -p'ClO835Pass!apatel638' -e "INSERT INTO clo835db.records (student_id, note, added_on) VALUES ('apatel638','<NOTE>','<YYYY-MM-DD>');"
```

## 6. Run the restore Job against a named backup

Edit `manifests/restore-job.yaml` and change the `BACKUP_FILE` value to the target filename.
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

## 7. Prove rotation (before / after)

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

## 8. Tear down and rebuild if something goes wrong mid-demo

```bash
kind delete cluster --name backup-apatel638
./bootstrap.sh
```
