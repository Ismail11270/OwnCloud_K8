<br/><br/><br/><br/><br/><br/><br/><br/>
<h1 align=center>Microservice orchestration platforms<br>
Using Kubernetes<br/><br/>Project Report</h1><br/><br/>
<h2 align=center>Implementation of the ownCloud system based on the<br/>Kubernetes cluster</h2>
<br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/>
  <h3>Authors:</h3>
  <ul>
  <li>Ismoil Atajanov</li>
  <li>Brijesh Varsani</li>
  <li>Jiahao Tang</li>
  <li>Maria Vazquez</li>
  </ul>
  <br/><br/><br/>

---
<br/>

## Task description
- The aim of this exercise is to design and deploy production-ready (in terms of reliability,
  security, and efficiency) Kubernetes cluster hosting system of ownCloud services hereinafter
  referred to as system services.
- The system should be composed of software components and publicly available images
  encapsulating services necessary to implement the following features.

## Solution
<h3>Following features have been implemented:</h3>

- [x] system services are available at the dedicated DNS name or at least exposed locally
- [x] all configuration is contained in a dedicated namespace 
- [x] database service is not available from outside the cluster
- [x] data persistence is ensured (i.e. data is stored independently of the system services
container(s))
- [x] system services instances are multiplied to achieve basic availability
- [x] basic cluster monitoring is deployed (e.g. Kuberbetes Dashboard)


---

## Implementation

### Following kubectl commands aliases were set for the sake of simplicity:
| Alias | Command |
| --------- | ---------|
| kget | kubectl get |
| klogs | kubectl logs |
| kpods | kget pods |
| kdelete | kubectl delete |
| kdeletef | kubectl delete -f |
| kapply | kubectl apply -f |
| krestart | kubectl rollout restart |
| kdrestart | krestart deployment |

### Using prepared yaml configuration files, following main kubernetes components have been created:
1. Deployment + service - owncloud
2. StatefulSet + service - mariadb
3. Ingress


### Namespace

All kubernetes components were placed in a new **my-cloud** namespace created from [init.yaml](https://github.com/Ismail11270/AEII_2020_MSK_-Ismoil_Atajanov-/blob/master/owncloud/init.yaml)
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-cloud
  labels:
    name: my-cloud
```
With ease of namespaces management in mind, **kubens** tool was installed and set to **my-cloud** as the default namespace.
<br/>
![kubens](https://github.com/Ismail11270/AEII_2020_MSK_-Ismoil_Atajanov-/blob/master/ss/kubens.png)
<br/>


### Storage

For storing the data NFS Persistence Volume was configured. <br/>
Nfs server was set up locally on the same machine using nfs-kernel-server. 
/private directory was created with 777 access parameters and exported:
<br/>
![etc/exports](https://github.com/Ismail11270/AEII_2020_MSK_-Ismoil_Atajanov-/blob/master/ss/etc-exports.png)
<br/>
Commands to export nfs directory and start nfs server: <br/>
```shell
sudo exportfs -arvf
sudo systemctl start nfs-kernel-server
```
#### Persistence Volume and Persistence Volume Claim
#### Volume configuration file for PV & PVC using NFS Storage [storage.yaml](https://github.com/Ismail11270/AEII_2020_MSK_-Ismoil_Atajanov-/blob/master/owncloud/storage/storage.yaml)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata: 
  name: pv-owncloud
  namespace: my-cloud
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /private
    server: 192.168.0.24
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-owncloud
  namespace: my-cloud
spec:
  storageClassName: nfs
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

### As a result following PV and PVC resources were created:
![pv/pvc](https://github.com/Ismail11270/AEII_2020_MSK_-Ismoil_Atajanov-/blob/master/ss/etc-exports.png)
<br/>
##### Created volume was used further in implementation to mount volumes for the database and owncloud.

### Database configuration
MariaDB was chosen as the main database server, and it was implemented as a single replica stateful application defined in [mariadb.yaml]()
<br/>

- MariaDB username and password were defined in [**mariadb-secret.yaml**](https://github.com/Ismail11270/AEII_2020_MSK_-Ismoil_Atajanov-/blob/master/owncloud/storage/mariadb-secret.yaml)
- MariaDB environment variables were defined in [**config-map.yaml**](https://github.com/Ismail11270/AEII_2020_MSK_-Ismoil_Atajanov-/blob/master/owncloud/storage/config.yaml)
- MariaDB application configuration was defined in [**mariadb.yaml**](https://github.com/Ismail11270/AEII_2020_MSK_-Ismoil_Atajanov-/blob/master/owncloud/storage/mariadb.yaml)
>In order to configure the database correctly **all the configurations** must be applied *in the same order*

####Previously created persistent volume claim is used here to mount volume for mariadb database.
```yaml
        volumeMounts:
        - name: storage
          mountPath: /var/lib/mysql
          subPath: mysql
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: pvc-owncloud
```

### Owncloud configuration
- In order to start only single configuration [owncloud.yaml](https://github.com/Ismail11270/AEII_2020_MSK_-Ismoil_Atajanov-/blob/master/owncloud/owncloud.yaml) is required.
<br/>
  
Most important part of the configuration is the container image which was set to ***owncloud*** and volume mounts suggested by the official 
documentation for the image. Volume mounts are again mounted on the PVC created earlier:
```yaml
        volumeMounts:
        - name: owncloud-storage
          mountPath: /var/www/html/data
          subPath: owncloud/data
        volumeMounts:
        - name: owncloud-storage
          mountPath: /var/www/html/apps
          subPath: owncloud/apps
        volumeMounts:
        - name: owncloud-storage
          mountPath: /var/www/html/config
          subPath: owncloud/config
      volumes:
      - name: owncloud-storage
        persistentVolumeClaim:
          claimName: pvc-owncloud
```
#### The two created applications (deployment and statefulset) were created together with internal services to provide access to the pods
![pv/pvc](https://github.com/Ismail11270/AEII_2020_MSK_-Ismoil_Atajanov-/blob/master/ss/resources.png)
<br/>

### Ingress
Next step of the implementation was to configure nginx ingress to provide external access to owncloud service.
The host for the owncloud service access was set to ***my-cloud.site***. After applying the ingress.yaml configuration owncloud application was exposed at the given host.

- Ingress rules
```yaml
  rules:
  - host: my-cloud.site
    http:
      paths:
      - path: "/"
        pathType: Prefix
        backend:
          service:
            name: owncloud
            port:
              number: 80
```
### Kubernetes Dashboard
Kubernetes dashboard resources come together with minikube installation. To use them a number of minikube addons have to be enabled.
- Below is the list of all minikube addons enabled for this project:
  
![pv/pvc](https://github.com/Ismail11270/AEII_2020_MSK_-Ismoil_Atajanov-/blob/master/ss/addons.png)
<br/>
