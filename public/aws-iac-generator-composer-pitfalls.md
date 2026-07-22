---
title: CloudFormation IaCジェネレーターでAWSアカウントを丸ごとApplication Composerに読み込もうとしてハマった話
tags:
  - AWS
  - CloudFormation
  - IaC
  - ApplicationComposer
  - AWSCLI
private: false
organization_url_name: null
slide: false
ignorePublish: false
---
## はじめに

「今使っているAWSリソースを、CloudFormation IaCジェネレーターでスキャンしてApplication Composerの画面で俯瞰したい」

そんな軽い気持ちで始めた検証が、想像以上にハマりどころの多い作業になったので記録として残します。結論から言うと、**アカウント全体を1枚の構成図として俯瞰する用途にはこの組み合わせは向いていません**。ただし、その過程で得られた知見はいくつもあったので共有します。

## やろうとしたこと

1. AWS CLIでアカウント内のリソースをスキャン（IaCジェネレーター）
2. スキャン結果からCloudFormationテンプレートを生成
3. Application Composerの画面で構成を可視化する

やること自体はシンプルに見えますが、実際にはAWS CLIの未インストールから始まり、いくつもの壁にぶつかりました。

## つまずきポイント1: リソース数が想像以上に多い

`aws cloudformation start-resource-scan` でアカウントをスキャンすると、**1819件**のリソースが検出されました。個人利用でOrganizations未使用の小規模アカウントでもこの数です。

内訳を見ると、実際にはAWSが自動生成する既定リソースがかなりの割合を占めていました。

| リソースタイプ | 件数 | 正体 |
|---|---|---|
| `AWS::SSM::Document` | 820 | AWS製の公開ドキュメント（`AWSEC2-*`など） |
| `AWS::Logs::LogStream` | 212 | ロググループ配下の個別ストリーム |
| `AWS::RAM::Permission` | 119 | `arn:aws:ram::aws:permission/*` というAWS所有のARN |
| `AWS::CodeDeploy::DeploymentConfig` | 17 | `CodeDeployDefault.*` というAWS組み込みの設定 |
| `AWS::ElastiCache::ParameterGroup` 等 | 数十 | `default.*` という各エンジンの既定パラメータグループ |

これらはユーザーが作成したものではなく、費用も発生しないAWSの「備え付け」リソースです。こうしたノイズを resource type / identifier のパターンで除外していき、最終的に **451件** まで絞り込みました。

```powershell
# 例: IAMロールのうちAWSサービスリンクロールを除外
$filtered = $data.Resources | Where-Object {
    -not ($_.ResourceType -eq "AWS::IAM::Role" -and $_.ResourceIdentifier.RoleName -like "AWSServiceRoleFor*")
}
```

## つまずきポイント2: 「Application Composer」はもう存在しない

生成したテンプレートを画面で見ようとして、いくら探しても「Application Composerで表示」のようなボタンが見当たりませんでした。

種明かしをすると、**Application Composerは Infrastructure Composer に名称変更されていました**。手順書やブログ記事が旧名前提だと、最新のコンソールでは詰みます。

## つまずきポイント3: 451件では図として成立しない

CloudFormationコンソールの「IaC ジェネレーター」画面には1テンプレートあたり最大500リソースという制限があります。451件はギリギリ収まる数でしたが、実際にInfrastructure Composerで開いてみると、こんな警告が出ました。

> Infrastructure Composerでは現在、大量のリソースセットの視覚化についての限定的なサポートが提供されています。

そして実際の見た目は、期待していた「ノードが線で繋がった構成図」ではなく、**カードが縦一列にひたすら積み重なるだけ**のものでした。ズームやレイアウト自動調整（「調整」ボタン）を試しても改善しません。

「じゃあ実際に課金・稼働しているリソースだけに絞れば？」と思い、Lambda・S3・DynamoDB・CloudFront・API Gatewayなど実運用に関わるリソースタイプだけに絞り込み（107件）ましたが、結果は変わりませんでした。

## つまずきポイント4: 本当の原因はテンプレートの「依存関係の有無」だった

ここでようやく本質に気づきました。IaCジェネレーターがアカウントスキャンから生成するテンプレートは、各リソースを **フラットに列挙するだけ** で、`Ref` や `Fn::GetAtt` による依存関係の記述がほとんどありません。Composerは依存関係を頼りにレイアウトするため、繋ぐ材料がなければ縦一列に並べるしかないのです。

そこで方針を変えて、**既存のCloudFormationスタック**（人間やSAM CLIが最初から書いた、依存関係を含む本来のテンプレート）を読み込ませてみました。

```powershell
aws cloudformation get-template --stack-name cron-to-sns --query TemplateBody --output json
```

結果、Lambda関数・API Gateway・IAMロールなどが関連リソースとしてカードにまとまって表示されるようになりました。ただし期待していた「矢印線で繋がるノードグラフ」ではなく、**親リソースのカードの中に、関連する子リソース（IAMロール、Permission、Deploymentなど）がリスト形式でネストされる**という表示方式でした。ノードグラフ的な図が欲しい場合は、これはこれで期待と少しズレます。

## つまずきポイント5: `--output yaml` の罠

CLIの出力をそのままComposerに貼り付けたところ、何も反映されず沈黙されるという不可解な現象に遭遇しました。

原因は `aws cloudformation get-template --output yaml` の出力形式にありました。

```yaml
!!omap
- AWSTemplateFormatVersion: '2010-09-09'
- Outputs: !!omap
  - CronToSNSApi: !!omap
```

`!!omap` はPythonの順序付き辞書をYAMLタグとして表現したもので、CloudFormationの標準テンプレート構文としては不正です。Composer側は不正なYAMLを検知しても特にエラーを表示せず、静かに変更を無視していました。

**対処法はシンプルで、`--output json` を使うことです。** ComposerのエディタはYAML/JSON切り替えトグルを持っているので、JSON側に切り替えて貼り付ければ問題なく読み込めます。

```powershell
aws cloudformation get-template --stack-name <stack> --query TemplateBody --output json
```

## つまずきポイント6: 大きいテンプレートをどうやってComposerに流し込むか

Composerのメニューには「テンプレートファイルを開く」という項目がありますが、これはOSのネイティブファイル選択ダイアログを呼び出すため、ブラウザ自動操作では制御できません（見えないダイアログを操作できないため）。

かといって数十万文字のテンプレートを1文字ずつタイプさせるのは非現実的です。最終的に落ち着いたのは、OSのクリップボード経由で流し込む方法でした。

```powershell
Get-Content -Raw <file> | Set-Clipboard
```

その後、Composerのテンプレートエディタをクリックしてフォーカスし、`Ctrl+A` → `Ctrl+V` で流し込みます。数十万文字規模でも問題なく貼り付けられました。

## まとめ

| やりたかったこと | 結果 |
|---|---|
| アカウント全体を1枚の俯瞰図にする | ✗ Composerは大量リソースの視覚化に非対応。縦一列になるだけ |
| 実際に課金・稼働しているものだけに絞る | ✗ 絞り込んでも依存関係がなければ変わらない |
| 既存スタック単位で構成を確認する | ○ 関連リソースがカードにまとまって表示される（ネスト形式） |
| 矢印で繋がるノードグラフが欲しい | △ Composerの標準機能では出せない。自前でRef/GetAttを解析してMermaid等で描く方が確実 |

IaCジェネレーター + Application (Infrastructure) Composer は、**「1つの既存アプリ/スタックの構成をざっと確認する」用途** には便利ですが、**「アカウント全体を俯瞰する」用途** には現状向いていません。同じことをやろうとしている人の参考になれば幸いです。
