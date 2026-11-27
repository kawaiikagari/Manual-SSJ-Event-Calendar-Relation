# Google本人確認と利用許可

この資料は2026-11-27のローカル実装に対応します。実行ファイルはMain.gs、SheetSetup.gs、appsscript.jsonの3つです。独立した認証サービスや新しいHTMLファイルは作成しません。

## 対象と権限

| 対象 | 固定値 |
| --- | --- |
| 元スプレッドシート | `1-bmnwQxANON3jEu8G2qgXFf8R09zrZTagBmuKGuy1so` |
| 対象タブ | sheetId `0` |
| 反映先カレンダー | `10bbcc5e95ad3ebc84ca5245eabc6a817e21cc54898d0ebe61a9ea202e38442a@group.calendar.google.com` |
| 許可グループ | `ssj-sl-data-manager@googlegroups.com` |

処理を実行するGoogleアカウントは元シート所有者です。操作本人はGoogleのauthorization-code flowで確認したアカウントです。所有者の実行権限をアクセス者の本人情報へ置き換えません。Session.getActiveUser()が空なら所有者として許可しません。

スタッフには、指定グループへの有効な編集共有と、活動中の所属の両方が必要です。個別編集共有・担当者名簿・別グループだけでは利用できません。元シート所有者の管理権限は維持します。所有者のIDE実行や初回設定ではGoogleが返したActiveUserが実際の元シート所有者と一致する場合に限って利用できます。共有ドライブなど所有者を取得できない対象へ自動で緩和しません。

所属照会は直接所属と子グループを扱います。OWNER・MANAGER・MEMBERを有効とし、INVITED・PENDING・BANNEDはその経路で許可しません。最大30グループ、深さ5、各グループの子100件で探索を制限し、循環を終了させます。上限・照会不能・不正応答は確認不能として停止します。getRoleの例外だけで非メンバーと決めず、hasUserで非所属を確認できる場合に子グループへ進みます。担当者一覧は認可に使いません。

各サーバー操作で共有と所属を照会し、AIなど外部通信後と書き込み前にも再確認します。脱退・共有取消・期限切れ・確認不能では新たな変更を止めます。複数APIにまたがる変更は一括で巻き戻せないため、途中までのシート更新・カレンダー操作数を結果に残します。削除記録などのメタデータ更新も処理済み数に含めます。

## 所有者が行う更新と初回設定

以下は**次に所有者が行う操作**です。この依頼ではGoogle側の更新・承認・登録・デプロイ・試験書き込みを実施していません。

1. [対象スプレッドシート](https://docs.google.com/spreadsheets/d/1-bmnwQxANON3jEu8G2qgXFf8R09zrZTagBmuKGuy1so/edit?gid=0)を開き、URLのIDとsheetIdを確認します。「拡張機能 → Apps Script」から既存プロジェクトを開きます。別のプロジェクトやカレンダーへ切り替えません。
2. 既存のMainへ`src/Main.gs`、既存のSheetSetupへ`src/SheetSetup.gs`の全文を反映して保存します。Code.gsやIndex.htmlは追加しません。既存シートの再初期化や列追加は不要です。`setupEventDateColumns`は今回の認証更新では実行しません。
3. 左の歯車「プロジェクトの設定」でマニフェスト表示を有効にし、Google上の`appsscript.json`とローカルを**比較して差分を適用**します。Calendar v3・Sheets v4、Asia/Tokyo、V8、既存の業務スコープや独自設定を保持し、Drive v3サービス（識別子Drive）を追加します。Google上ですでにDrive v3があれば重複追加しません。追加は下表の2スコープです。Web設定は`executeAs: USER_DEPLOYING`、`access: ANYONE`へ合わせます。既存マニフェスト全体を無条件に上書きしません。
4. 元シート所有者のアカウントで、エディターの関数選択から`checkCalendarGoogleLoginGroup`を選び「実行」します。`ScriptApp.requireScopes`が必要な承認を要求します。承認画面は所有者本人が操作します。成功時は実行ログの`ok: true`、`sharing: writer`、`membershipQuery: completed`を確認します。`ownerIsDirectMember: false`でも照会成功を意味し、所有者権限は失われません。これは読み取り専用で、所属の追加・共有変更・予定の更新はしません。
5. 承認で止まった場合は必要な権限を承認して同じ診断を再実行します。API・グループ段階のエラーは後述のコードで確認します。Drive APIが無効な場合は、対象GASの「プロジェクトの設定」に実際に表示されるGCPプロジェクトを確認します。「デフォルト」ではDrive API v3サービスの追加・保存によってAPIが自動で有効化される仕様です。追加済みなら削除・再追加を繰り返しません。標準Cloudの場合は実際に紐付くプロジェクトでGoogle Drive APIを有効にします。別アプリのCloudへ変更したり、新しい名前のプロジェクトを推測して作成したりしません。
6. 「デプロイ → デプロイを管理」からこのアプリの既存Webデプロイを選び、新バージョンへ更新します。「次のユーザーとして実行」は**自分（元シート所有者）**、「アクセスできるユーザー」は**Googleアカウントを持つ全員**です。匿名を含む設定へ変更しません。未デプロイならWebアプリのデプロイが先に必要です。所有者以外のアカウントで代理デプロイしないでください。
7. 初回設定には、エディターから`openCalendarLoginSetup`を実行します。実行ログの「Googleログイン初回設定URL」を元シート所有者のGoogleアカウントで開きます。URLの末尾は`/exec?ssjSetup=google`です。シートのgetUi()は呼ばないため、シート内画面を使えない実行環境でもURLを取得できます。Webで所有者と確認できた場合に、既存の初回設定フォームを開きます。通常のWeb画面にある「所有者：Googleログインの初回設定」からも開けます。Webの本人メールが空なら所有者とみなさず、設定を許可しません。URL自体は認証情報ではありません。
8. 設定画面の「Googleに登録する戻り先URL」で「戻り先URLをコピー」を押します。値は実際のデプロイの`/exec?ssjLogin=google`です。URL取得不能と表示された場合は未デプロイや開発用URLを確認し、架空のURLを入力しません。
9. TourFinderで使った**既存のGoogle CloudプロジェクトとGoogle Auth Platform設定は共用できます**。そのプロジェクトの実物を確認し、下記「共通Cloudでカレンダー用OAuthクライアントを作成」の順序でウェブ用クライアントを追加します。GAS側のGCP「デフォルト」のひも付けは変更しません。Cloudの既存名を推測して登録したり、プロジェクトを作り直したりしません。
10. 作成直後の画面で設定JSONをダウンロードします。Googleは原則として作成画面を閉じた後に完全なシークレットを再表示・再ダウンロードしないため、その場で保存します。この秘密ファイルはリポジトリや共有フォルダーへ置かず、スタッフに配布しません。初回設定画面の「Googleのウェブ用クライアント設定JSON」でファイルを選びます。画面ではクライアントIDと戻り先の一致を確認し、シークレットは表示しません。
11. 「設定を保存」を押します。サーバーでも所有者・共有照会・JSON・戻り先を検証してScript Propertiesへ保存します。既存の本人確認セッションは無効になります。Googleで再確認して管理画面へ進みます。既存の「Gemini API Key」の入力・変更は不要です。
12. 本番の確認は別途、所有者と活動中のスタッフでログイン、非メンバー拒否、所属照会、期限切れ時の入力維持を確認します。実際のシート／カレンダーへ試験保存・反映する場合は、その対象と操作を先に決めてから行ってください。今回のローカルテストでは実施していません。

| 所有者へ追加するApps Scriptスコープ | 用途 |
| --- | --- |
| `https://www.googleapis.com/auth/drive.metadata.readonly` | 固定元シートのpermissionsをDrive.Permissions.list（登録済みDrive v3サービス）で読み、指定グループの有効なwriter共有を確認 |
| `https://www.googleapis.com/auth/groups` | GroupsAppで指定グループ・所属・子グループを読み取り |

既存の`spreadsheets`、`calendar.events`、`script.container.ui`、`userinfo.email`、`script.external_request`を保持しています。スタッフのGoogleログイン要求は`openid email`のみです。Googleのログイン設定と、所有者が業務APIへ与えるApps Scriptの承認は別です。マニフェストへgroupsがあるだけで所有者の追加承認が済んだとは扱いません。

## 共通Cloudでカレンダー用OAuthクライアントを作成

TourFinderのCloudプロジェクトとGoogle Auth Platformのブランド・同意画面設定は共用できます。同じプロジェクト内にカレンダー用のウェブクライアントを追加し、カレンダー側のScript Propertiesへその設定を保存します。TourFinderのソース・保存先・セッションキーには接続しません。ログイン画面のアプリ表示名はプロジェクト内の共通ブランド設定を使うため、クライアントの管理名だけを変えてもブランド名は変わりません。

1. カレンダーの初回設定画面で「戻り先URLをコピー」を押します。コピーするのは`/exec?ssjLogin=google`で終わる値です。設定画面を開く`?ssjSetup=google`のURLとは異なります。
2. [Google Auth Platformのクライアント一覧](https://console.cloud.google.com/auth/clients)を開き、画面上部で**TourFinderのクライアントが実際に載っているCloudプロジェクト**を選びます。プロジェクトの名前・IDを見て確認します。GASのデフォルトCloudを切り替える操作はしません。
3. そのプロジェクトでGoogle Auth Platformの登録が完了済みなら、ブランド・同意画面の初期登録をやり直しません。既存の公開範囲・表示名・連絡先は共通設定として扱い、他アプリへ影響する変更を勝手に行いません。
4. 「クライアントを作成」を押し、「アプリケーションの種類」で「ウェブ アプリケーション」を選びます。新しく作るクライアントの管理名は、この対象プロジェクトと同じ「SSJ Event Calendar Relation」とします。これは新規クライアントに付ける管理名で、既存の登録名がこの名前だと断定するものではありません。
5. このサーバー側コード交換方式では「承認済みのJavaScript生成元」への追加は不要です。「承認済みのリダイレクトURI」に手順1の完全なURLを追加します。TourFinderの戻り先をコピーしません。スタッフへのログイン要求は`openid email`のみです。
6. 「作成」を押し、**表示された作成完了画面を閉じる前に「JSONをダウンロード」**で保存します。秘密ファイルはGit管理外の自分だけが扱う場所に保存します。既存クライアントを閉じた後でシークレットが見えないことを理由に、TourFinderのシークレットを勝手にリセット・無効化しません。
7. カレンダーの初回設定画面へ戻り、「Googleのウェブ用クライアント設定JSON」で保存したファイルを選び、「設定を保存」を押します。JSONの中身やシークレットをチャットへ貼る必要はありません。

同じクライアントID・シークレットを複数アプリで使うことも技術的には可能ですが、当初の「TourFinderのシークレットを流用しない」という指定に合わせ、ここでは**Cloud・認証設定を共用し、クライアントをアプリごとに追加する**手順を示します。既存クライアントの共用へ変更する場合は、登録済みの戻り先、手元の完全な秘密設定、他アプリへの影響を確認してから手順を変えます。この資料だけでTourFinderの設定を変更しません。

[Google公式：同一プロジェクト内の複数クライアントとブランド設定](https://developers.google.com/identity/protocols/oauth2/cross-client-identity)、[ウェブ用クライアントの作成](https://developers.google.com/workspace/guides/create-credentials#web-application)、[作成時だけ取得できるシークレット](https://support.google.com/cloud/answer/15549257?hl=en)。

## スタッフの操作

1. シートの「SSJカレンダー → 管理パネルを開く」からWeb URLを開きます。「Googleでログイン」でGoogleの確認画面を開き、利用するアカウントを選びます。開けない場合は画面内の「Googleの確認画面を開く」を使います。
2. 「このブラウザーで本人確認を7日間保持する」は初回ONです。同じブラウザープロファイル・同じアプリで、本人確認した時点から最大7日間です。OFFは現在のタブだけ、最大6時間です。選択自体も次回へ保存します。利用のたびに延長はしません。
3. 確認用タブの終了を試み、元の管理画面で続行します。終了を拒否するブラウザーではそのタブを手動で閉じ、元のタブを選んでください。元のタブを閉じた場合は確認画面の「アプリを開く」で通常URLへ戻れます。
4. 保存をブラウザーが拒否した場合は、実際にはタブ内だけの保持であることを画面に表示します。保存前の検査後に拒否された場合も、サーバーの期限を最初の本人確認から6時間以内へ短縮します。タブ保存も拒否された場合はメモリー内だけで保持し、再読み込みで失います。
5. 期限切れでは入力・対象選択・期間・AI設定を残して再確認を開きます。同じアカウントを選んでください。別アカウントなら編集中の操作を引き継がず停止します。切り替える場合はログアウトしてから利用します。
6. 失敗した保存・削除・カレンダー反映・AI生成は再認証後も自動再実行しません。「経過を表示」で結果を確認してから、必要な操作を手動で行います。処理開始後に画面の応答だけ失われる場合もあるため、結果を確認せず保存を連打しないでください。
7. 「ログアウト」でブラウザーの本人確認情報と管理画面の内容を消去します。処理実行中はログアウトを無効にしています。共有・所属が取り消された場合は次の操作から利用できません。

入力内容は再認証中に同じ画面内へ保持します。タブを閉じた後に未保存入力を復元する機能ではありません。ブラウザーによるセッション復元・保存消去・Google側のアカウント切替でも再確認が必要になる場合があります。

## 安全なエラーの読み方

| コード | 何を確認するか |
| --- | --- |
| `AUTH_REQUIRED` / `AUTH_SESSION_EXPIRED` | 本人確認または再確認。入力を残し、失敗操作を自動再送しない |
| `AUTH_OWNER_EXECUTION_REQUIRED` | 元シート所有者がデプロイした所有者実行のWebか |
| `AUTH_SETUP_REQUIRED` | この対象アプリ用のJSONが初回設定で保存済みか |
| `AUTH_DEPLOYMENT_URL_REQUIRED` | 実際の本番`/exec`を取得できるデプロイがあるか |
| `AUTH_CREDENTIAL_FILE_INVALID` | ウェブ用JSONか、コピーした戻り先が完全一致するか |
| `AUTH_ACCOUNT_MISMATCH` / `AUTH_BROWSER_CHANGED` | 元の画面・同じアカウントとブラウザーの文脈で再確認したか |
| `AUTH_GOOGLE_CANCELLED` | Googleでキャンセル。再確認する場合は利用者が再操作 |
| `AUTH_GOOGLE_UNAVAILABLE` / `AUTH_GOOGLE_RESPONSE_INVALID` | 通信・Google応答検証の失敗。自動再送しない |
| `AUTH_STORAGE_UNAVAILABLE` | Script Propertiesの容量・保存・読出し失敗。所有者へ連絡 |
| `ACCESS_GROUP_NOT_SHARED` | 元シートの指定グループ共有が有効な編集権限か |
| `ACCESS_DENIED` | 活動中の所属を確認できなかった。確認不能のエラーとは区別 |
| `ACCESS_CHECK_UNAVAILABLE` | 下の詳細コードで照会段階を確認。非メンバーと断定しない |
| `SSJ_CAL_ACCESS_DRIVE_SCOPE_REQUIRED` | 所有者がIDEで診断を実行し、Driveメタデータ読取を承認 |
| `SSJ_CAL_ACCESS_DRIVE_SERVICE_UNAVAILABLE` | このGASにDrive API v3サービスが識別子Driveで登録されているか確認 |
| `SSJ_CAL_ACCESS_DRIVE_API_DISABLED` | 対象GASに実際に紐付くCloudのDrive APIを確認 |
| `SSJ_CAL_ACCESS_DRIVE_HTTP_403` 等 | 共有情報の取得段階で拒否。グループ照会の失敗と混同しない |
| `SSJ_CAL_ACCESS_GROUP_LOOKUP_FAILED` | 所有者のGroups追加承認、指定グループの実在・取得権限を確認 |
| `SSJ_CAL_ACCESS_GROUP_MEMBERSHIP_FAILED` | グループ取得後の在籍照会段階。承認・メンバー閲覧権限を確認 |
| `SSJ_CAL_ACCESS_GROUP_CHILDREN_FAILED` / `GROUP_SEARCH_LIMIT` | 子グループの読取または探索上限。自動で許可を緩めない |
| `JOB_ALREADY_CLAIMED` | 同じ本人の同じ処理番号は受け付け済み。経過・結果を確認 |
| `OPERATION_BUSY` | 別の変更処理が実行中。完了後に手動で操作 |

生のHTTP応答、Googleのトークン、クライアントシークレットはエラーに含めません。共有・グループ照会の安全な詳細コードはサーバーから画面まで保持します。診断関数はrequireScopesをAPI例外をまとめるcatchの前に呼ぶため、承認要求を単なるグループエラーへ置き換えません。

## 公開関数とRPCの境界

末尾が`_`の関数はGASのクライアント呼出し対象外です。末尾が`_`でない公開関数は以下の27個です。テストで一覧を固定し、追加した場合は境界の再確認を要求します。

| 関数 | 境界・処理 |
| --- | --- |
| `doGet` | 空の画面／Google callbackのみ。本人確認前に予定・通知文・秘密設定・経過を埋め込まない |
| `getCalendarLoginAvailability` | 設定済みか・実在アプリURL・所有者の設定可否だけ |
| `beginCalendarGoogleLogin` | 形式・ブラウザー文脈・開始回数を制限してOAuth開始 |
| `pollCalendarGoogleLogin` | stateと開始ブラウザーのproof・binding、設定版、所属を確認して回収 |
| `getCalendarSessionStatus` | 有効なアプリセッションと現在の利用許可を確認 |
| `endCalendarGoogleLogin` | セッション・binding・ブラウザー照合後にそのセッションだけ無効化 |
| `limitCalendarSessionToTab` | 照合後に保持期限を短縮するだけ。延長・データ取得不可 |
| `calendarAuthenticatedCall` | 本人を確定したうえで下記の7操作だけを呼出し |
| `getCalendarLoginSettings` / `configureCalendarGoogleLogin` / `checkCalendarGoogleLoginGroup` | 元シート所有者だけ。シークレットを応答へ返さない |
| `configureCalendarManualUrl` | 元シート所有者だけが独立した説明書のHTTPS掲載URLを登録・解除。HTML本文は受け取らない |
| `getWebCalendarData` / `getWebCalendarProgress` / `runWebCalendarAction` | 現在の管理権限が必要。直接呼出しも拒否。Webの通常操作は認証済みRPC経由 |
| `runCalendarPeriodAction` / `previewSpreadsheetCalendar` / `generateSpreadsheetGroupNotices` / `syncSpreadsheetToCalendar` | 旧経路も配下で管理権限を確認。新しいWebはこれらを直接呼ばない |
| `openCalendarManager` | 空のリンク案内だけ。シートUIを使えない場合は通常Web URLをログへ出す。実処理は所有者実行Webへ移動 |
| `openCalendarLoginSetup` | 所有者だけが実在デプロイの初回設定URLをログへ出す。getUi()には依存しない |
| `setupEventDateColumns` | 所有者だけが列設定を変更。通常のWeb RPCへ公開しない |
| `previewSpreadsheetCalendarFromMenu` / `generateGroupNoticesFromMenu` / `syncSpreadsheetToCalendarFromMenu` | 旧メニュー入口。安全なWebリンクを開くだけ |
| `onOpen` | メニュー追加だけ。承認・所属照会なし |
| `onEdit` | 本物の対象シート編集イベントのみ。クライアントJSONの偽イベントでは処理しない |

認証済みRPCの許可リストは`getWebCalendarData`、`getWebCalendarProgress`、`runWebCalendarAction`、`getCalendarLoginSettings`、`configureCalendarGoogleLogin`、`checkCalendarGoogleLoginGroup`、`configureCalendarManualUrl`です。任意関数名、クライアント指定メール、内部実行フラグでは権限を得られません。保存・削除・確認・反映・通知文のactionも固定リストです。

本物のonEditは承認済みWeb操作とは別です。対象のネイティブSpreadsheet/Rangeオブジェクト、authMode、紐付く元シートを確認し、通常通知文だけを更新します。Groups/Drive照会やAI通信はしません。トリガー実行中の内部状態はprivateな同期呼出し内だけで設定し、必ず解除します。新しい定期トリガーは追加しません。

## セッション・ジョブ・秘密情報

元シートの共有情報は、登録済みのDrive v3サービスの`Drive.Permissions.list`から取得します。HTTPでの直接取得や別の認証情報への自動切替は行いません。固定元シート、共有の有効期限、ページ送りの上限、指定グループのwriter共有という確認条件は維持します。診断ログには取得経路を表示します。API無効エラーが続く場合も、原因が解消したとみなしたり、共有確認を省略して先へ進めたりしません。サービスを使う実環境での成功は別途確認が必要です。

Google本人確認はstate・nonce・サーバーPKCE（S256）と、元画面のランダムproof・bindingを使います。一時状態は最大10分、同じGoogle側ブラウザー文脈で10分6回、全体で10分60回の開始制限です。コード交換は短いロックで一度だけ確保し、ロックを解放して固定のGoogle HTTPS token endpointへPOSTします。IDトークンはこの認証済み直接応答からだけ受け取り、issuer・audience・azp・expiry・iat・nonce・subject・確認済みemailを検証します。ブラウザーからIDトークンを受け取ってBase64展開する方式ではありません。直接TLS応答を信頼する方式であり、任意に持ち込まれたJWTの署名検証機能ではありません。

ブラウザーの保存キーは`ssj-event-calendar-relation-auth-v1:`と実在するアプリURLを組み合わせます。保持選択を別キーで保存し、セッションはランダムなopaque tokenとランダムbinding、保持区分だけを保存します。Googleのaccess token・ID token・クライアントシークレットはブラウザー保存へ入れません。タブはnoopener相当を保ち、アプリ用認証トークンをURLへ入れません。

| 保存先 | 専用キー・寿命 |
| --- | --- |
| Script Properties | `SSJ_CAL_GOOGLE_CLIENT_ID` / `SSJ_CAL_GOOGLE_CLIENT_SECRET` / `SSJ_CAL_GOOGLE_LOGIN_VERSION` |
| Script Properties | `SSJ_CAL_MANUAL_URL`。所有者が後から登録する独立した説明書のHTTPS URL。本人確認前のHTMLへ含めず、認証済みの一覧応答で返す |
| Script Properties | `SSJ_CAL_AUTH_SESSION_V1_`＋アプリセッショントークンのSHA-256ハッシュ。本人・期限・発行時刻・bindingのハッシュ・Googleの一時ユーザーキーのハッシュ・設定版を保存。生tokenをキーにしない |
| ScriptCache | `SSJ_CAL_AUTH_V1_`。OAuth一時状態・開始回数・回収用結果。早期消去ならログインをやり直すが、既存の7日間セッションは消えない |
| Script Properties / ScriptCache | `SSJ_CAL_JOB_V1_`＋確認済み本人メールのハッシュ＋jobId。重複送信記録と結果要約は1日、詳細経過は30分 |
| Script Properties | `SSJ_CAL_WRITE_LEASE_V1`。変更処理の期限付き排他リース。通常終了で削除、異常終了は7分で期限切れ |

セッションは発行時刻から7日／6時間に固定し、設定版やブラウザー照合が変われば無効です。新規ログイン時にこのアプリの期限切れ・旧設定版のセッションを整理し、最大400記録、保存量約430KBを超える場合は停止します。ジョブは新規受付時に期限切れを整理し、最大250記録・約420KBで制限します。保存失敗・破損・照会不能を許可に転換しません。他用途のプロパティや`Gemini API Key`を削除・移行しません。

ジョブ受付のScriptLockは短い記録確保だけに使い、Googleコード交換・Drive照会・Gemini送信中には保持しません。業務の変更処理は期限付きリースで競合を防ぎ、AI応答待ちの間は変更リースも解放して保存時に内容を再照合します。本物のonEditも同じリースと競合し、承認不要の通知文生成中に共有ロックを握り続けません。操作ログは確認済み本人・action・jobIdで記録します。

## 確認済みと未確認

2026-11-27に接続済みDriveツールで対象ファイルのメタデータを読み取りました。元シートの表示名は**SSJ Events List**、所有者は`kawaiikagari@gmail.com`、指定グループの共有roleは**writer**でした。共有は変更していません。このメタデータ応答だけではGoogle上の最新コード、実行時の追加承認、全ての共有有効期限や在籍照会の成功までは確認できません。

利用者から提示された同日18:37:12の実行ログでは、Drive.Permissions.listを使う診断がok: true、sharing: writer、scopeRequestCompleted: true、membershipQuery: completed、ownerIsDirectMember: trueで完了しました。これは対象GASでのDrive共有読取と所有者による直接所属照会の成功を示します。その後、初回設定の旧getUi()呼出しが18:39に失敗したため、初回設定をWeb URLから開く実装へ変更しました。

次は未確認です：対象GASとCloudの実際のひも付け、Google上の現在のマニフェスト・コード・デプロイ全体、使えるOAuthクライアントとその名前・戻り先登録、スタッフや子グループの実際の所属照会、所有者の本番カレンダー編集権限、修正後の初回設定Webフォーム、ブラウザーからGoogle確認・タブ終了・7日間保持を通す動作。診断成功をログイン全体の成功とは報告しません。

ローカルテストはGoogle／Gemini応答とブラウザー要素を模擬して実コードを実行します。実際の同意画面やOAuth設定は操作せず、シート・カレンダーへ試験書き込みしません。SL-TourFinderは認証部分の読み取り専用参考として使用し、他リポジトリを変更しません。

## 公式仕様の参照

- [Google OpenID Connect](https://developers.google.com/identity/openid-connect/openid-connect)：直接HTTPSのcode交換、IDトークンとnonce・宛先等の確認。
- [OAuth 2.0 Web server flow](https://developers.google.com/identity/protocols/oauth2/web-server)：ウェブ用クライアントと完全一致のredirect URI。
- [ScriptApp.requireScopes](https://developers.google.com/apps-script/reference/script/script-app)：所有者による追加承認要求。Webのcatchに閉じ込めずIDEから実行。
- [GroupsApp Group](https://developers.google.com/apps-script/reference/groups/group)・[Role](https://developers.google.com/apps-script/reference/groups/role)：所属・例外・子グループとロール。
- [Drive permissions.list](https://developers.google.com/workspace/drive/api/reference/rest/v3/permissions/list)：共有メタデータの照会。
- [Web app manifest](https://developers.google.com/apps-script/manifest/web-app-api-executable)：実行主体とログイン済み利用者のアクセス区分。
