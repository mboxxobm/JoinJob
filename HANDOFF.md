# JoinJob 実装ハンドオフ（LUNA向け）

作成: 2026-10-24（JST） / 対象: 2週間MVP / 原典: 企画会話メモ（末尾「参考」）

> この文書は、方針が何度も変わった企画会話を **1つの決定セット** にまとめたもの。
> 会話内で矛盾がある場合は **最新の発言を優先** した。決まっていないことは「8. 未決事項」に分けてあり、LUNAは勝手に決めないこと（決定が必要になったら実装を止めずに差し替えやすい形で作り、ユーザーに確認する）。

---

## 0. リポジトリの現状（2026-10-24 時点で確認）

- `https://github.com/mboxxobm/JoinJob`（public、説明「企業マッチング」、default branch `main`）は **空（コミット0件）**。2026-10-21 17:33（JST）作成以降、push されていない。
- 会話で「作成した」とされた `README.md`／`composer.json`／`.env.supabase.example`／デプロイ手順書／`architecture.webp`／各種ZIP（`JoinJob_push_package.zip`、`JoinJob-FINAL-PACKAGE.zip`、`JoinJob-TICKET-FINAL.zip`、`JoinJob-MASTER-ALL.zip`）は **リポジトリに存在しない**。中身も手元に無いので、**すべてゼロから作る前提**で進めること。
- リポジトリは **public**。`.env`・APIキー・DBダンプ・音声ファイルは絶対にコミットしない（`.gitignore` 必須）。

---

## 1. プロジェクト概要

- **JoinJob**: ocosba.com 型の起業・複業仲間マッチングの2026年版。**北海道（札幌・小樽）特化**で開始。
- **2階建てモデル**
  - **1F 仲間探しフロア（無料）**: 募集投稿・プロフィール・**任意の10秒音声自己紹介**・制限付きDM。
  - **2F 壁打ちチケットフロア（有料）**: TimeTicket 型。ユーザーが「30分 ◯円で壁打ちします」等のチケットを出品・販売。プラットフォームが手数料 **約20〜25%** を取る。**販売者への支払いは当面手動（銀行振込）**。
- 収益は **チケット手数料のみ**。ocosba型の月額サブスク（300円/500円）・成功報酬案は **不採用**（会話の途中案）。

---

## 2. 確定した技術構成

| 項目 | 決定 |
|---|---|
| LP（www） | **Astro**（`output: 'static'`）+ Tailwind。`dist/` を **DOMAINKING に FTP**。WordPressは使わない |
| バックエンド | **Laravel 11 / PHP 8.2**（JSON API） |
| DB | **Supabase（PostgreSQL）**。DOMAINKING の MySQL 5.5 は **使わない** |
| 決済 | **Stripe**（チケット購入は Checkout `mode: payment`／PaymentIntent）。カード情報は保存しない |
| 音声 | 10秒・任意。フロントは **MediaRecorder**、文字起こしは **Whisper** |
| AI | 文章添削は **DeepSeek API**（OpenAI互換、`base_url` 差し替え） |
| チャット | リアルタイム無し（ポーリング／再読込で可）。Pusher無し |
| 検索 | `ILIKE` の単純検索のみ |

### サブドメイン

| サブドメイン | 用途 | 置き場所 |
|---|---|---|
| `www.<domain>` | Astro静的LP・法務ページ（・将来ブログ） | DOMAINKING（FTP）【確定】 |
| `app.<domain>` | アプリ画面 | **未決**（8章） |
| `api.<domain>` | Laravel API | **未決**（DOMAINKING or Render.com、8章） |

`<domain>` 自体も未決。コード上は環境変数（`APP_URL`, `FRONTEND_URL`, `WWW_URL`）で受け、ハードコードしない。

### MVPでは入れないもの（リリース後）
Embedding相性マッチング / pgvector / Pusher・リアルタイムチャット / レビュー / Cloudflare R2 / Filamentの凝った権限 / Meilisearch / GitHub・X連携スキル証明 / チーム・フォロー・投資家機能 / サブスク / 成功報酬 / SEO用URL・JobPosting構造化データ（Phase 2施策）

---

## 3. MVP範囲（2週間）

**やること**
1. Astro LP 1ページ（「声でわかる起業マッチング JoinJob」、CTA「無料で始める」→ app）
2. 新規登録・ログイン・ログアウト（方式は未決: Supabase Auth or Laravel認証）
3. プロフィール（表示名・自己紹介・エリア・スキル等）＋ **任意の10秒音声**（会話の提案: 「10秒自己紹介」「10秒 一緒にやりたいこと」の2種）
4. 募集投稿 CRUD と一覧（`ILIKE` 検索のみ）
5. DM（**1日3通まで**、リアルタイム無し）
6. チケット出品・購入（**レビュー無し、振込は手動**）
7. 通報ボタン、NGワードフィルタ、簡易管理画面（通報処理・購入/振込状況・ユーザー停止）
8. **退会**（音声ファイルも物理削除）
9. **バックアップ**（`pg_dump` 定期実行の仕組み）
10. **エラー監視・ログ**（会話ではSentryが例示）
11. 法務ページは **ルートとプレースホルダのみ**（本文はユーザーが公開直前に作成）
12. DeepSeek による募集文の添削（MVPの最後・時間があれば。PMレビューのMVPリストには明記なし）

**やらないこと**: 2章「MVPでは入れないもの」のとおり。法務文面の起草もしない。

---

## 4. データベース定義（PostgreSQL / Supabase）

方針: 最小限。ID は外部に出るものを `uuid`、内部連番は `bigint identity`。時刻は `timestamptz`。
Laravel 標準テーブル（`migrations`, `sessions`, `cache`, `jobs`, `password_reset_tokens` 等）は Laravel の migration で別途作成。
**全アプリテーブルで RLS を有効化（ポリシー無し）** し、Supabase の Data API（anon key）から直接読めないようにする。アクセスは Laravel 経由のみ。

> 認証方式（8章）で `users` の扱いが変わる:
> - Laravel認証 → 下記 `users` をそのまま使う（Laravel側は `HasUuids`）。
> - Supabase Auth → `users.id` を `auth.users.id` と同値にし、`password_hash` は不要。

```sql
-- ユーザー
create table users (
  id                 uuid primary key default gen_random_uuid(),
  email              text not null unique,
  password_hash      text,                       -- Supabase Auth採用時は不要
  stripe_customer_id text unique,
  is_admin           boolean not null default false,
  status             text not null default 'active' check (status in ('active','suspended')),
  created_at         timestamptz not null default now(),
  updated_at         timestamptz not null default now()
);

-- プロフィール（1:1）
create table profiles (
  user_id      uuid primary key references users(id) on delete cascade,
  display_name text not null,
  bio          text,
  area         text,                              -- 例: 札幌 / 小樽
  user_type    text check (user_type in ('join','scout','both')), -- 参画したい/募集したい/両方
  skills       text[] not null default '{}',
  avatar_url   text,
  created_at   timestamptz not null default now(),
  updated_at   timestamptz not null default now()
);

-- 音声（ファイル名 = id。推測不能なランダムID）
create table voice_files (
  id           uuid primary key default gen_random_uuid(),
  user_id      uuid not null references users(id) on delete cascade,
  kind         text not null check (kind in ('intro','want')),  -- 自己紹介 / 一緒にやりたいこと
  storage_path text not null,
  mime_type    text not null,                     -- audio/webm or audio/mp4(iOS Safari)
  duration_ms  integer not null check (duration_ms > 0 and duration_ms <= 10500),
  size_bytes   integer not null,
  transcript   text,                              -- Whisper結果
  status       text not null default 'active' check (status in ('active','hidden')),
  created_at   timestamptz not null default now(),
  unique (user_id, kind)
);

-- 募集投稿
create table recruit_posts (
  id         bigint generated always as identity primary key,
  user_id    uuid not null references users(id) on delete cascade,
  title      text not null,
  body       text not null,
  area       text,
  status     text not null default 'open' check (status in ('open','closed','hidden')),
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);
create index on recruit_posts (status, created_at desc);

-- DM
create table conversations (
  id              bigint generated always as identity primary key,
  user_a          uuid not null references users(id) on delete cascade,
  user_b          uuid not null references users(id) on delete cascade,
  last_message_at timestamptz,
  created_at      timestamptz not null default now(),
  check (user_a < user_b),
  unique (user_a, user_b)
);

create table messages (
  id              bigint generated always as identity primary key,
  conversation_id bigint not null references conversations(id) on delete cascade,
  sender_id       uuid not null references users(id) on delete cascade,
  body            text not null,
  read_at         timestamptz,
  created_at      timestamptz not null default now()
);
create index on messages (conversation_id, created_at);
create index on messages (sender_id, created_at);  -- 1日3通の判定用

-- チケット（商品）
create table tickets (
  id           bigint generated always as identity primary key,
  seller_id    uuid not null references users(id) on delete cascade,
  title        text not null,
  description  text not null,
  duration_min integer not null check (duration_min in (30, 60)),
  price_jpy    integer not null check (price_jpy > 0),
  status       text not null default 'published' check (status in ('draft','published','archived')),
  sales_count  integer not null default 0,
  created_at   timestamptz not null default now(),
  updated_at   timestamptz not null default now()
);

-- チケット購入（会計記録なので退会時も残す: FKは set null + スナップショット）
create table ticket_purchases (
  id                         bigint generated always as identity primary key,
  ticket_id                  bigint references tickets(id) on delete set null,
  seller_id                  uuid references users(id) on delete set null,
  buyer_id                   uuid references users(id) on delete set null,
  title_snapshot             text not null,
  duration_min               integer not null,
  amount_jpy                 integer not null,
  platform_fee_jpy           integer not null,   -- 手数料率は設定値（未決: 20〜25%）
  seller_payout_jpy          integer not null,
  status                     text not null default 'pending'
    check (status in ('pending','paid','approved','completed','canceled','refunded')),
  stripe_checkout_session_id text unique,
  stripe_payment_intent_id   text unique,
  meeting_url                text,
  scheduled_at               timestamptz,
  payout_status              text not null default 'unpaid' check (payout_status in ('unpaid','paid')),
  paid_out_at                timestamptz,
  created_at                 timestamptz not null default now(),
  updated_at                 timestamptz not null default now()
);

-- 通報
create table reports (
  id          bigint generated always as identity primary key,
  reporter_id uuid references users(id) on delete set null,
  target_type text not null check (target_type in ('user','recruit_post','voice','message','ticket')),
  target_id   text not null,
  reason      text not null,
  status      text not null default 'open' check (status in ('open','resolved','dismissed')),
  resolved_by uuid references users(id) on delete set null,
  resolved_at timestamptz,
  created_at  timestamptz not null default now()
);

-- Stripe Webhook の冪等性（同じイベントを二重処理しない）
create table stripe_events (
  id           text primary key,                  -- evt_...
  type         text not null,
  received_at  timestamptz not null default now(),
  processed_at timestamptz
);

-- RLS（ポリシー無し = Data API からは不可視。Laravelはpostgresロールで接続）
alter table users            enable row level security;
alter table profiles         enable row level security;
alter table voice_files      enable row level security;
alter table recruit_posts    enable row level security;
alter table conversations    enable row level security;
alter table messages         enable row level security;
alter table tickets          enable row level security;
alter table ticket_purchases enable row level security;
alter table reports          enable row level security;
alter table stripe_events    enable row level security;
```

補足
- `status` 遷移（チケット）: `pending`（Checkout作成）→ `paid`（Webhook）→ `approved`（販売者が承認・meeting_url/日時入力）→ `completed` → 翌月末目安で手動振込し `payout_status='paid'`。`canceled`/`refunded` は管理者操作 or Webhook。会話の元案は `pending/approved/completed` + `reviewed`（レビューはMVP外なので削除）。
- NGワードはMVPでは設定ファイル（`config/ngwords.php`）で十分。
- `voice_files.duration_ms` 上限は10秒＋誤差。

---

## 5. 画面・API一覧

### 画面
| 場所 | 画面 |
|---|---|
| www | `/` LP、`/terms` 利用規約、`/privacy` プライバシーポリシー、`/tokushoho` 特商法表記（3つは **TODO プレースホルダ**） |
| app | 登録 / ログイン |
| app | マイページ・プロフィール編集（音声録音・再録音・削除、文字起こし表示） |
| app | プロフィール表示（音声プレイヤー「▶ 10秒で聞く」、通報ボタン） |
| app | 募集一覧（検索）/ 募集詳細 / 募集作成・編集（AI添削ボタンはMVP後半） |
| app | DM一覧 / DMスレッド（残り送信数表示） |
| app | チケット一覧 / 詳細 / 出品・編集（**手数料内訳と受取額を表示**） |
| app | 購入完了・キャンセル戻り先 / 購入履歴（買い手）/ 販売管理（売り手：承認・meeting_url入力・完了） |
| app | 退会（確認画面） |
| admin | 通報一覧・対応、ユーザー停止、購入一覧・振込済みチェック（Filament 基本機能でOK、凝った権限は不要） |

### API（`/api/v1`、JSON）
```
# 認証（方式未決。Laravel認証ならSanctum、Supabase AuthならJWT検証ミドルウェア）
POST   /auth/register        POST /auth/login        POST /auth/logout
GET    /me                   DELETE /me              # 退会（音声ファイル物理削除を含む）

# プロフィール・音声
GET    /profiles/{userId}    PUT /me/profile
POST   /me/voices/{kind}     # multipart, kind=intro|want, 10秒・サイズ上限チェック
DELETE /me/voices/{kind}
GET    /voices/{voiceId}     # ランダムIDで配信（ディレクトリ一覧不可、public直置きしない）

# 募集
GET    /posts?q=&area=       GET /posts/{id}
POST   /posts                PUT /posts/{id}         DELETE /posts/{id}
POST   /ai/proofread         # DeepSeek添削（MVP後半）

# DM
GET    /conversations        POST /conversations     # 相手userIdで作成or取得
GET    /conversations/{id}/messages
POST   /conversations/{id}/messages                  # 1日3通超過は 429

# チケット
GET    /tickets              GET /tickets/{id}
POST   /tickets              PUT /tickets/{id}       DELETE /tickets/{id}(=archived)
POST   /tickets/{id}/checkout                        # Stripe Checkout Session作成→URL返却
GET    /me/purchases         GET /me/sales
POST   /purchases/{id}/approve   POST /purchases/{id}/complete

# 通報
POST   /reports

# Stripe
POST   /stripe/webhook       # 署名検証必須、認証ミドルウェア・CSRF対象外

# 管理（is_admin）
GET /admin/reports  PATCH /admin/reports/{id}
PATCH /admin/users/{id}/suspend
GET /admin/purchases  PATCH /admin/purchases/{id}/payout
```

---

## 6. 実装の順番（LUNAはこの順で進める）

1. **リポジトリ雛形**: モノレポ `www/`（Astro）・`api/`（Laravel 11）・`docs/`。`.gitignore`（`.env`, `vendor/`, `node_modules/`, `storage/app/voices/`, `*.sql`, `*.dump`）、`api/.env.example`（値は空）。
2. **Supabase接続**: `DB_CONNECTION=pgsql`、`sslmode=require`。4章DDLを Laravel migration 化して適用。
3. **Stripe テストモードから着手**: シードした仮ユーザー2人で、チケット作成 → Checkout（`mode: payment`、`metadata` に purchase_id）→ 成功/キャンセル戻り → Webhook。
   - Webhook は `Stripe-Signature` を **必ず検証**（`STRIPE_WEBHOOK_SECRET`）。`stripe_events` で冪等化。
   - 処理イベント: `checkout.session.completed`（→ `paid`、`sales_count`+1）、`checkout.session.expired`（→ `canceled`）、`charge.refunded`（→ `refunded`）。
   - 金額はサーバ側で `tickets.price_jpy` から決定（クライアント値を信用しない）。手数料率は `PLATFORM_FEE_RATE` 設定値。
   - ローカル検証は Stripe CLI（`stripe listen --forward-to`）。
4. **認証**（8章の決定に従う。未決のうちは認証部分をミドルウェア1か所に閉じ込め、差し替え可能にしておく）。
5. **プロフィール** CRUD。
6. **募集投稿** CRUD + `ILIKE` 検索 + 一覧ページング。
7. **DM**（1日3通制限、JST日付で判定、超過は429）。
8. **音声**: MediaRecorder（`MediaRecorder.isTypeSupported` で `audio/webm`→不可なら `audio/mp4`。iOS Safari対策）、10秒で自動停止。サーバで長さ・サイズ・MIME検証 → ランダムID名で保存 → Whisper で文字起こし（`language: ja`）→ NGワード検査（ヒット時は `hidden` + 通報キュー）。
9. **通報 + NGワードフィルタ**（募集・DM・チケット文面・文字起こしに適用）+ **管理画面**。
10. **退会**: ユーザー削除（cascade）＋ 音声ファイル物理削除、購入記録は set null で保持。
11. **チケット画面の仕上げ**: 出品時の手数料内訳表示、販売者の承認・meeting_url入力・完了、管理画面の振込済みチェック。
12. **Astro LP**（www）＋ 法務ページ3つのプレースホルダ（本文は `TODO: ユーザーが公開直前に作成`）。アプリのフッター・購入画面からリンク。
13. **運用**: エラー監視（Sentry等）、Laravelログ（daily）、`pg_dump` バックアップスクリプト。
14. （余力があれば）DeepSeek 添削 `/ai/proofread`。

各ステップ完了ごとに動作確認手順を `docs/` に1行ずつ残す。デプロイ先が決まるまではローカル + Stripe テストモードで完結させる。

---

## 7. セキュリティ・運用の必須事項

- **カード情報は一切保存しない**。保存するのは `stripe_customer_id`、Checkout Session ID、PaymentIntent ID のみ。
- **Stripe Webhook 署名検証必須**（未検証だと偽通知で無料購入される）。冪等処理。
- **音声ファイル名は推測不能なランダムID**（会話初期案の `voices/{user_id}.webm` は不可）。Web公開ディレクトリに直置きしない。
- 通報ボタン（プロフィール・音声・募集・DM・チケット）＋ NGワードフィルタ。モデレーションは管理画面から（担当は運営者本人）。
- **退会機能必須**、音声ファイルも削除。
- **バックアップ**: Supabase 無料枠は自動バックアップ無し（会話の指摘）→ `pg_dump` を定期実行。ダンプは個人情報を含むので暗号化・非公開保存、リポジトリに入れない。
- **エラー監視・ログ**必須（500エラーに気づける状態）。
- 秘密情報は全て環境変数。public リポジトリにコミットしない。
- 認可: 自分のリソースのみ編集可、`is_admin` 以外は管理API不可、停止ユーザーは投稿・DM・出品不可。
- レート制限: ログイン・音声アップロード・Checkout作成・通報に Laravel の throttle。
- 手数料の内訳（手数料率＋Stripe決済手数料3.6%で売り手受取額が減る点）を出品画面で明示（「ぼったくり」と言われない対策、PMレビューより）。
- Stripe 本番切替は **法務ページ（特に特商法表記）公開後**。

---

## 8. 未決事項（LUNAは決めない。ユーザーの決定待ち）

### 8-1. API の置き場所【ユーザー指示により未決のまま】
| 案 | 利点（会話より） | 欠点・リスク |
|---|---|---|
| **DOMAINKING**（PHP 8.2、FTP） | 既に契約済み（月1,100円）、100GBあり音声保存に余裕、Laravel 11 が動く（PHP 8.2） | **SSH無し**→ローカルで `composer install` して `vendor/` ごとFTP、権限エラー地獄（PMレビュー）。※補足: `pdo_pgsql` 拡張の有無、外部DB（Supabase）への外向き接続、cron・キュー実行可否を要確認 |
| **Render.com 無料枠** | PMレビューの推奨（「10倍楽」）、git連携デプロイ、Supabaseと相性良 | ※補足（会話外の一般情報）: 無料枠はアイドル時スリープ→初回応答が遅い、ファイルシステムが永続しない→音声ファイルを外部ストレージ（例: Supabase Storage）に置く必要あり |

→ どちらでも動くよう、音声保存先は Laravel の `Storage` ディスク抽象で実装し、設定で切替可能にする。

### 8-2. その他の矛盾・未決
| # | 内容 | 会話内の揺れ | 本書での扱い |
|---|---|---|---|
| 1 | **app（フロント）の置き場所・技術** | Next.js/Nuxt（初期案）→ Vercel or DOMAINKING に Astro/Next.js → laravel/breeze から始める案も | **未決**。APIを先に作り、フロントは薄く。候補: (a) Vercel + Astro/Next.js、(b) DOMAINKING に静的書き出し、(c) Laravel Blade（breeze）でAPIと同居 |
| 2 | **認証方式** | Laravel（breeze）案 vs PMレビューの Supabase Auth | **未決**。認証層を差し替え可能に実装 |
| 3 | **ドメイン名** | 「あなたのドメイン」「yourdomain.com」のまま。`sapporo.<domain>` で切る案も | **未決**。環境変数化 |
| 4 | 手数料率 | 「20〜25%」「80%を販売者へ（=20%）」「25%+Stripe 3.6%で7,140円」 | **未決**。`PLATFORM_FEE_RATE` 設定値 |
| 5 | キャンセル・返金ポリシー | PMが「24時間前まで無料とか決めてないと揉める」と指摘のみ | **未決**。MVPは管理者がStripeダッシュボードで手動返金→Webhookで反映 |
| 6 | 販売者への振込情報の収集方法 | 「翌月末に手動振込、最初は銀行振込で十分」のみ | **未決**。口座情報をDBに持つか、サービス外（メール等）で受けるか |
| 7 | 日程調整の方法 | 「買う人が買って日程調整」のみ | 暫定: DM + 販売者が `meeting_url`/`scheduled_at` を入力。要確認 |
| 8 | 「DM 1日3通」の数え方 | 送信メッセージ3通か、新規会話3件か不明 | 暫定: **送信メッセージ3通/日（JST）**、設定値化。要確認 |
| 9 | Whisper の実行方法 | OpenAI Whisper API（10秒≈0.15円）vs ローカル whisper.cpp | 暫定: OpenAI Whisper API（SSH無し環境でローカル実行は現実的でないため）。要確認 |
| 10 | 収益モデル | サブスク300/500円 → 成功報酬5,000円/3,000円 → チケット制 | **解消済み**: チケット手数料のみ（最新）。サブスク・成功報酬は実装しない |
| 11 | DB | DOMAINKING MySQL 5.5 / 「MySQLのJSONで十分」→ Supabase | **解消済み**: Supabase PostgreSQL |
| 12 | 動画30秒必須 → 音声10秒 → 音声は任意 | — | **解消済み**: 音声10秒・任意 |
| 13 | 期間・範囲 | 4週間ロードマップ → 2週間MVP | **解消済み**: 2週間MVP（3章） |
| 14 | 検索 | Meilisearch → LIKE | **解消済み**: LIKE |
| 15 | マスター版の「PostgreSQL + pgvector」 | PMはpgvectorをリリース後に | **解消済み**: pgvectorはMVP外 |
| 16 | Embedding のモデル | OpenAI text-embedding-3-small vs bge-m3 | リリース後に決定 |
| 17 | DeepSeek添削の MVP 内外 | 「AI添削は喜ばれる」vs PM のMVPリストに記載なし | MVP最後の任意ステップ |
| 18 | バックアップの実行場所 | 「pg_dumpのcronメモ」のみ | **未決**（DOMAINKINGのcron可否、ユーザーPC等） |
| 19 | エラー監視ツール | Sentry が例示されたのみ | 暫定: Sentry 無料枠。要確認 |

---

## 9. ユーザー側でやること（LUNAにはできない）

- [ ] **8章の未決事項を決定**（特に API置き場所・app置き場所・認証方式・ドメイン・手数料率）
- [ ] **Supabase プロジェクト作成**（接続情報を `.env` に設定。LUNAへは値を渡さず、環境変数名だけで連携）
- [ ] **Stripe アカウント作成**（まずテストモード）、テスト用 API キーと Webhook シークレット発行。KYCは自宅住所で可（会話より）
- [ ] **DeepSeek API キー**、**OpenAI API キー**（Whisper用。ChatGPT Plus とは別課金）
- [ ] DOMAINKING: PHP 8.2 選択、FTP情報、（API置く場合）`pdo_pgsql`・外部DB接続・cron の確認（`phpinfo()`）
- [ ] （Render案なら）Render アカウント
- [ ] ドメイン・サブドメイン（www / app / api）の DNS 設定
- [ ] Sentry 等のアカウント（使う場合）
- [ ] **法務ページ本文**（利用規約・プライバシーポリシー・特商法表記）を公開直前に作成。特商法は「請求があれば遅滞なく開示」方式の予定。**LUNAは文面を書かない**
- [ ] 開業届（屋号 JoinJob）など事業側の準備（会話のPM指示）
- [ ] リリース日に投稿してくれる札幌・小樽の起業家への声かけ、ユーザーストーリー10個の整理（PMの「今日やること」）

---

## 10. 参考

- 企画会話（原典）: https://github.com/mboxxobm/memo/blob/main/001.md
- 実装先リポジトリ（現在空）: https://github.com/mboxxobm/JoinJob
- 参考サービス: https://www.ocosba.com/ ・ https://www.timeticket.jp/
