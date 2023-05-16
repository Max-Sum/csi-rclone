
# CSI rclone mount plugin

This project implements Container Storage Interface (CSI) plugin that allows using [rclone mount](https://rclone.org/) as storage backend. Rclone mount points and [parameters](https://rclone.org/commands/rclone_mount/) can be configured using Secret. 

## Kubernetes cluster compatability
Works (tested):
- `deploy/kubernetes/1.19`: K8S>= 1.19.x (due to storage.k8s.io/v1 CSIDriver API)

Does not work:
- v1.12.7-gke.10, driver name csi-rclone not found in the list of registered CSI drivers

## Installing CSI driver to kubernetes cluster
TLDR: Edit `deploy/kubernetes/1.19/csi-rclone-storageclass.yaml` and `deploy/kubernetes/1.19/csi-rclone-secret.yaml`. Then run` kubectl apply -f deploy/kubernetes/1.19`.

1. Set up storage backend. You can use [Minio](https://min.io/), Amazon S3 compatible cloud storage service.
i.e. 
```
helm upgrade --install --create-namespace --namespace minio minio minio/minio --version 6.0.5 --set resources.requests.memory=512Mi --set secretKey=SECRET_ACCESS_KEY --set accessKey=ACCESS_KEY_ID
```

2. Configure the backend in `csi-rclone-storageclass.yaml` and `csi-rclone-secret.yaml`.

csi-rclone-storageclass.yaml
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rclone
  namespace: csi-rclone
provisioner: csi-rclone
parameters:
  remote: "minio"
  path: "rclone-kubernetes"
  csi.storage.k8s.io/provisioner-secret-name: rclone-secret
  csi.storage.k8s.io/provisioner-secret-namespace: csi-rclone
  csi.storage.k8s.io/node-publish-secret-name: rclone-secret
  csi.storage.k8s.io/node-publish-secret-namespace: csi-rclone
```

csi-rclone-secret.yaml
```
apiVersion: v1
kind: Secret
metadata:
  name: rclone-secret
  namespace: csi-rclone
type: Opaque
stringData:
  rclone.conf: |
    [minio]
    type = s3
    provider = Minio
    access_key_id = ACCESS_KEY_ID
    secret_access_key = SECRET_ACCESS_KEY
    endpoint = http://minio-release.default:9000
```

3. Deploy by `kubectl apply -f deploy/kubernetes/1.19`

Optionally, you can configure more than 1 dynamic provisioner by editing and pushing `csi-rclone-storageclass.yaml` and its secrets.

Deploy example definition
> `kubectl apply -f example/kubernetes/nginx-example.yaml`


## Building plugin and creating image
Current code is referencing projects repository on github.com. If you fork the repository, you have to change go includes in several places (use search and replace).


1. First push the changed code to remote. The build will use paths from `pkg/` directory.

2. Build the plugin
```
make plugin
```

3. Build the container and inject the plugin into it.
```
make container
```

4. Change docker.io account in `Makefile` and use `make push` to push the image to remote. 
``` 
make push
```
## Changelog

See [CHANGELOG.txt](CHANGELOG.txt)
