# azure-local-virtual-deployment-skill

[English README is here / 英語版 README](./README.md)

Azure VM 上のネステッド仮想化環境に、シングルノードの **Azure Local**(旧 Azure Stack HCI)クラスターをラボ目的でデプロイする際の、エンドツーエンドの手順をガイドします。

Microsoft Learn の [Deploy Azure Local using virtual machines](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deployment-virtual) を実際に試して詰まったポイント(コマンドの誤記、DNS/NAT の落とし穴、パスワードポリシーの差異など)に対する**修正済みコマンド**込みのウォークスルーです。

> ⚠️ このスキルは**ラボ・学習・PoC 用**です。物理ハードウェアでの本番デプロイには使用しないでください。

## このスキルでできること

AI から呼び出すと、Azure Local のラボ環境構築を 6 つのフェーズに分けて、対話的に手順をガイドします。

| フェーズ | 内容 | 目安時間 |
|---|---|---|
| 1 | ホストの仮想スイッチ + NAT 設定 | 15 分 |
| 2 | Node VM の作成、Azure Stack HCI OS インストール、NIC 設定 | 60-90 分 |
| 3 | Azure 側の準備(リソースプロバイダー、RG、ロール割り当て) | 15 分 |
| 4 | Node を Azure Arc に登録 | 15 分 |
| 5 | Domain Controller VM + AD + LCM ユーザー作成 | 60 分 |
| 6 | Azure ポータルからクラスターデプロイ | 入力 30 分 + 自動デプロイ 2 時間 |

合計 4〜5 時間(うち多くは自動処理の待ち時間)。

## 前提条件

- ネステッド仮想化を有効化した Azure VM(`Standard_E32-8s_v5` 推奨 / 8 vCPU・256 GiB RAM。コア数を絞ったコンストレインド SKU なので、ネスト上の DC VM と Node VM に十分なメモリを確保しつつコア課金ライセンスのコストを抑えられます)
- Azure サブスクリプションで **Owner** もしくは **Contributor + User Access Administrator**
- Windows Server 2022/2025 の ISO(Domain Controller VM 用)
- Azure Stack HCI 24H2 の ISO
- 連続して 3〜5 時間作業できる時間枠
- Azure コストが発生することの理解(ホスト VM の課金 + 60 日トライアル後の Azure Local サービス料金)

## インストール

### 方法 A — Claude App (claude.ai) のカスタムスキルとして追加

`azure-local-virtual-deployment.skill`(`SKILL.md` + `references/` を固めた ZIP)をカスタムスキルとしてアップロードします。

1. https://claude.ai/settings/capabilities を開く
2. **Skills** セクションの **Upload skill**(カスタムスキル)を選択
3. 本リポジトリの `azure-local-virtual-deployment.skill` を選択してアップロード
4. `SKILL.md` の frontmatter から自動的にスキルが登録されます(再起動不要)

### 方法 B — CLI ツール(Claude Code / Copilot CLI / Codex)

`~/[.copilot|.claude|.codex]/skills/` 配下にコピーします。

```bash
# このリポジトリをクローン
git clone https://github.com/koudaiii/azure-local-virtual-deployment-skill.git
cd azure-local-virtual-deployment-skill

# ln -s "$PWD" ~/[.copilot|.claude|.codex]/skills/azure-local-virtual-deployment
```

AI を再起動するとスキルが認識されます。

## 使い方

AI との会話中に、以下のようなプロンプトを送るとスキルが自動的に起動します。

- 「Azure Local のラボを作りたい」
- 「Azure VM 上で Azure Stack HCI を試したい」
- 「Microsoft Learn の `deployment-virtual` の手順で詰まった」
- 「ネステッド仮想化で Azure Local をデプロイする手順を教えて」

スキルは現在どのフェーズにいるかを確認したうえで、該当する参照ファイル(`references/0X-*.md`)を読み、コマンドブロック単位で手順を提示します。出力を共有しながら一歩ずつ進める想定です。

エラーに遭遇した場合は `references/troubleshooting.md` を参照し、既知の落とし穴と修正手順を案内します。

## ファイル構成

```
.
├── README.md                                  ← このファイル
├── SKILL.md                                   ← スキル本体(全体方針・原則)
└── azure-local-virtual-deployment/
    └── references/
        ├── 01-networking-host.md             ← Phase 1: ホストネットワーク
        ├── 02-node-vm.md                     ← Phase 2: Node VM
        ├── 03-azure-prep.md                  ← Phase 3: Azure 準備
        ├── 04-arc-registration.md            ← Phase 4: Arc 登録
        ├── 05-ad-setup.md                    ← Phase 5: AD セットアップ
        ├── 06-cluster-portal.md              ← Phase 6: クラスター デプロイ
        └── troubleshooting.md                ← 全フェーズ共通のトラブルシューティング
```

## このスキルが効く「落とし穴」の例

公式ドキュメントどおりに進めるとハマりやすいポイントを、修正コマンド込みで網羅しています。

- `New-VMSwitch -SwitchType External` は実際には動かない(`-SwitchType` は `Internal/Private` のみ受け付ける)
- LCM ユーザーのパスワードは AD のデフォルトポリシー(7+ 文字)を通っても、ポータルウィザード側で **14 文字以上 4 種混在**を要求される
- Node を**クラスターデプロイ前**にドメイン参加させると失敗する(自動参加に任せる)
- DC VM の vSwitch を後から切り替えると、PowerShell Direct の認証キャッシュが壊れる(VM 再起動で復旧)
- `New-NetNat` はホストごとに 1 つしか作れない

詳細は `SKILL.md` の "Critical principles" と `references/troubleshooting.md` に記載しています。

## クリーンアップ

ラボ終了時の課金停止手順は `references/troubleshooting.md` の "Cleanup procedure" に記載しています。Azure Local のサービス料金を止めるには **クラスター リソースの削除**が必須で、Node VM を停止するだけでは課金は止まりません。

## ライセンス

MIT License

## 免責

このスキルは Microsoft 社の公式コンテンツではなく、Microsoft Learn の手順を個人的に検証して得た知見をまとめた非公式ドキュメントです。Azure Local の仕様変更やドキュメント更新によって、内容が古くなる可能性があります。本番環境での利用は想定していません。
