# Multi Master/Multi NodeのOpenShift環境

次のようなモデル図に沿った環境の構築手順イメージを記載します。

![モデル図](https://github.com/h-kojima/openshift/tree/master/ocp3u3/images/openshift-deployement-model.png)

## 構築手順イメージ

Step1. OpenShiftをインストールする最新版のRHEL7サーバ(物理でも仮想でも可)を用意します。
LB x1台、Master x3台、Infra Node x2台、Node x2台の計8台を用意します。
Master/Nodeの推奨スペックは[こちら](https://access.redhat.com/documentation/en/openshift-container-platform/3.3/single/installation-and-configuration/#install-config-install-prerequisites)をご参照下さい。
LBのスペックはRHEL7のシステム要件(1コア, 2GBメモリー, ディスク容量20GB程度)を満たせば、最低限動作するはずです。

Step2. OpenShiftをインストールするサーバ全台で、OpenShiftのリポジトリ利用を有効にします。

Step3. Infra Node/Nodeの全台で、Dockerサービスを起動します。

```
  # yum -y install docker
  # echo "INSECURE_REGISTRY='--insecure-registry 172.30.0.0/16'" >> /etc/sysconfig/docker
  # systemctl start docker; systemctl enable docker
```

Step4. OpenShiftインストール用に用意されたAnsibleインベントリファイルを[こちら](https://github.com/h-kojima/openshift/tree/master/ocp3u3/ansible/sample-ansible-hosts)からダウンロードします。インベントリファイルで指定しているDNSワイルドカードについては、[こちらのファイル](https://github.com/h-kojima/openshift/tree/master/ocp3u3/bind-chroot)を参考に設定して下さい。

Step5. 適当なサーバで作成したSSH公開鍵を、OpenShiftをインストールするサーバ全台に配布します。

```
  # ssh-keygen
  # ssh-copy-id root@OPENSHIFT_INSTALL_SERVER
```

Step6. Step5.で作成したSSH公開鍵を持つサーバ上で、OpenShiftインストール用に用意されたPlaybookを実行します。

```
  # yum -y install atomic-openshift-utils
  # ansible-playbook -i /root/sample-ansible-hosts /usr/share/ansible/openshift-ansible/playbooks/byo/config.yml
```

Step7. HTPasswd認証用のファイルを作成して、Master全台に配布します。

```
  # yum -y install httpd-tools
  # htpasswd -c /root/htpasswd USERNAME1
  # scp /root/htpasswd root@OPENSHIFT_MASTER_SERVER:/etc/origin/master/
```

Step8. https://lb.example.com:8443 にアクセスするとOpenShiftのログイン画面が表示されるので、
Step7.で作成したユーザ情報を利用してログインし、OpenShift環境を利用できるようになります。

Extra Step. ここまでの手順だとLBが1台構成でSPOFになりますので、必要に応じてKeepAlived + Virtual IPでLBを冗長化して下さい。LBの冗長化手順については[こちら](https://access.redhat.com/documentation/ja-JP/Red_Hat_Enterprise_Linux/7/html/Load_Balancer_Administration/index.html)をご参照下さい。
  
  
  
## Revision History:

2016-11-18  version 0.1 初版リリース
