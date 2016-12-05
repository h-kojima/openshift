# Multi Master/Multi NodeのOpenShift環境

次のようなモデル図に沿った環境の構築手順イメージを記載します。作成した環境のメンテナンス(Nodeの追加など)については、[こちら](https://github.com/h-kojima/openshift/blob/master/ocp3u3/maintenance.md)に記載します。

![モデル図](https://github.com/h-kojima/openshift/blob/master/ocp3u3/images/openshift-deployment-model.png)

## version3.3の構築手順イメージ

Step1. OpenShiftをインストールする最新版のRHEL7サーバ(物理でも仮想でも可)を用意します。
LB x1台、Master x3台、Infra Node x2台、Node x2台の計8台を用意します。
Master/Nodeの推奨スペックは[こちら](https://access.redhat.com/documentation/en/openshift-container-platform/3.3/single/installation-and-configuration/#install-config-install-prerequisites)をご参照下さい。
LBのスペックはRHEL7のシステム要件(1コア, 2GBメモリー, ディスク容量20GB程度)を満たせば、最低限動作するはずです。

Step2. OpenShiftをインストールするサーバ全台で、OpenShiftのリポジトリ利用を有効にします。

Step3. Master/Infra Node/Nodeの全台で、Dockerサービスを起動します。本番環境を想定する場合、Dockerのイメージ領域として未使用のディスク領域が必要となります。以下の「sdb」はシステム毎に、「vdb」や「nvme1n1」などに置き換えて下さい。

```
  # yum -y install docker
  # echo "INSECURE_REGISTRY='--insecure-registry 172.30.0.0/16'" >> /etc/sysconfig/docker
  # cat <<EOF >> /etc/sysconfig/docker-storage-setup
  DEVS=/dev/sdb
  VG=docker-vg
  EOF
  # docker-storage-setup
  # systemctl start docker; systemctl enable docker
```

Step4. OpenShiftインストール用に用意されたAnsibleインベントリファイルを[こちら](https://github.com/h-kojima/openshift/blob/master/ocp3u3/ansible/sample-ansible-hosts)からダウンロードします。この時、Docker Registryの共有ストレージとして利用するNFSや、ホスト名、アプリケーションのドメイン名などは適宜修正して下さい。インベントリファイルで指定しているDNSワイルドカードについては、[こちらのファイル](https://github.com/h-kojima/openshift/blob/master/ocp3u3/bind-chroot)を参考に設定して下さい。

Step5. 適当なサーバで作成したSSH公開鍵を、OpenShiftをインストールするサーバ全台に配布します。

```
  # ssh-keygen -f /root/.ssh/id_rsa -N ''
  # ssh-copy-id root@$OPENSHIFT_INSTALL_SERVER
```

Step6. Step5.のssh-keygenを実行したサーバで、OpenShiftインストール用に用意されたPlaybookを実行します。

```
  # yum -y install atomic-openshift-utils
  # ansible-playbook -i /root/sample-ansible-hosts /usr/share/ansible/openshift-ansible/playbooks/byo/config.yml
```

Step7. HTPasswd認証用のファイルを作成して、Master全台に配布します。

```
  # yum -y install httpd-tools
  # htpasswd -c /root/htpasswd $USERNAME1
  # scp /root/htpasswd root@$OPENSHIFT_MASTER_SERVER:/etc/origin/master/
```
Step8. Infra NodeへのアプリケーションPodの配置を無効化します。次のコマンドを任意のMasterサーバ上で実行します。

```
  # oc login -u system:admin
  # oadm manage-node $INFRA_NODE1 $INFRA_NODE2 --schedulable=false
```

Step9. https://LB_SERVER_FQDN:8443 にアクセスするとOpenShiftのログイン画面が表示されるので、
Step7.で作成したユーザ情報を利用してログインし、OpenShift環境を利用できるようになります。

なお、アプリケーションへのルーティングには、デフォルトのRouter(HAProxy) Pod以外にもF5のルータを利用することもできます。その場合、Router Podの起動が必要なくなるので、Infra Nodeを構築する必要がなくなります。F5ルータの利用手順の詳細は[こちら](https://access.redhat.com/documentation/en/openshift-container-platform/3.3/single/installation-and-configuration/#install-config-router-f5)をご参照下さい。

### Extra Step

Step10. ここまでの手順だとLBが1台構成でSPOFになります。そこで、KeepAlivedでHAProxyサービスを簡易的に冗長化します。まず、新しいLBとなるRHEL7サーバを2台(Master1台、Backup1台の計2台構成)用意し、必要なパッケージをインストールします。

```
  # yum -y install keepalived haproxy iptables-services
```
Step11. 既存のLBのhaproxy/iptablesサービスの設定ファイルを、新しいLB全台にコピーします。

```
  # scp /etc/haproxy/haproxy.cfg root@$OPENSHIFT_NEW_LB_SERVER:/etc/haproxy/
  # scp /etc/sysconfig/iptables root@$OPENSHIFT_NEW_LB_SERVER:/etc/sysconfig/
```
Step12. [こちら](https://github.com/h-kojima/openshift/blob/master/ocp3u3/keepalived/keepalived.conf)からダウンロードしたkeepalived.confを、新しいLBの/etc/keepalived/に保存します。この時、設定ファイルのコメントを参考にして、「state, priority, unicast_peer, virtual_ipaddress」の4項目を適宜修正して下さい。

Step13. 既存LBの電源を落とします。そして、新しいLB全台で各サービスを起動・有効化します。この時、iptablesサービスを起動しますので、firewalldサービスを停止しておきます。

```
  # systemctl stop firewalld; systemctl disable firewalld
  # systemctl start haproxy; systemctl start keepalived; systemctl start iptables
  # systemctl enable haproxy; systemctl enable keepalived; systemctl enable iptables
```
Step14. MasterとなるLBで、ipコマンドなどで仮想IPアドレスが割り当てられていることを確認できます。また、このkeepalivedの設定ではHAProxyのプロセスが起動しているかどうかを見ているため、「# systemctl stop haproxy」などでHAProxyを停止すると、BackupとなるLBに仮想IPアドレスが引き継がれることを確認できます。

## Revision History:

2016-12-05 メンテナンスのリンクを追加  
2016-11-19 初版リリース
