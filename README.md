# WambdaInitProject CSR001 Frontend

Vue.js 3 + Vite で構築されたシングルページアプリケーション（SPA）です。CloudFront + S3 でホスティングし、WAMBDAバックエンドと統合されています。

## 概要

このプロジェクトは、CSR001のフロントエンドアプリケーションです。
Backend（WambdaInitProject_CSR001_Backend）と分離され、独立したリポジトリで管理されます。

### 技術スタック

- **フレームワーク**: Vue.js 3
- **ビルドツール**: Vite
- **ルーティング**: Vue Router (History Mode)
- **ホスティング**: S3 + CloudFront (CDKで構築)
- **認証**: WAMBDA バックエンド (`/accounts/*`) と連携
- **CI/CD**: CodeBuild (自動デプロイ)

## プロジェクト構造

```
WambdaInitProject_CSR001_Frontend/
├── src/
│   ├── App.vue           # メインアプリケーション
│   ├── main.js           # エントリーポイント + Vue Router
│   └── views/            # ページコンポーネント
│       ├── Home.vue      # ホームページ
│       └── Protected.vue # 認証が必要なページ
├── index.html
├── package.json
├── package-lock.json     # CodeBuild必須
├── vite.config.js        # Vite設定
├── buildspec.yml         # CodeBuild設定
└── README.md
```

## 開発環境

### 前提条件

- Node.js 20以上
- npm

### セットアップ

```bash
# 依存関係をインストール
npm install

# 開発サーバー起動（localhost:5173）
npm run dev

# プロダクション用ビルド
npm run build

# ビルド結果をプレビュー
npm run preview
```

## デプロイ

### 自動デプロイ（推奨）

GitHubのmainブランチにpushすると、CodeBuildが自動的にビルド＆デプロイします。

```bash
git add .
git commit -m "Update frontend"
git push origin main
```

CodeBuildは以下を自動実行します：
1. `npm ci` - 依存関係インストール
2. `npm run build` - プロダクションビルド
3. `aws s3 sync dist/ s3://hakira0627-s3-wambda-csr001-main/CloudFront/` - S3アップロード
4. CloudFrontキャッシュ無効化

### 手動デプロイ

```bash
# 1. ビルド
npm run build

# 2. S3にアップロード
aws s3 sync ./dist/ s3://hakira0627-s3-wambda-csr001-main/CloudFront/ --delete --profile wambda

# 3. CloudFrontキャッシュ無効化
DISTRIBUTION_ID=$(aws cloudformation describe-stacks \
  --stack-name stack-wambda-csr001-infra-main \
  --query "Stacks[0].Outputs[?OutputKey=='DistributionId'].OutputValue" \
  --output text --profile wambda)
aws cloudfront create-invalidation --distribution-id $DISTRIBUTION_ID --paths "/*" --profile wambda
```

## アーキテクチャ

### ルーティング構成（CloudFront）

- **`/`**: Vue.js SPA（S3オリジン）
- **`/accounts/*`**: WAMBDA認証画面（API Gatewayオリジン）
- **`/api/*`**: API エンドポイント（API Gatewayオリジン）

### 認証フロー

1. 認証が必要なページアクセス時に `/accounts/status` で認証状態確認
2. 未認証の場合は `/accounts/login?next=/protected` にリダイレクト
3. 認証後は `/?next=/protected` にリダイレクト
4. Vue.jsが`next`パラメータを読み取り、元のページに遷移

### Vue Router構成

```javascript
// History mode で SPA ルーティング
const router = createRouter({
  history: createWebHistory('/'),
  routes: [
    { path: '/', name: 'Home', component: Home },
    { path: '/protected', name: 'Protected', component: Protected, meta: { requiresAuth: true } }
  ]
})

// 認証ガード
router.beforeEach(async (to, from, next) => {
  if (to.meta.requiresAuth) {
    try {
      const response = await fetch('/accounts/status')
      const data = await response.json()
      if (data.authenticated) {
        next()
      } else {
        window.location.href = '/accounts/login?next=' + encodeURIComponent(to.fullPath)
      }
    } catch (error) {
      window.location.href = '/accounts/login?next=' + encodeURIComponent(to.fullPath)
    }
  } else {
    next()
  }
})
```

## CI/CD

CodeBuildによる自動デプロイが設定されています。

- **Webhook**: mainブランチへのpushで自動起動
- **ビルド環境**: Node.js 20
- **S3バケット**: `hakira0627-s3-wambda-csr001-main/CloudFront/`
- **CloudFront**: 自動キャッシュ無効化

詳細は `WambdaInitProject_CICD/csr001-frontend/` を参照。

## CloudFront設定

CloudFrontはCDKで自動構築されます（`WambdaInitProject_Infra`）。

### 主要設定

- **Default Behavior**: S3 オリジン（Vue.js アプリ）
- **Behavior `/accounts/*`**: API Gateway オリジン（認証SSR）
- **Behavior `/api/*`**: API Gateway オリジン（JSON API）
- **Error Response**: 404 → 200 `/index.html`（SPA routing対応）
- **Origin Path**: S3は `/CloudFront`、API Gatewayは `/stage-01`

## 連携

- **Backend**: WambdaInitProject_CSR001_Backend（WAMBDA + Lambda + API Gateway）
- **Infrastructure**: WambdaInitProject_Infra（CDK - Cognito, CloudFront, S3）

## トラブルシューティング

### よくある問題

1. **ルーティングエラー**: CloudFrontのError Response（404→200 /index.html）が設定されているか確認
2. **認証ループ**: `/accounts/status` が正しくレスポンスを返しているか確認
3. **CORS エラー**: 開発時はバックエンドとポートが異なる場合に発生、本番環境では同一ドメイン経由なので問題なし
4. **ビルドエラー**: `package-lock.json`がコミットされているか確認（CodeBuildで`npm ci`を使用）

### デバッグ

```bash
# 開発サーバーでバックエンド確認
# ブラウザの開発者ツールでネットワークタブを確認
# Vue.js Devtools でルーティング状態を確認
```

## 開発のポイント

- **History Mode**: Vue RouterはHistory Modeを使用、CloudFrontの404エラーハンドリングが必要
- **認証統合**: `/accounts/status` APIでバックエンドの認証状態を確認
- **CORS不要**: CloudFront経由で同一ドメイン、CORSの問題なし
- **package-lock.json**: 必ずコミット、CodeBuildで`npm ci`を使用

## ライセンス

MIT License

## 作者

h-akira
