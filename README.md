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

# 使用 images 區塊統一管理版本
images:
  # 修改 ECK Operator 版本
  - name: docker.elastic.co/eck/eck-operator
    newName: docker.elastic.co/eck/eck-operator
    newTag: 2.16.1

  # 修改 Elasticsearch 版本
  - name: docker.elastic.co/elasticsearch/elasticsearch
    newName: docker.elastic.co/elasticsearch/elasticsearch
    newTag: 8.17.0

  # 修改 Kibana 版本
  - name: docker.elastic.co/kibana/kibana
    newName: docker.elastic.co/kibana/kibana
    newTag: 8.17.0
patches:
- patch: |-
    - op: replace
      path: /spec/nodeSets/0/volumeClaimTemplates/0/spec/storageClassName
      value: nfs-delete              # 修改 csi 名稱
  target:
    kind: Elasticsearch
    name: elasticsearch
- patch: |-
    - op: replace
      path: /spec/nodeSets/1/volumeClaimTemplates/0/spec/storageClassName
      value: nfs-delete              # 修改 csi 名稱
  target:
    kind: Elasticsearch
    name: elasticsearch

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
