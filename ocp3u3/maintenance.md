## Nodeの追加
[こちら](https://github.com/h-kojima/openshift/blob/master/ocp3u3/ansible/sample-ansible-hosts)を参考に
Ansibleインベントリファイルを設定します。そしてNode追加用のPlaybookを実行します。

```
  # ansible-playbook -i /root/sample-add-nodes /usr/share/ansible/openshift-ansible/playbooks/byo/openshift-node/scaleup.yml
```

Infra Nodeを追加しても、「oc get pods -n default」で確認できるRouter Podの数が増えていない場合は、Router Podのレプリカ数を増やして下さい。

```
  # oc login -u system:admin
  # oc scale rc router-1 --replicas=$REPLICA_NUMBER -n default
```


## Nodeの削除

Step1. 削除対象のNodeからPodを移動します。"--evacuate"でPodが新規作成された後に、該当NodeのPodが削除されます。
```
  # oc adm manage-node $NODE_NAME --schedulable=false
  # oc adm manage-node $NODE_NAME --evacuate 
```

Step2. [こちら](https://github.com/h-kojima/openshift/blob/master/ocp3u3/ansible/sample-ansible-hosts)を参考に
Ansibleインベントリファイルを設定します。そして該当Nodeを削除して、Nodeクリーンアップ用のPlaybookを実行します。

```
  # oc delete node $NODE_NAME ("-l zone=zone02"などでラベル指定も可能)
  # ansible-playbook -i /root/sample-delete-nodes /usr/share/ansible/openshift-ansible/playbooks/adhoc/uninstall.yml
```
