# Running dask-kubernetes to deploy scalable dask-centos nodes on k8s cluster

## Setup

```bash
sudo yum install python36
sudo yum install python36-devel
sudo python36 -m ensurepip

python36 -m venv venv
. venv/bin/activate
pip install dask-kubernetes ipython
```

## Create a file `worker-spec.yml` with the following content

```yaml
kind: Pod
metadata:
  generateName: foo-
  labels:
    foo: bar
spec:
  restartPolicy: Never
  containers:
  - image: daskdev/dask:latest
    args: [dask-worker, --nthreads, '2', --no-bokeh, --memory-limit, 1GB, --death-timeout, '60']
    name: foo-container
    env:
      - name: EXTRA_PIP_PACKAGES
        value: fastparquet git+https://github.com/dask/distributed
    resources:
      limits:
        cpu: "1"
        memory: 1G
      requests:
        cpu: "1"
        memory: 1G
```

This worker spec is somewhat based on: http://dask-kubernetes.readthedocs.io/en/latest/.

## Create a kubernetes config file at `~/.kube/config`

For an ansible role that automatically creates an admin user, look at: https://github.com/stackhpc/k8s-inception-ansible. It looks somewhat as follows:
e

```yaml
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: https://10.233.0.1:443
  name: internal
contexts:
- context:
    cluster: internal
    namespace: default
    user: admin-user
  name: external
- context:
    cluster: internal
    namespace: default
    user: admin-user
  name: internal
current-context: internal
kind: Config
preferences: {}
users:
- name: admin-user
  user:
    as-user-extra: {}
    token: eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLThwY2Q2Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJlYzliMDA5Zi0yYWNjLTExZTgtOTMwYi01MjU0MDBhZDNiNDMiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.i7UuaLAuUi2N9iwBY6Z1J2cXt0h1tT_qsKU3fJxlUrm-IKsjnAyHPSHtNmERa_JdWoWc1eT7f3irOub9WSzqLnTMH-4Gb4pcZiSaneifHNcCJCeWUJxfXKSiWNmjGiCAqIicAd14OQOj4HnSpSW9ng6FrD0O1BU8aDWjUio-VpDGfBPxZCbCB0YsZKB2rFMsEaPFl2m-Swzk2xb6z9kitH1xWdWrX-_oVAZO5kiwV53vOtmlNlJtzH7ozgzq1lwhDMClGOW0WpR6Dnk1QTEGZpi9w0MaHe4KNEv0U8tHh1C66nDbsKGpQMKbShubmsJLb0y1LT5QfBid_pa6YAUY0w
```

## Inside `ipython`:

```python
from dask_kubernetes import KubeCluster
cluster = KubeCluster.from_yaml('worker-spec.yml')
cluster.scale(2)
!kubectl get pods
```

You should see a list of a pods that have been created. To scale up:

```python
cluster.scale_up(3)
!kubectl get pods
```

To cleanup:

```python
cluster.close()
```

