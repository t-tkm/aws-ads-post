# 1. はじめに
この投稿では、AWSの既存システムデータ収集サービスである[Application Discovery Service
(ADS)](https://aws.amazon.com/jp/application-discovery/)の検証内容を紹介します。
ADSは、AWS移行の計画フェーズにおいて、既存システムの情報が整理・更新されていない(可視化が
不十分)場合に、効率的な既存システム情報の収集を手助けしてくれるサービスになります。
**特に「既存リソースがオーバースペック気味で、実際リソースに余裕があるのでは？このAWS移行を
機に適切なサイジングを見直したい」** という場合、是非本ツールで最新状況を確認されると良いかと
思います。

コンテンツとしては、下記が含まれています:
- ADSによる情報収集(3章)
- AthenaやQuickSightによる簡単な分析手法(4章)
- 生データもダウンロードできるようにしています。「時間がなくてツールを試している時間は無いが、
  ADSでどのような情報が取得できるか見てみたい」という方は、是非ご活用ください。

検証には、下記AWS Workshopのコンテンツに従い、手元にある環境(移行元)でADSを試しました。本記事で言葉足らずな箇所につきましては、是非ご参考下さい。

- [AWS Workshops -AWS APPLICATION DISCOVERY SERVICE](https://migration-immersionday.workshop.aws/en/rehost/ads.html)

# 2. システム構成
検証のシステム構成は次のようになります。

<div align="center">
  <a href="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img1.png?raw=true">
    <img alt="img1" width="600" src="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img1.png?raw=true">
  </a>
</div>

AWS Workshopに沿って、LinuxのWordpressシステムを使います。手元のPC(iMaC)のVirtualBoxに、
VMを2個作成し、wordpressのシステムをプロビジョニングしています。

また、折角の機会ですので、上記のwordpressシステムに加え、AWS、Azure、GCPにプロビジョニングされたVMも追加しております。

# 3. ADSを用いた情報収集
この章では、ADSによる情報収集のための設定と、収集されたデータの確認、及び、適切なEC2サーバの
推奨を取得する機能を見ていきます。

また、次の4章でAthenaとQuickSightを用いた分析を行いますが、Athenaでデータ分析を行うには、
S3へ収集データをエクスポートしておく必要があります。これは「Amazon Athenaでのデータ探索」
というオプションを有効化しておくだけで、後はユーザの見えないところでAWSサービスが自動で必要な
設定・リソースを作成してくれるのですが、ここでは背後でどのようなリソースが実際作成されている
かも確認しています。

それでは、早速試していきたいと思います。

## 3-1. Migration Hubの有効化
ADSを使うために、Migration Hubの有効化を行います。

突然見慣れない「Migration Hub」というサービスからスタートします。ADSはMigration Hubの
一部として組み込まれているため、ADSを使うにはMigration Hubの設定が必須なのですが、
Migration Hubは、ADSだけでなく、その後に続く移行ツール(例えばCloud Endure)などを統合的に
追跡するためのダッシュボードサービスになっています。

<div align="center">
  <a href="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img2.png?raw=true">
    <img alt="img2" width="600" src="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img2.png?raw=true">
  </a>
</div>

## 3-2. Athena統合の有効化
4章で分析をするために必要な設定になります。従って、収集した情報をADS(Migration Hub)だけで
確認するのであれば、この機能は不要になります。

この機能を有効にすると、後述するADSのエージェントから収集されたデータ(ストリーム)は
Kinesis Data Firehose経由でS3へ格納されるようになります。そして、Athenaからクエリ(検索)
できるように、Glueを使ったデータカタログが自動で生成されます。

<div align="center">
  <a href="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img3.png?raw=true">
    <img alt="img3" width="600" src="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img3.png?raw=true">
  </a>
</div>

- 自動設定されたKinesisとS3バケット
- <div align="center">
  <a href="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img4.png?raw=true">
    <img alt="img4" width="600" src="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img4.png?raw=true">
  </a>
</div>

- 自動設定されたGlueデータカタログ
- <div align="center">
  <a href="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img5.png?raw=true">
    <img alt="img5" width="600" src="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img5.png?raw=true">
  </a>
</div>

## 3-3. ADSエージェントのインストール&セットアップ
情報収集したい対象のサーバに、個別にエージェントを設定します。設定は、ナビゲーションの「ツール」
から「検出エージェント」を選択すると、Windows・Linux用エージェントのダウンロード&インストール
の方法が表示されますので、指示に従い、個別に仮想マシンに導入します。

<div align="center">
  <a href="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img6.png?raw=true">
    <img alt="img6" width="600" src="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img6.png?raw=true">
  </a>
</div>

- Linux用

    ```bash
    curl -o ./aws-discovery-agent.tar.gz https://s3-us-west-2.amazonaws.com/aws-discovery-agent.us-west-2/linux/latest/aws-discovery-agent.tar.gz
    tar -xzf aws-discovery-agent.tar.gz
    sudo bash install -r "ap-northeast-1" -k "<AWS key id>" -s "<AWS key secret>"
    ```

- Windows用

    ```powershell
    Import-Module BitsTransfer
    Start-BitsTransfer -Source "https://s3-us-west-2.amazonaws.com/aws-discovery-agent.us-west-2/windows/latest/AWSDiscoveryAgentInstaller.msi" -Destination "C:\Users\Administrator\Downloads\"
    msiexec.exe /i c:\Users\Administrator\Downloads\AWSDiscoveryAgentInstaller.msi REGION="ap-northeast-1" KEY_ID="<aws key id>" KEY_SECRET="<aws key secret>" /q
    ```

本投稿では、各仮想マシンにエージェントを導入する方法を試しましたが、vCenter上の仮想マシンであれ
ば、「検出コネクター」(仮想アプライアンス-OVA/OVFパッケージ)を使うことで、エージェントレスでの
情報収集が可能です。

暫くすると「Data collection」のAgentsタブに、各エージェントが表示されます。エージェントを
選択し「データ収集を開始」ボタンを押します。

<div align="center">
  <a href="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img7.png?raw=true">
    <img alt="img7" width="600" src="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img7.png?raw=true">
  </a>
</div>

## 3-4. 収集された情報の確認
エージェントから収集された取得項目を確認してみましょう。

<div align="center">
  <a href="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img8.png?raw=true">
    <img alt="img8" width="600" src="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img8.png?raw=true">
  </a>
</div>

<div align="center">
  <a href="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img9.png?raw=true">
    <img alt="img9" width="600" src="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img9.png?raw=true">
  </a>
</div>

ネットワーク関連も表示できますが、必要のない接続先(IP)も多いです(怪しい接続先がないかの確認に
使えるかもしれません)。

<div align="center">
  <a href="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img10.png?raw=true">
    <img alt="img10" width="600" src="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img10.png?raw=true">
  </a>
</div>

## 3-5. EC2推奨取得
ADSが収集した情報をベースに、EC2インスタンスの推奨を取得できます。各サーバのCPU、メモリ使用率な
ど、インベントリ情報の詳細を分析しているようです。

<div align="center">
  <a href="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img11.png?raw=true">
    <img alt="img11" width="600" src="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img11.png?raw=true">
  </a>
</div>

取得データは、下記のようになります。
<div align="center">
  <a href="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img14.png?raw=true">
    <img alt="img14" width="600" src="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img14.png?raw=true">
  </a>
</div>

生データは、下記からダウンロード頂けます。
- [CSVファイルのダウンロード](EC2InstanceRecommendations-DirectMatch-2021-07-04-05-45.csv)
- [エクセルファイルのダウンロード](EC2InstanceRecommendations-DirectMatch-2021-07-04-05-45.xlsx)

# 4. AthenaとQuickSightを用いた分析
## 4-1. Athenaの有効化
はじめに、Athenaサービスを有効化します。

<div align="center">
  <a href="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img12.png?raw=true">
    <img alt="img12" width="600" src="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img12.png?raw=true">
  </a>
</div>

## 4-2. テーブル一覧
§3-2で自動設定されたAthena用のリソースでは、公式ドキュメント「[Amazon Athena でのデータ
探索の操作](https://docs.aws.amazon.com/ja_jp/application-discovery/latest/userguide/working-with-data-athena.html)」より、デフォルトでエージェントから取得できます:
<div align="center">
  <a href="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img13.png?raw=true">
    <img alt="img13" width="600" src="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img13.png?raw=true">
  </a>
</div>

下記は、各テーブル(7つのテーブル)に取得したデータになります:

<div align="center">
  <a href="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img15.png?raw=true">
    <img alt="img15" width="600" src="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img15.png?raw=true">
  </a>
</div>

<div align="center">
  <a href="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img16.png?raw=true">
    <img alt="img16" width="600" src="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img16.png?raw=true">
  </a>
</div>

<div align="center">
  <a href="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img17.png?raw=true">
    <img alt="img17" width="600" src="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img17.png?raw=true">
  </a>
</div>

生データは下記からダウンロード頂けます。ただし、生データ内のアカウントIDはお見せする訳には
いかないため、ダミーで「123456789012」と加工しています。その他の値は未加工です。
- [生データのダウンロード(zip)](Athena-table-csv/athena-table-csv.zip)


## 4-3. Athenaで分析
参考サイト[3]のハンズオンに従い、実験してみたクエリは下記になります:
>**Obtain IP Addresses and Hostnames for Servers**
>This view helper function retrieves IP addresses and hostnames for a 
>given server. You can use this >view in other queries.

<div align="center">
  <a href="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img18.png?raw=true">
    <img alt="img18" width="600" src="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img18.png?raw=true">
  </a>
</div>

<div align="center">
  <a href="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img19.png?raw=true">
    <img alt="img19" width="600" src="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img19.png?raw=true">
  </a>
</div>

また、このような実験も試してみましょう。
<div align="center">
  <a href="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img20.png?raw=true">
    <img alt="img20" width="600" src="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img20.png?raw=true">
  </a>
</div>

- Amazon Linux(on AWS)のHypervisorが「Xen」と表示されました。
- Azure VM(on Azure)のHypervisorは「Hyper-V」とは表示されないのですね。

他にも、下記サイトにあるクエリを参考にできます。
- [EXPLORE ADS DATA IN ATHENA](https://migration-immersionday.workshop.aws/en/rehost/ads/3.athena/explore_with_athena.html)
- 公式ドキュメント「[Athena で使用する事前定義されたクエリ](https://docs.aws.amazon.com/ja_jp/application-discovery/latest/userguide/working-with-data-athena.html#predefined-queries)」

## 4-4. QuickSightで分析
ここではQuickSightの詳細は省略します。下記をご参考ください。
- [Amazon QuickSight とは](https://docs.aws.amazon.com/ja_jp/quicksight/latest/user/welcome.html)
- [AWS Black Belt Online Seminar QuickSight](https://d1.awsstatic.com/webinars/jp/pdf/services/20201117_Blackbelt_Amazon_QuickSight_for_ISV_A.pdf)
- [AWS Black Belt Online Seminar Amazon QuickSight アップデート](https://d1.awsstatic.com/webinars/jp/pdf/services/20200204_AWS_BlackBelt_QuickSight_Update.pdf)
- [Amazon QuickSight の製品デモ動画](https://aws.amazon.com/jp/quicksight/resources/product-videos/)

初めに、Athenaをデータソースとして設定します。
<div align="center">
  <a href="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img22.png?raw=true">
    <img alt="img22" width="600" src="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img22.png?raw=true">
  </a>
</div>

<div align="center">
  <a href="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img21.png?raw=true">
    <img alt="img21" width="600" src="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img21.png?raw=true">
  </a>
</div>

こちらは、6台のサーバについて、cpu利用率を確認したものになります。収集されたデータに基づき、
いろいろな観点での分析が可能になります。

<div align="center">
  <a href="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img23.gif">
    <img alt="img23" width="600" src="https://github.com/t-tkm/aws-ads-post/blob/images/imgs/img23.gif">
  </a>
</div>

# 5. まとめ
本投稿では、様々なサーバにADSエージェントを設定し、ADSを用いた情報の収集、分析方法を紹介しまし
た。今回は4章で分析を詳細にするためにAthenaやQuickSightまで試してみましたが、実際の現場では
移行先サイジングの目安だけでも十分である場合が多いと思います(3章)。とても簡単に使えるツール
ですので、移行の際には是非一度お試し下さい。