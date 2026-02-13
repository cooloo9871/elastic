## 下載 yaml

```
$ git clone https://github.com/cooloo9871/elastic.git; cd elastic/
```

```
$ ls -l
total 28
drwxrwxr-x 3 bigred bigred  4096 Sep  9 15:11 base
drwxrwxr-x 3 bigred bigred  4096 Sep  9 11:27 env
-rw-rw-r-- 1 bigred bigred   133 Sep  9 11:27 img.list
-rw-rw-r-- 1 bigred bigred 11357 Sep  9 11:27 LICENSE
-rw-rw-r-- 1 bigred bigred    75 Sep  9 15:22 README.md
```


## 部署

* 環境已安裝好 nfs csi
```
$ kubectl get sc
NAME         PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-delete   nfs.csi.k8s.io   Delete          Immediate           false                  25d
```

* 修改 `kustomiz` 設定自己的環境
```
$ nano env/test/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base

configurations:
- kustomizeconfig.yaml

commonLabels:
  # 根據 ECK Operator 版本修改一致
  app.kubernetes.io/version: "3.3.0"

# 使用 images 區塊統一管理版本
images:
  # 修改 ECK Operator 版本
  - name: docker.elastic.co/eck/eck-operator
    newName: docker.elastic.co/eck/eck-operator
    newTag: 3.3.0

  # 修改 Elasticsearch 版本
  - name: docker.elastic.co/elasticsearch/elasticsearch:8.16.1
    newName: docker.elastic.co/elasticsearch/elasticsearch
    newTag: 9.3.0

  # 修改 Kibana 版本
  - name: docker.elastic.co/kibana/kibana:8.16.1
    newName: docker.elastic.co/kibana/kibana
    newTag: 9.3.0

patches:
- patch: |-
    - op: replace
      path: /spec/version
      # 根據 Elasticsearch 版本修改一致
      value: 9.3.0
  target:
    kind: Elasticsearch
    name: elasticsearch
- patch: |-
    - op: replace
      path: /spec/nodeSets/0/volumeClaimTemplates/0/spec/storageClassName
      value: nfs-csi
  target:
    kind: Elasticsearch
    name: elasticsearch
- patch: |-
    - op: replace
      path: /spec/nodeSets/1/volumeClaimTemplates/0/spec/storageClassName
      value: nfs-csi
  target:
    kind: Elasticsearch
    name: elasticsearch

- target:
    group: elasticsearch.k8s.elastic.co
    version: v1
    kind: Elasticsearch
    name: elasticsearch
  path: elastic-tolerations.json

patchesStrategicMerge:
  - kibana-patch.yaml
```

* 先部署 CRD
```
$ kubectl apply -f base/resource/crds.yaml
```

* 透過 `kustomiz` 部署

```
$ kubectl apply -k env/test
```

## 環境檢查

```
$ kubectl -n elastic-system get pod
NAME                 READY   STATUS    RESTARTS   AGE
elastic-operator-0   1/1     Running   0          12m
```
```
$ kubectl -n elastic get pod
NAME                              READY   STATUS    RESTARTS   AGE
elasticsearch-es-data-nodes-0     1/1     Running   0          12m
elasticsearch-es-data-nodes-1     1/1     Running   0          12m
elasticsearch-es-data-nodes-2     1/1     Running   0          12m
elasticsearch-es-master-nodes-0   1/1     Running   0          12m
elasticsearch-es-master-nodes-1   1/1     Running   0          12m
elasticsearch-es-master-nodes-2   1/1     Running   0          12m
kibana-kb-54fcdb8d77-thbcq        1/1     Running   0          12m
```
