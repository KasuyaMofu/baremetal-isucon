# ISUCON10予選 #ISUCON の環境をUbuntu 18.04 LTSの2台構成(ベンチマーカー+競技用サーバ)で構築する

こんにちは、糟屋もふです。先日 Virtual ISUCON Live に出演したVTuberです。かすやもふという名前だけでも今回は覚えて帰ってください。
以下、である調で記載します。

## はじめに

このお盆休みにISUCONの問題に取り組む人は多いと思う。しかし、ISUCONの環境を構築できない場合も多いはずだ。特に、ISUCONの過去問にチャレンジする方法はAWSを使用する、vagrantを利用するなど、いくつか方法が紹介されているが、オンプレミスの環境に構築している記事は公式にはない。

[ISUCONの過去問にチャレンジするためのシンプルな環境構築](https://isucon.net/archives/54946542.html)

私は、残念ながらケチなのでAWSを使えず、かつvagrantをベンチマーカー/競技用サーバの2台起動できるようなスペックのPCを持ち合わせていないので困ったところである。

そんな中、偶然にも **6 CPUs x Intel(R) Core(TM) i5-10400 CPU @ 2.90GHz、メモリ16GB、512GB NVMe/64GB SSDが搭載されているVostro 3681があった**ので、これにESXi 6.7をインストールして、以下の構成で作成する。

- `isubench`: ベンチマーカー。4CPU、メモリ4GB、100GB HDD
- `isu1`: 競技用サーバ。4CPU、メモリ4GB、100GB HDD

本質的にはESXi上のVMとオンプレミス環境は変わらず、DHCPを用いたLANへの接続、及びインターネット接続環境がOSから認識出来れば、以下の手順は実行できるはずなので、参考にしていただけると嬉しい。


それでは、以下のリポジトリをベースに進めていきましょう。

[isucon/isucon10-qualify](https://github.com/isucon/isucon10-qualify)


また、以下の手順について、動画にもしておりますので、詰まった場合などは併せてご確認ください。


[YouTube Link](https://www.youtube.com/watch?v=drcquHgGTDo)




## OSのセットアップ

[https://releases.ubuntu.com/18.04.5/] から **Server install image** ([ubuntu-18.04.5-live-server-amd64.iso](https://releases.ubuntu.com/18.04.5/ubuntu-18.04.5-live-server-amd64.iso)) をダウンロード、及びインストールを行う。

特に構成によって手順が変わることは無いと思うが、ホスト名は `isubench` および `isu1` とし、`OpenSSH server`のインストールは行っているものとして、以下の手順は記載する。この際、`isubench` および  `isu1` のIPは控えておくと導入がスムーズになる。

## 環境構築

OSのインストール後、以下は、特に断らない限り、 **isubench** 上でのコマンド実行である。

### アップデート

まずはアップデートを行う。

```bash
sudo apt update
sudo apt upgrade -y
```

### Visual Studio Code用の設定

ファイルを追跡する関連でエラーが出るので、 `sysctl` を変更しておく。

```
sudo bash -c 'echo "fs.inotify.max_user_watches=524288" >> /etc/sysctl.conf'
sudo sysctl -p
```

#### 参考

- [Running Visual Studio Code on Linux](https://code.visualstudio.com/docs/setup/linux#_visual-studio-code-is-unable-to-watch-for-file-changes-in-this-large-workspace-error-enospc)

### SSHの設定

もし、あなたがSSHの設定に詳しいのであれば、以下の手順に従う必要はなく、例えば手元の秘密鍵を転送するなどを行っても構わない。

```bash
ssh-keygen
cat ~/.ssh/id_rsa.pub
vim ~/.ssh/config
```

```
Host git
  Hostname github.com
  User git
  IdentityFile ~/.ssh/id_rsa

Host isu1
  Hostname TARGET_IP
  User USERNAME
  IdentityFile ~/.ssh/id_rsa
```

ssh鍵を生成後は https://github.com/settings/keys に公開鍵を登録後、GitHubに接続できるか確認する。

```
ssh -T git
```

`~/.ssh/config` には、GitHubと競技用サーバの設定を行う。`TARGET_IP` には競技用サーバのIPを登録する。GitHubに公開鍵を登録後、競技用サーバをセットアップすると authorized_keys のインポートができ、競技用サーバからすぐにsshができるようになるので楽かもしれない。



GitHubに接続できたら、以下のような表示がなされる。

```
Hi KasuyaMofu! You've successfully authenticated, but GitHub does not provide shell access.
```

本番同様、競技用サーバからアプリケーションをGitHubリポジトリに登録するなどを行う練習をするのであれば、 `isu1` にも秘密鍵とconfigを転送しておくと良い。

```bash
scp ~/.ssh/id_rsa ~/.ssh/config isu1:~/.ssh/
```

### git周りの設定

gitの `user.name` と `user.email` 設定を行う。 もし、gitにパブリックな設定を行いたくない場合、 [https://github.com/settings/emails] から `users.noreply.github.com` のメールアドレスを確認できる。例えば、`KasuyaMofu` の場合であれば以下のようになるので、各自に合わせたものを実行すること。

```bash
git config --global user.name KasuyaMofu
git config --global user.email 56345674+KasuyaMofu@users.noreply.github.com
```

### 最新版Golangのインストール

リポジトリを管理する `ghq` と [ISUCONの初期データ作成で使用](https://github.com/isucon/isucon10-qualify/blob/master/initial-data/README.md)する `wayt` をインストールするには、Ubuntu 18.04 LTSで入るgolangだと古いため、最新のものを入れる。

2021/08/09 時点での最新版は `1.16.7` なので、以下のように実行する。バージョンに合わせて、 [https://golang.org/dl/] を確認し、 `GOBINARY` を変更しても構わないが、動作の保証はできない。

```bash
cd ~
GOBINARY=go1.16.7.linux-amd64.tar.gz
wget https://golang.org/dl/$GOBINARY
sudo tar -C /usr/local -xzf $GOBINARY && rm $GOBINARY
```

`.bash_profile` などに以下を追加

```bash
vim ~/.bash_profile
```

```bash
export GOPATH=~/go
export GOBIN=$GOPATH/bin
export PATH=$PATH:$GOBIN
export PATH=$PATH:/usr/local/go/bin
```

反映のため、 `source` を実行する。

```bash
source ~/.bash_profile
```

`ghq` をインストールする。`git clone` ができる人はそれでも構わないが、後に `~/ghq/github.com/isucon/isucon10-qualify` にリポジトリがあるものとして手順を記載するので留意すること。

```bash
go get github.com/x-motemen/ghq
```

#### 参考

- [Ubuntuに最新のGolangをインストールする](https://qiita.com/notchi/items/5f76b2f77cff39eca4d8)

### ansibleのインストール

競技用サーバの構築には `ansible-playbook` を用いるため、ansibleをインストールする。

```bash
sudo apt update
sudo apt install software-properties-common
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt install ansible -y
```

#### 参考

- [Ansible のインストール — Ansible Documentation ](https://docs.ansible.com/ansible/2.9_ja/installation_guide/intro_installation.html#ubuntu-ansible)

### dockerのインストール

ベンチマーカーのデータ生成のため、dockerとdocker-composeが必要となるのでインストールする。

dockerのインストール手順はこちら。

```bash
sudo apt-get update
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release -y
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io -y
```

docker-composeもインストールする。

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

ユーザをdockerグループに入れる。`Makefile` の関係で、[initialdataの作成にsudo無しのdocker-composeを実行する](https://github.com/isucon/isucon10-qualify/blob/master/initial-data/Makefile#L21)ので、入れとかないとコケる。セキュリティ上の懸念がある場合は `Makefile` を修正すること。

```bash
sudo usermod -aG docker $USER
```

docker周りを綺麗にするために再起動する。

```bash
sudo shutdown -r now
```

#### 参考

- [Install Docker Engine on Ubuntu | Docker Documentation ](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)
- [Install Docker Compose | Docker Documentation ](https://docs.docker.com/compose/install/#install-compose-on-linux-systems)


### リポジトリの取得、データ生成、ベンチマーカーのビルド

make に必要なパッケージをインストールする。

```
sudo apt install make python3-pip -y
```

`ghq` で [https://github.com/isucon/isucon10-qualify.git] を取得する。リポジトリのクローンのみなので、 `git clone` に読み替えて貰っても構わないが、 `~/ghq` 配下にcloneする前提で手順を作成しているので留意すること。

初期データの `make` の際は、原因は分からないが1回目は `wayt` で止まる場合があるので、一度中断してもう一度実行するとうまく行く。

```bash
ghq get https://github.com/isucon/isucon10-qualify.git
cd ~/ghq/github.com/isucon/isucon10-qualify/initial-data/
pip3 install -r requirements.txt
go get github.com/orisano/wayt
make
^C
make
```

ベンチマーカーをmakeする。

```bash
cd ~/ghq/github.com/isucon/isucon10-qualify/bench/
make
```

#### 参考

- [isucon10-qualify/initial-data at master · isucon/isucon10-qualify](https://github.com/isucon/isucon10-qualify/tree/master/initial-data)
- [isucon10-qualify/bench at master · isucon/isucon10-qualify](https://github.com/isucon/isucon10-qualify/tree/master/bench )



### 競技用サーバの構築

`provisioning/ansible` で作業を行う。

```
cd ~/ghq/github.com/isucon/isucon10-qualify/provisioning/ansible
```

`isu1` に ansibleを実行するため、 `inventory/hosts` を書き換える。

```bash

cat << EOS > inventory/hosts
[competitor]
isu1
EOS
cat inventory/hosts
```

`ansible.cfg` を編集する。`pipelining=True` か `allow-world-readable-tmpfiles=true` を有効にしなければ、ansibleを実行したときに怒られる。

```bash
cat << EOS > ansible.cfg
[defaults]
private_key_file=~/.ssh/id_rsa
callback_whitelist = profile_tasks
pinelining=True
allow-world-readable-tmpfiles=true

[ssh_connection]
#pipeline = false
ssh_args = -o ControlMaster=auto -o ControlPersist=300s 
EOS
cat ansible.cfg
```

`deno` と `rust` はコンパイルに失敗するので除外する。なにが失敗したかは後述する。

```bash
sed -i -e "s/-.*\(deno\|rust\)/#&/g" competitor.yaml
cat competitor.yaml
```

いよいと `ansible-playbook` を実行する。各種ビルドが走るのでかなり時間が掛かるので気長に待つこと。 `-K` は、sudo を実行するためのパスワードを入力するためにつける。

```bash
ansible-playbook competitor.yaml -i inventory/hosts -K
```

ansibleの実行後、アプリケーションで使用するデータを転送する。ベンチマーカーで生成した `isucon10-qualify/initial-data/result/*.sql` を `isucon/isuumo/webapp/mysql/db/` に転送する。

```bash
scp ~/ghq/github.com/isucon/isucon10-qualify/initial-data/result/*.sql isu1:~/ && ssh -t isu1 "sudo mv -fv ~/*.sql /home/isucon/isuumo/webapp/mysql/db; sudo chown isucon: /home/isucon/isuumo/webapp/mysql/db/*.sql"
```

#### 参考

-  [isucon10-qualify/provisioning/ansible at master · isucon/isucon10-qualify](https://github.com/isucon/isucon10-qualify/tree/master/provisioning/ansible)


### ベンチマークの実行

ベンチを掛ける

```
cd ~/ghq/github.com/isucon/isucon10-qualify/bench/
./bench -target-url http://TARGET_IP:1323
```

以下のように表示が出る。負荷などの理由によってコネクションが切れてしまったときなどはエラーができるが正常動作である。最終的に、`score` にスコアが表示される。

```
2021/08/09 06:06:14 bench.go:78: === initialize ===
2021/08/09 06:06:16 bench.go:90: === verify ===
2021/08/09 06:06:16 bench.go:100: === validation ===
2021/08/09 06:06:23 load.go:181: 負荷レベルが上昇しました。
2021/08/09 06:06:30 load.go:181: 負荷レベルが上昇しました。
...
2021/08/09 06:07:16 bench.go:102: 最終的な負荷レベル: 11
{"pass":true,"score":2099,"messages":[{"text":"POST /api/estate/nazotte: リクエストに失敗しました (タイムアウトしました)","count":81}],"reason":"OK","language":"go"}
```

## おわりに

以上で手順は終了です。お疲れさまでした。ISUCON10に限らず、他の競技であっても、基本的な構築手順は変わらないと思いますので、是非試して共有してください。

競技では、この状態で構築されたサーバ `isu1`, `isu2`, `isu3` などにログインして作業を行うと思っていただいて相違ないかと思います。

例えば、`isuumo/webapp` を GitHubで管理するとか、 `alp` をインストールするとか、様々な作業があると思いますが、何をするべきかというのは、ISUCON運営の @rosylilly さんから詳細な事前講習がなされていますので、是非参考にしてください。


[ISUCON 事前講習2021 座学 を開催しました（資料と動画と問題あり）](https://isucon.net/archives/55835733.html)


実は私も、 Virtual ISUCON Live 前にこの講習をいただき、非常に勉強になりました。是非こちらもご覧ください。


[VTuberチームでISUCON 10本戦を解く企画「Virtual #ISUCON Live 2021」に出演しました！](https://kasuyamofu.hatenablog.com/entry/2021/07/24/virtual-ISUCON-Live-2021)


公式の講評はこちらです。

[https://kasuyamofu.hatenablog.com/entry/2021/07/24/virtual-ISUCON-Live-2021](https://isucon.net/archives/55943682.html)



### rustとdenoについて

文末にはなりますが、途中で記載しているrustとdenoについて、コメントしておきます。

原因としては、恐らく作った当時から状況が変わっているからだと思います。私はrustやdenoに詳しくなく、ansibleが通ったからと言って状況が正しいかどうかの判断ができませんので、issueを立てて誰かやってくれると嬉しいです。

調べた結果は[Twitterにメモしていました](https://twitter.com/KasuyaMofu/status/1423944452029091842)。大まかに以下のような状態でした。

- rust: `rustup default 1.46.0` に変更するとansibleは通る。動作ができているかはわからない。
  - https://github.com/isucon/isucon10-qualify/blob/master/provisioning/ansible/roles/langs/tasks/main.yaml#L19
- deno: よくわからない。https://raw.githubusercontent.com/denjucks/organ/master/mod.ts が消えているのは確か。
  - https://deno.land/x/organ/mod.ts に置き換えても駄目だった


以上、糟屋もふでした。