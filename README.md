Name: Amir Mamdouh

# Kubernetes Wiki + MySQL Deployment

This project sets up a MySQL StatefulSet and a Wiki application Deployment on Kubernetes. It includes persistent storage, service exposure, configuration, secrets management, and backup/restore functionality.

- StatefulSet for MySQL
- Deployment for Wiki.js
- Persistent Volumes (PV) and Persistent Volume Claims (PVC)
- Internal Services for communication
- ConfigMap and Secret for DB credentials
- Backup and restore Jobs using `mysqldump`

##  Components

### 1. **MySQL (StatefulSet)**

- **YAML:** `mysql.yml`
- **Persistent Volume Claim:** `mysql-pvc.yml`
- **Persistent Volume:** `mysql-pv.yml`
- **Service:** `mysql-service.yml`

MySQL is deployed as a **StatefulSet** to ensure stable network identity and persistent storage for database data.

### 2. **Wiki (Deployment)**

- **YAML:** `wiki.yml`
- **Service:** `wiki-service.yml`
- **ConfigMap:** `wiki-configmap.yml`
- **Secrets:** `wiki-secrets.yml`

The Wiki app connects to MySQL using environment variables defined in the ConfigMap and Secrets.

### 3. **Storage**

- **MySQL Data Storage**
  - PVC: `mysql-pvc.yml`
  - PV: `mysql-pv.yml`
- **Backup Storage**
  - PVC: `mysql-backup-pvc.yml`
  - PV: `mysql-backup-pv.yml`

### 4. **Backup & Restore Jobs**

- **Backup Job:** `mysql-backupjob.yml`
  - Periodically backs up MySQL data to the backup volume.
- **Restore Job:** `mysql-restorejob.yml`
  - Used to restore MySQL data from the backup volume.

>  All persistent volumes are configured using hostPath (for local development) or suitable alternatives for cloud-based clusters.

---

##  Usage

### 1. Deploy Storage

```bash
kubectl apply -f mysql-pv.yml
kubectl apply -f mysql-pvc.yml

kubectl apply -f mysql-backup-pv.yml
kubectl apply -f mysql-backup-pvc.yml
```

### 2. Deploy MySQL

```bash
kubectl apply -f mysql-service.yml
kubectl apply -f mysql.yml
```

### 3. Deploy Wiki

```bash
kubectl apply -f wiki-configmap.yml
kubectl apply -f wiki-secrets.yml
kubectl apply -f wiki-service.yml
kubectl apply -f wiki.yml
```

### 4. Run Backup Job

```bash
kubectl apply -f mysql-backupjob.yml
```

### 5. Run Restore Job

```bash
kubectl apply -f mysql-restorejob.yml
```

---

##  Directory Structure

```plaintext
task3/
│
├── mysql.yml                # MySQL StatefulSet
├── mysql-service.yml        # MySQL Service
├── mysql-pvc.yml            # MySQL PersistentVolumeClaim
├── mysql-pv.yml             # MySQL PersistentVolume
│
├── wiki.yml                 # Wiki Deployment
├── wiki-service.yml         # Wiki Service
├── wiki-configmap.yml       # Wiki Configuration
├── wiki-secrets.yml         # Wiki Secrets (e.g., DB password)
│
├── mysql-backupjob.yml      # CronJob or Job for backing up MySQL
├── mysql-restorejob.yml     # Job to restore MySQL from backup
├── mysql-backup-pv.yml      # Backup Volume
├── mysql-backup-pvc.yml     # Backup VolumeClaim
│
├── task3-doc.odt            # Project documentation (ODT)
└── task3-doc.pdf            # Project documentation (PDF)
```

---


## Kubernetes Manifests

### `mysql.yml` – MySQL StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 1
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:5.7
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: rootpassword
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mysql-storage
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-storage
          persistentVolumeClaim:
            claimName: mysql-pvc
````

---

### `mysql-service.yml` – MySQL Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
    - port: 3306
  selector:
    app: mysql
  clusterIP: None
```

---

### `mysql-pv.yml` – MySQL PersistentVolume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data/mysql"
```

---

### `mysql-pvc.yml` – MySQL PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

---

### `wiki.yml` – Wiki.js Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wiki
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wiki
  template:
    metadata:
      labels:
        app: wiki
    spec:
      containers:
        - name: wiki
          image: ghcr.io/linuxserver/wikijs
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: wiki-config
            - secretRef:
                name: wiki-secrets
```

---

### `wiki-service.yml` – Wiki.js ClusterIP Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wiki
spec:
  selector:
    app: wiki
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP
```

---

### `wiki-configmap.yml` – ConfigMap for DB Settings

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: wiki-config
data:
  DB_TYPE: mysql
  DB_HOST: mysql
  DB_PORT: "3306"
  DB_USER: root
```

---

### `wiki-secrets.yml` – Secret for DB Password

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: wiki-secrets
type: Opaque
data:
  DB_PASS: cm9vdHBhc3N3b3Jk
```

>  The value `cm9vdHBhc3N3b3Jk` is base64 for `rootpassword`.

---

### `mysql-backup-pv.yml` – Backup PersistentVolume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-backup-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data/mysql-backup"
```

---

### `mysql-backup-pvc.yml` – Backup PersistentVolumeClaim

```yaml
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

---

### `mysql-backupjob.yml` – Job to Backup MySQL

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: mysql-backup
spec:
  template:
    spec:
      containers:
        - name: backup
          image: mysql:5.7
          command: ["/bin/sh", "-c"]
          args: ["mysqldump -h mysql -uroot -prootpassword --all-databases > /backup/all.sql"]
          volumeMounts:
            - name: backup-storage
              mountPath: /backup
      restartPolicy: OnFailure
      volumes:
        - name: backup-storage
          persistentVolumeClaim:
            claimName: mysql-backup-pvc
```

---

### `mysql-restorejob.yml` – Job to Restore MySQL

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: mysql-restore
spec:
  template:
    spec:
      containers:
        - name: restore
          image: mysql:5.7
          command: ["/bin/sh", "-c"]
          args: ["mysql -h mysql -uroot -prootpassword < /backup/all.sql"]
          volumeMounts:
            - name: backup-storage
              mountPath: /backup
      restartPolicy: OnFailure
      volumes:
        - name: backup-storage
          persistentVolumeClaim:
            claimName: mysql-backup-pvc
```

---

## How to Deploy

1. **Create volumes**:

   ```bash
   kubectl apply -f mysql-pv.yml
   kubectl apply -f mysql-pvc.yml
   kubectl apply -f mysql-backup-pv.yml
   kubectl apply -f mysql-backup-pvc.yml
   ```

2. **Deploy MySQL**:

   ```bash
   kubectl apply -f mysql.yml
   kubectl apply -f mysql-service.yml
   ```

3. **Deploy Wiki.js**:

   ```bash
   kubectl apply -f wiki-configmap.yml
   kubectl apply -f wiki-secrets.yml
   kubectl apply -f wiki.yml
   kubectl apply -f wiki-service.yml
   ```

4. **Run Backup Job (optional)**:

   ```bash
   kubectl apply -f mysql-backupjob.yml
   ```

5. **Run Restore Job (optional)**:

   ```bash
   kubectl apply -f mysql-restorejob.yml
   ```

## Test taking backups and restoring
1. **get pods**
* amir pod is for debugging…



2. **open wiki URL on minikube_IP:30001  →    pod_ID:3000**


3. **Apply the mysql-backjob.yml job  and check the backup we took right now**


4. **Check for users inside the mysql database itself**


5. **Delete wiki , mysql pods , PVC and PV  and recreate them.**


We deleted everything!!!


6. **run the restore job and check for data**

* zerosploit and halwagy users exist..

---

##  Notes

* All volumes use `hostPath`, so ensure `/mnt/data/...` directories exist on all cluster nodes.
* Backup job will save `all.sql` inside the backup volume.
* PVC and PV to make them bounded you need to make sure for three things:
1- Same StorageClassName
2- PVC storage <= PV capacity 
3- Access Modes are the same
* Secrets must be base64 encoded. Example:

  ```bash
  echo -n 'rootpassword' | base64
  ```





















