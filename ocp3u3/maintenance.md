# OpenShift環境のメンテナンス

## Nodeの追加
[こちら](https://github.com/h-kojima/openshift/blob/master/ocp3u3/ansible/sample-ansible-add-nodes)を参考に
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

Step1. 削除対象のNodeからPodを移動します。  
"--evacuate"でPodが新規作成された後に、該当NodeのPodが削除されます。
```
  # oc login -u system:admin
  # oc adm manage-node $NODE_NAME --schedulable=false
  # oc adm manage-node $NODE_NAME --evacuate 
```

Step2. [こちら](https://github.com/h-kojima/openshift/blob/master/ocp3u3/ansible/sample-ansible-delete-nodes)を参考に
Ansibleインベントリファイルを設定します。  
そして該当Nodeを削除して、Nodeクリーンアップ用のPlaybookを実行します。

```
  # oc delete node $NODE_NAME ("-l zone=zone02"などでラベル指定も可能)
  # ansible-playbook -i /root/sample-delete-nodes /usr/share/ansible/openshift-ansible/playbooks/adhoc/uninstall.yml
```

## Dockerイメージの共有
OpenShiftのDocker Registryを使い、プロジェクト内でカスタムDockerイメージを共有できます。  
ただし、OpenShift環境ではDockerイメージからアプリケーションを起動する際に、セキュリティを考慮してコンテナ内のプロセスのrootユーザでの実行を禁止し、プロセス実行時にはランダムなUIDを割り当てます。そのため、コンテナ内のプロセスは任意の一般ユーザで実行できるようにするために、Dockerfileでの次の設定を忘れないようにしてください。

・USER項目での一般ユーザの指定 (USERを指定しない場合rootユーザでプロセスが実行されます)  
・RUN項目などでの設定/実行ファイルのアクセス権追加 (chmod ugo+rxなど)

Step1. OpenShift環境のDocker RegistryのWeb UIを提供するRegistry ConsoleのURLを確認します。

```
  # oc login -u system:admin
  # oc get route -n default
  … (中略) ...
  NAME                HOST/PORT
  docker-registry     docker-registry-default.cloudapps.com 
  registry-console    registry-console-default.cloudapps.com 
```

Step2. Registry ConsoleのURLにWebブラウザ(上の例では`https://registry-console-default.cloudapps.com`)からアクセスします。
ログインには、OpenShift環境が利用している認証情報と同じものを利用します。

<img src="https://github.com/h-kojima/openshift/blob/master/ocp3u3/images/registry-console.png" width="50%" height="50%">

Step3. 上記画面で確認したLogin/Pushに関する情報を利用して、指定したプロジェクトにDockerイメージをPushします。  
これにより、PushしたDockerイメージを利用してプロジェクト内でアプリケーションをデプロイできるようになります。

```
  # docker login -p … (Registry Consoleで表示されるログインコマンドを実行)
  # docker tag $CUSTOM_IMAGE:latest \
        docker-registry-default.cloudapps.com/$PROJECT_NAME/$CUSTOM_IMAGE:latest
  # docker push docker-registry-default.cloudapps.com/$PROJECT_NAME/$CUSTOM_IMAGE:latest
```

Step4. OpenShift環境の全ユーザが利用できるopenshiftプロジェクトにDockerイメージをPushするには、  
次のコマンドを実行して適当なユーザにopenshiftプロジェクトの管理権限を追加し、  
Step2, Step3 と同じ要領でopenshiftプロジェクトにDockerイメージをPushします。
```
  # oc adm policy add-role-to-user admin $USERNAME -n openshift
```
