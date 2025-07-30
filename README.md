# K8s-mysql-backup-using-cronjob


## 🎯 Goal

Take regular backups of your MySQL database running in Kubernetes using a CronJob, and save those dumps inside a persistent volume.

---

## 🧱 Environment Assumptions

* MySQL is running in a Pod accessible via a service (let’s say `mysql-service`).
* Your database username is `root` and password is `your_password`.
* You are using MySQL 5.7 or compatible.

---

## 🪜 Step-by-Step Guide

---

### ✅ Step 1: Create a Secret for MySQL Credentials

This keeps your credentials secure rather than hardcoding them.

```yaml
# mysql-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
stringData:
  MYSQL_USER: root
  MYSQL_PASSWORD: your_password
```

Apply it:

```bash
kubectl apply -f mysql-secret.yaml
```

---

### ✅ Step 2: Create a PersistentVolumeClaim for Backups

This storage will hold the backup `.sql` files.

```yaml
# mysql-backup-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-backup-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Apply it:

```bash
kubectl apply -f mysql-backup-pvc.yaml
```

> Note: Make sure your cluster supports `ReadWriteOnce` volumes. On Minikube or EKS, this will work fine with the default storage class.

---

### ✅ Step 3: Create the CronJob

This job will run daily at 2 AM and dump all databases into `/backup` directory in PVC.

```yaml
# mysql-backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mysql-backup
spec:
  schedule: "0 2 * * *"  # Runs daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: mysql-backup
            image: mysql:5.7  # Use your actual version
            envFrom:
              - secretRef:
                  name: mysql-secret
            command:
              - /bin/sh
              - -c
              - |
                mysqldump -h mysql-service -u$MYSQL_USER -p$MYSQL_PASSWORD --all-databases > /backup/all-databases-$(date +%F-%H-%M-%S).sql
            volumeMounts:
              - name: backup-storage
                mountPath: /backup
          restartPolicy: OnFailure
          volumes:
            - name: backup-storage
              persistentVolumeClaim:
                claimName: mysql-backup-pvc
```

Apply it:

```bash
kubectl apply -f mysql-backup-cronjob.yaml
```

---

### ✅ Step 4: Verify CronJob

Check if the CronJob was created:

```bash
kubectl get cronjobs
```

After the scheduled time (2 AM), it will automatically run.

To manually trigger it now:

```bash
kubectl create job --from=cronjob/mysql-backup mysql-backup-manual
```

Check if the Job completed:

```bash
kubectl get jobs
kubectl get pods
kubectl logs <pod-name>
```

---

### ✅ Step 5: Check the Backup Files

To inspect the contents of the PVC (where `/backup/*.sql` is stored), you can create a temporary Pod that mounts the same PVC:

```yaml
# debug-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: backup-debug
spec:
  containers:
  - name: shell
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
      - name: backup-storage
        mountPath: /backup
  volumes:
    - name: backup-storage
      persistentVolumeClaim:
        claimName: mysql-backup-pvc
  restartPolicy: Never
```

```bash
kubectl apply -f debug-pod.yaml
kubectl exec -it backup-debug -- sh
ls /backup
```

You should see your `.sql` backup files there.

---

