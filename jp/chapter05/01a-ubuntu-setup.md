# ESXi上にUbuntuをセットアップする方法

## Ubuntu のダウンロード

[Ubuntuのサイト](https://jp.ubuntu.com/download) から、**Ubuntu Server 22.04 LTS** の最新版をダウンロードします。
<img src="./images/01-ubuntu.png" width="50%">

## Ubuntuイメージを、ESXiにアップロード

**ESXi Host Client**を開き、「**ストレージ**」のメニューに移り、「**データストアブラウザ**」をクリックします。

<img src="./images/02-storage-menu.png" width="100%">

**データストアブラウザ**で、**アップロード**ボタンを押し、先ほどダウンロードしたファイルを選択します。

<img src="./images/03-upload.png" width="100%">

すると、アップロード処理が始まるので、完了までしばらく待ちます。<br>
完了後、以下のように**datastore**に対象ファイルが追加されていることを確認しましょう。

<img src="./images/04-upload-done.png" width="50%">


## VM のセットアップ

続いて、「**仮想マシン**」メニューに移り、「**仮想マシンの作成／登録**」をクリックします。

<img src="./images/05-vm-create.png" width="50%">
<br><br>

「**1. 作成タイプの選択**」では、「**新規仮想マシンの作成**」を選択します。

<img src="./images/06-step1.png" width="50%">
<br><br>

「**2. 名前とゲストOSの選択**」では、以下の情報を入力します。
* **名前** : 任意のVM名を入力します。後で変更することも可能です。
* **互換性など** : 以下の図を参考に選択しましょう。

<img src="./images/07-step2.png" width="70%">
<br><br>

「**3. ストレージの選択**」では、先ほど使用したデータストアが選択されていることを確認し、次に進みます。

<img src="./images/08-step3.png" width="60%">
<br><br>

「**4. 設定のカスタマイズ**」では、以下の情報を入力した後、「**仮想マシンオプション**」をクリックします。（※. まだ「**次へ**」には行かない）

* **CPU** : 「**2**」 にします。
* **メモリ** : 「**4** (最低)」または「**8** (推奨)」の「GB」にします。
* **ハードディスク1** : 「**40** GB」位あると良いでしょう。
* **CD/DVDドライブ1** : 「**データストアISOファイル**」を選択すると、先ほどの様にデータストアブラウザが開くので、事前にアップロードしたファイルを選択します。

<img src="./images/09-step4-01.png" width="100%">
<br><br>

「**仮想マシンオプション**」では、「**VMware Tools**」項目で、「**パワーオン前に毎回 VMware Tools をチェックしてアップグレード**」にチェックを入れておきましょう。その後、「**次へ**」に進みます。

<img src="./images/10-step04-02.png" width="100%">
<br><br>

「**5. 設定の確認**」では、内容に誤りがないことを確認し、「**完了**」を押します。


## Ubuntuのセットアップ

「**仮想マシン**」メニューで「**作成したVM名**」を選択すると、**VMの詳細画面**が表示されます。ここで、「**パワーオン**」をクリックし、VMを起動します。

<img src="./images/11-vm-detail.png" width="100%">
<br><br>


「**画面プレビュー**」（上記のペンギンの絵のあたり）をクリックすると、**コンソール**が起動します。<br>
コンソール画面内をクリックすると、キーボード操作が可能です。<br>
**上下キー**で移動し、**Spaceキー**で選択し、**Enterキー**で決定できます。<br>
ここでは、「**Try or Install Ubuntu Server**」を選びます。

<img src="./images/12-console.png" width="100%">
<br><br>

しばらく待つと、Ubuntuの設定画面に変わります。<br>
最初は、**言語の選択**です。Server用途で使うので、そのまま「**English**」にしましょう。

<img src="./images/13-language.png" width="100%">
<br><br>

もしDHCP等により、既にネットワークが自動構成されている場合は、ここで新たなバージョンを利用するか聞かれます。<br>
ここでは、「**Continue without updating**」とします。

<img src="./images/14-update-info.png" width="100%">
<br><br>

次は**キーボードレイアウト**です。お使いのキーボードレイアウトに合わせて選択して下さい。

<img src="./images/15-key-layout.png" width="100%">
<br><br>

**インストールタイプ**を選択できます。デフォルトの「**Ubuntu Server**」で問題ありません。

<img src="./images/16-install-type.png" width="100%">
<br><br>

**ネットワーク設定**では、お使いの状況に合わせて設定し、次に進みます。
- **DHCP** : 利用可能な場合は、既にIPアドレスが表示されているはずなので、そのまま次に進みます。
- **マニュアル** : 「**ens34**」→「**Edit IPv4**」と選択し、「**IPv4 Method**」を「**Manual**」にすると、手動設定することができます。

<img src="./images/17-network-settings.png" width="100%">
<br><br>

**Proxy設定**。直接インターネットに接続できない環境の場合のみ、設定が必要です。

<img src="./images/18-proxy.png" width="100%">
<br><br>

**Ubuntuパッケージのミラーサイト設定**。「**This mirror location passed tests**」を確認し、次に進みます。<br>
必要に応じて、別のミラーサイトを設定することも可能です。

<img src="./images/19-mirror.png" width="100%">
<br><br>

**ストレージ設定**。必要に応じて、**LVM**や**暗号化**などを設定できますが、そのまま次に進みます。

<img src="./images/20-storage.png" width="100%">
<br><br>

**ストレージ設定の確認画面**。先に進むと、確認ウィンドウが出るので、「**Confirm**」を選択。

<img src="./images/21-storage-confirm.png" width="100%">
<br><br>

**プロファイル設定**。<br>
**あなたの名前**と、**サーバ名**と、ログイン時に使用する**ユーザ名**および**パスワード**を登録します。

<img src="./images/22-user.png" width="100%">
<br><br>

**Ubuntu Proサブスクリプション**設定。何も入力しなくても、先に進めます。

<img src="./images/23-ubuntu-pro.png" width="100%">
<br><br>

**SSH設定**。これは、セットアップ後のリモートログインが容易になるので、**結構重要**です。
* **Install OpenSSH server** : チェック(**X**)を入れる。
* **Import SSH identify** : 事前に**自分のPCの公開鍵**(SSH Public key)を**GitHub**に登録している場合、<br>「**from GitHub**」にすると、その鍵を流用してログインできるようになります。
* **GitHub username** : 上記の設定を行う場合、自分の**GitHubアカウント**を入力します。
* **Allow password authentication over SSH** : チェック(**X**)を入れる。

<img src="./images/24-ssh-setup.png" width="100%">
<br><br>

**追加パッケージの選択**。後で必要なものは個別で入れるので、基本は必要ありません。<br>
次に進むと、インストールが始まります。

<img src="./images/25-add-package.png" width="100%">
<br><br>

インストール完了後は、「**Reboot Now**」を選び、再起動します。

<img src="./images/26-confirm.png" width="100%">
<br><br>

※. 以下のように、「**Failed unmounting /cdrom.**」のメッセージが出る場合、対応が必要です。<br>
1. 「**ESXi Host Client**」の**VM詳細画面**から、「**パワーオフ**」で電源を止めます。
2. 「**編集**」で、「**CD/DVDドライブ1**」を「**ホストデバイス**」にします。
3. 再度、「**パワーオン**」します。

<img src="./images/27-failed-unmount.png" width="100%">
<img src="./images/28-cd-drive.png" width="100%">
<br><br>

「**(サーバ名) login:**」のプロンプトが表示されれば、セットアップ完了です。コンソールを閉じて下さい。

<img src="./images/29-login.png" width="50%">
<br><br>

## Ubuntuへのアクセス

セットアップ中にSSH設定を行った場合、`ssh (ユーザ名)@(サーバ名)` で、SSHログインできます。<br>
今回のセットアップ例の場合、以下のようになります。

```
$ ssh ubuntu@k8s-node1
```

ログインに成功したら、セットアップ完了です。お疲れ様でした！