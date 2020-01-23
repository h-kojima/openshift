# ARO ハンズオン

このコンテンツは、Red Hatが提供している[OpenShiftのハンズオンコンテンツの一部](https://github.com/RedHat-Middleware-Workshops/cloud-native-workshop-v2m1-guides)をベースにしています。他にも[様々なコンテンツ](https://github.com/RedHat-Middleware-Workshops)を提供していますので、興味がありましたらご参照ください。

## 前準備
- ARO環境にログインできていること
- OpenShiftのCLIツールである[ocコマンド](https://mirror.openshift.com/pub/openshift-v3/clients/3.11.154/)をダウンロードして、実行できるようにしていること
  - [Linux](https://mirror.openshift.com/pub/openshift-v3/clients/3.11.154/linux/)/[macOS](https://mirror.openshift.com/pub/openshift-v3/clients/3.11.154/macosx/)/[Windows](https://mirror.openshift.com/pub/openshift-v3/clients/3.11.154/windows/)
- 開発ツールとして利用する、Red Hat Codeready WorkspacesがARO上にデプロイされていること

### Red Hat CodeReady WorkspacesのAROへのデプロイ
開発ツールとして利用する[Red Hat CodeReady Workspaces](https://developers.redhat.com/products/codeready-workspaces/overview)をARO上にデプロイいます。Red Hat CodeReady WorkspacesはEclipse CheをベースにしたクラウドIDE(IntelliJ IDEA、VSCode、Eclipse IDEに類似)であり、本ハンズオンでは、ここからコードを記述、テスト、デプロイします。

Red Hat CodeReady Workspacesのインストール用プログラムを、[Red Hat Developer](https://developers.redhat.com/)からダウンロードします。Red Hat Developerは開発者向けに様々なRed Hatのリソースを提供しているサイトです。こちらに登録してRed Hat Developer Suiteサブスクリプションを取得することで、RHEL, OpenShift, JBoss EAPを始めとした各種ミドルウェア製品をダウンロードできるようになります。**1ユーザ(共用不可)、1台のみ、ソフトウェア開発用途のみ**、と用途は限定されますが、その代わり1年間有効のサブスクリプションとなっていて有効期限が切れたら再度更新することもできます。Red Hat Codeready Workspacesのダウンロードサイトは下記です。

https://developers.redhat.com/products/codeready-workspaces/download



Red Hat CodeReady Workspacesの最新版は2.0までリリースしていますが、本ハンズオンでは、1.0.2 をダウンロードしてください。ダウンロード後にインストーラを解凍して下記を実行します。このコマンドで、ARO上に`workspaces`プロジェクトが自動的に作成され、その中にRed Hat CodeReady Workspacesのコンテナアプリがデプロイされます。

```
$ oc login https://<URL_of_ARO> --token=<token_of_your_ARO>
$ tar xvf codeready-workspaces-1.0.2.GA-operator-installer.tar.gz
$ cd codeready-workspaces-operator-installer
$ ./deploy.sh --deploy \
  --operator-image=registry.redhat.io/codeready-workspaces/server-operator:1.0 \
  --server-image=registry.redhat.io/codeready-workspaces/server-rhel8:1.2
```

CodeReady Workspacesのデプロイが完了したら、`username: admin, password: admin`でログインします。



ログイン後は、作業スペースとなるWorkspaceを作成します。ログインあとは

