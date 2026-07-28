\# Runbook - Project 5 (apatel638)



All commands below have been run and tested. Passwords shown are the one used in this

project (`ClO835Pass!apatel638`) - swap in your own if you rebuild with a different secret.



\## 1. Bootstrap from nothing



```bash

./bootstrap.sh

```



Verify the cluster, MySQL, and CronJob are healthy:



```bash

kubectl get pods,pvc,cronjob -n proj5-apatel638

```



\## 2. List current backups, newest first



```bash

kubectl apply -f manifests/backup-helper-pod.yaml

sleep 5

MSYS\_NO\_PATHCONV=1 kubectl exec -n proj5-apatel638 backup-helper-apatel638 -- ls -1t /backups

```



Clean up the helper pod when done:



```bash

kubectl delete pod backup-helper-apatel638 -n proj5-apatel638

```



\## 3. Trigger an immediate backup



```bash

kubectl create job --from=cronjob/backup-apatel638 manual-backup-<NAME> -n proj5-apatel638

kubectl get pods -n proj5-apatel638

kubectl logs job/manual-backup-<NAME> -n proj5-apatel638

```



\## 4. Inspect a backup's contents without restoring it



```bash

kubectl apply -f manifests/backup-helper-pod.yaml

sleep 5

MSYS\_NO\_PATHCONV=1 kubectl exec -n proj5-apatel638 backup-helper-apatel638 -- sh -c "gunzip -c /backups/<FILENAME> | grep INSERT"

kubectl delete pod backup-helper-apatel638 -n proj5-apatel638

```



\## 5. Insert a new dated row into the live database



```bash

kubectl exec -it -n proj5-apatel638 deploy/mysql-apatel638 -- mysql -uroot -p'ClO835Pass!apatel638' -e "INSERT INTO clo835db.records (student\_id, note, added\_on) VALUES ('apatel638','<NOTE>','<YYYY-MM-DD>');"

```



\## 6. Run the restore Job against a named backup



Edit `manifests/restore-job.yaml` and change the `BACKUP\_FILE` value. Delete any previous restore Job first (Jobs are immutable):



```bash

kubectl delete job restore-job-apatel638 -n proj5-apatel638 --ignore-not-found

kubectl apply -f manifests/restore-job.yaml

kubectl logs -n proj5-apatel638 job/restore-job-apatel638

```



Verify:



```bash

kubectl exec -it -n proj5-apatel638 deploy/mysql-apatel638 -- mysql -uroot -p'ClO835Pass!apatel638' -e "SELECT \* FROM clo835db.records ORDER BY added\_on;"

```



\## 7. Prove rotation (before / after)



```bash

kubectl apply -f manifests/backup-helper-pod.yaml

sleep 5

MSYS\_NO\_PATHCONV=1 kubectl exec -n proj5-apatel638 backup-helper-apatel638 -- ls -1t /backups

kubectl delete pod backup-helper-apatel638 -n proj5-apatel638

```



Confirm the count stays at 7, oldest file gone, newest file present.



\## 8. Tear down and rebuild if something goes wrong mid-demo



```bash

kind delete cluster --name backup-apatel638

./bootstrap.sh

```

