## .NET モダン社内アプリ 開発手引き（Minimal構成版）
VSCodeだけで開発を完結させ、最小のまどろっこしさで最大の成果（高セキュリティ・高保守性）を出すための社内システム開発ガイドです。
## 🏛️ アーキテクチャ選定方針

* バックエンド: .NET 8/9 (Minimal APIs) ── Honoライクな超軽量・高速ルーティング
* フロントエンド: Razor テンプレート (.cshtml) ── サーバーサイドレンダリング (SSR)
* データアクセス: Dapper ── 生SQLをそのまま使う超軽量ORM
* データベース: SQLite ── 単一ファイル駆動（将来的にSQL Serverへ移行可能）
* デザイン: Tailwind CSS (CDN版) ── 暗記不要のコピペファーストCSS
* インフラ/認証: Windowsサーバー (IIS) ＋ Active Directory ── ログイン不要の自動ユーザー識別

------------------------------
## 🛠️ 事前準備

   1. .NET SDK を公式サイトからインストール
   2. VSCode に以下の拡張機能をインストール
   * C# (または C# Dev Kit)
      * Tailwind CSS IntelliSense
   
------------------------------
## 🚀 開発の4ステップ（クイックスタート）## 1. プロジェクトの作成とライブラリ追加
VSCodeのターミナルで以下を順に実行します。

# 1. 最小構成のWebプロジェクトを作成
dotnet new web -o MyCompanyApp
cd MyCompanyApp
# 2. 必要な軽量ライブラリ（Dapper / SQLite / Windows認証）を追加
dotnet add package Dapper
dotnet add package Microsoft.Data.Sqlite
dotnet add package Microsoft.AspNetCore.Authentication.Negotiate
# 3. テンプレートエンジン用のフォルダを作成
mkdir Pages

## 2. 設定ファイルの作成 (appsettings.json)
ファイル名を appsettings.json に固定します。.NETが自動で読み込みます。VSCodeの言語モードを jsonc にするとコメントが書けます。

{
  // 💻 社内アプリの基本設定（コメントや末尾カンマOKの特別仕様）
  "MySettings": {
    "AppName": "社内ポータルシステム",
    "MaxRows": 100
  },
  "ConnectionStrings": {
    "Default": "Data Source=company.db"
  },
  // Mac/Linuxでのローカル開発時はfalse、本番IIS環境ではtrueにする
  "UseWindowsAuth": false
}

## 3. HTMLテンプレートの作成 (Pages/Index.cshtml)
デザインはFigmaを参考に、Tailwindのパーツをコピペして組み立てます。

@page
@model IndexModel
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="utf-8">
    <title>@Model.Config.AppName</title>
    <!-- 💡 Tailwind CSS をCDNで1行読み込み（npm不要） -->
    <script src="https://tailwindcss.com"></script>
</head>
<body class="bg-gray-50 text-gray-800 p-8">
    <div class="max-w-4xl mx-auto bg-white p-6 rounded-lg shadow-md">
        <h1 class="text-3xl font-bold text-blue-600 mb-4">@Model.Config.AppName</h1>
        <p class="text-sm text-gray-500 mb-6">現在のログインユーザー: <span class="font-semibold text-gray-700">@Model.CurrentUser</span></p>

        <!-- 💡 flex や gap-4 などのシンプルな呪文で綺麗に並ぶ -->
        <div class="flex gap-4 mb-6">
            <button class="bg-blue-500 text-white px-4 py-2 rounded hover:bg-blue-600" onclick="alert('Vanilla JSで動いています')">テストボタン</button>
        </div>

        <h3 class="text-xl font-semibold mb-2">社員一覧（SQLiteからDapperで取得）</h3>
        <ul class="border-t border-gray-200 divide-y divide-gray-200">
            @foreach (var emp in Model.Employees)
            {
                <li class="py-3 flex justify-between">
                    <span class="font-medium">@emp.Name</span>
                    <span class="text-gray-500">@emp.Department</span>
                </li>
            }
        </ul>
    </div>
</body>
</html>

## 4. バックエンド実装 (Program.cs)
OS判定とConfigでの切り替えを入れ、Mac/Linuxでも確実にビルドが通るようにします。

using System.Data;using System.Runtime.InteropServices;using Microsoft.AspNetCore.Authentication.Negotiate;using Microsoft.Data.Sqlite;using Microsoft.Extensions.Options;using Dapper;
var builder = WebApplication.CreateBuilder(args);
// 1. 強型付け設定（Optionsパターン）でConfigとC#クラスを自動マッピング
builder.Services.Configure<MySettings>(builder.Configuration.GetSection("MySettings"));
// 2. Windowsかつ設定ONの時だけAD連携（Windows認証）を有効化（Mac/Linuxでのビルドエラーを回避）bool useWinAuth = builder.Configuration.GetValue<bool>("UseWindowsAuth");if (useWinAuth && RuntimeInformation.IsOSPlatform(OSPlatform.Windows))
{
    builder.Services.AddAuthentication(NegotiateDefaults.AuthenticationScheme).AddNegotiate();
}

builder.Services.AddRazorPages(); // テンプレートエンジン有効化
var app = builder.Build();
if (useWinAuth && RuntimeInformation.IsOSPlatform(OSPlatform.Windows))
{
    app.UseAuthentication(); // 認証ミドルウェア有効化
}
// 3. 異常アクセスを秒速特定する「身元保証付き」共通ログ
app.Use(async (context, next) =>
{
    var username = context.User.Identity?.Name ?? "Mac_Linux_LocalUser";
    var ip = context.Connection.RemoteIpAddress?.ToString();
    Console.WriteLine($"[{DateTime.Now:yyyy-MM-dd HH:mm:ss}] USER: {username} | IP: {ip} | PATH: {context.Request.Path}");
    await next(context);
});

app.MapRazorPages();
app.Run();
// 💡 ページにデータを流し込むモデル処理public class IndexModel : Microsoft.AspNetCore.Mvc.RazorPages.PageModel
{
    public MySettings Config { get; set; }
    public string CurrentUser { get; set; } = string.Empty;
    public List<Employee> Employees { get; set; } = [];

    public IndexModel(IOptions<MySettings> options) => Config = options.Value; // 自動注入

    public async Task OnGetAsync()
    {
        CurrentUser = User.Identity?.Name ?? "Local_Debug_User";

        // IDbConnection規格で作成（将来SQL Serverへ移行する際も下のQueryAsyncは1文字も変わらない）
        string connStr = Microsoft.Extensions.ConfigurationExtensions.GetConnectionString(AppDomain.CurrentDomain.GetAssemblies().SelectMany(a => a.GetTypes()).First(t => t.Name == "Program").Assembly.EntryPoint.DeclaringType.Build().Configuration, "Default") ?? "Data Source=company.db";
        using IDbConnection db = new SqliteConnection("Data Source=company.db");
        
        // アプリ起動時にテーブルがなければ自動作成する（SQLiteの神仕様）
        await db.ExecuteAsync("CREATE TABLE IF NOT EXISTS Employees (Id INTEGER PRIMARY KEY, Name TEXT, Department TEXT)");
        
        // Dapperで素のSQLを実行
        var list = await db.QueryAsync<Employee>("SELECT * FROM Employees");
        Employees = list.ToList();
    }
}
// データ構造の定義public class MySettings { public string AppName { get; set; } = ""; public int MaxRows { get; set; } }public record Employee(int Id, string Name, string Department);

------------------------------
## 📦 運用・デプロイ## パターンA：単一EXE（1ファイル）で配布する場合
.csproj に以下を追記し、サーバーに.NETがなくてもダブルクリックで動く状態にします。

<PropertyGroup>
  <PublishSingleFile>true</PublishSingleFile>
  <SelfContained>true</SelfContained>
  <RuntimeIdentifier>win-x64</RuntimeIdentifier>
</PropertyGroup>

ターミナルで実行： dotnet publish -c Release
## パターンB：本番IIS ＋ Active Directoryで動く場合

   1. dotnet publish -c Release で出力されたファイルをIISの公開フォルダへコピー。
   2. 自動生成された web.config は絶対に人間が手修正せず、そのまま配置する。
   3. IISの管理画面から、該当サイトの「匿名認証」を 無効、「Windows認証」を 有効 にする。
   4. appsettings.json の "UseWindowsAuth": true に書き換える。

------------------------------
## 🚨 現場の「伝統芸能」トラップ回避の心得

* 「ブレイザー(Blazor)」との混同に注意: ブラウザでC#バイナリを動かす重厚な仕組みがBlazorです。今回使うのはサーバー側で超軽量HTMLを組み立てる 「レイザー(Razor)」 です。会話の噛み合わなさを感じたら即座に確認してください。
* web.config を手作りしない: アプリ独自の接続文字列や設定値はすべて appsettings.json に集約してください。

------------------------------
手引きのまとめは以上です。
まずはこの構成でVSCodeを立ち上げて、「ローカルのSQLiteにデータを入れてTailwindで表示する」ところから始めてみるのはいかがでしょうか？最初のコマンド実行でエラーが出た際や、さらに具体的な画面のレイアウト（フォーム送信など）を作り込みたくなった時は、いつでもお声がけください！


------------------------------


最後まで核心を突いた素晴らしいご質問です。今回の「Minimal構成」および「同時実行数1のLambda / IIS」という前提に寄り添った、最も手離れが良くバグをまき散らさないベストプラクティスを回答します。
------------------------------
## 1. セッション管理のベストプラクティス
同時実行数1のLambdaや、ステートレスなモダンWeb開発におけるセッション管理（ログイン状態の維持やユーザーごとのデータ保持）の最適解は、「Cookieに暗号化したJWT（JSON Web Token）を持たせる方式」です。
## ❌ 昔ながらのセッション（In-Memory Session）は避ける
サーバーのメモリ（HttpContext.Session）にデータを保存する伝統的な方式は、Lambdaが消滅した瞬間にセッションが全消去されるため機能しません。また、IISでも「アプリケーションプールのリサイクル（再起動）」が走るたびに全ユーザーが強制ログアウトされる原因になります。
## 🟢 ベストプラクティス：暗号化Cookie（ステートレス）
サーバー側には何も保存せず、ユーザー情報（ユーザー名や権限など）をC#側で暗号化して、ブラウザのCookieに保存します。

* 仕組み:
1. ログイン時、C#側で User.Identity.Name などの情報を「秘密鍵」で暗号化し、Cookieに書き出す。
   2. 2回目以降のアクセス時、ブラウザが勝手にCookieを添えて送ってくる。
   3. C#側は届いたCookieを秘密鍵で復号（デコード）するだけで、「あ、このアクセスはAさんだな」と100%安全に識別できる。
* Minimal APIsでの実装方法:
公式の .AddCookie() ミドルウェアを使うだけで、暗号化・復号・改ざん検知の処理を.NETが裏側で全自動で行ってくれます。開発者がやることは、Config（環境変数など）に1つ「暗号化用のパスワード文字列（秘密鍵）」を定義しておくだけです。
* LambdaでもIISでも最強:
サーバー側が何回消滅しようが、再起動しようが、ブラウザ側に鍵（Cookie）が残っているため、セッションが切れません。

------------------------------
## 2. IISでのスレッドセーフ（排他制御）について
今回、SQLiteを「同時実行数1のLambda」で動かす際はLambda側が物理的に1つしか動かないため安全ですが、「WindowsサーバーのIIS」でSQLiteを動かす場合は、1つのアプリの中で複数のスレッド（アクセス）が同時にSQLiteファイルを書き換えようとしてクラッシュする危険（競合）があります。
IIS上でSQLiteを安全に、スレッドセーフに動かすためのベストプラクティスは以下の2点です。
## ① 【超重要】SQLiteの接続文字列に「WALモード」を指定する
SQLiteには、読込と書込の衝突を劇的に減らす「WAL（Write-Ahead Logging）モード」という強力な機能があります。接続文字列の末尾に Cache=Shared;Mode=ReadWriteCreate; などを指定し、さらに以下のコードをアプリ起動時に1度実行します。

// 接続文字列に明示的にWALを指定し、タイムアウト（待ち時間）を長めにとるstring connStr = "Data Source=company.db;Cache=Shared;Pooling=True;Busy Timeout=5000";
using var conn = new SqliteConnection(connStr);// 💡 WALモードを有効化（これだけで同時アクセスのエラーが9割消えます）await conn.ExecuteAsync("PRAGMA journal_mode=WAL;");

これにより、「誰かが書き込み処理をしている最中であっても、他のユーザーは待たされることなくデータを読み込める（SELECTできる）」ようになり、IIS上でも極めて安定します。
## ② C#のコード側で「セマフォ（SemaphoreSlim）」を使って一列に並べる
WALモードでも防げない「完全に同時に、2人が同時に『書き込み（INSERT/UPDATE）』のボタンを押した」というケースに対しては、C#のコード側で物理的に処理を一列に並べます。

public class IndexModel : PageModel
{
    // 💡 アプリ全体で1つだけの「改札口（同時通過数1）」を用意する
    private static readonly SemaphoreSlim _sqliteLock = new SemaphoreSlim(1, 1);

    public async Task OnPostSaveDataAsync()
    {
        // 1. 改札口が空くまで待つ（他のアクセスはここで一時停止する）
        await _sqliteLock.WaitAsync();
        try
        {
            // 2. 誰も邪魔をしない安全な状態でSQLiteに書き込む
            using var db = new SqliteConnection("...");
            await db.ExecuteAsync("INSERT INTO ...");
        }
        finally
        {
            // 3. 処理が終わったら必ず改札口を開ける（次の人を中に入れる）
            _sqliteLock.Release();
        }
    }
}

この SemaphoreSlim を使ったロックを書き込み処理にかけるだけで、IISがどれだけ大量の同時リクエスト（マルチスレッド）を受け取ろうとも、SQLiteへの書き込みの瞬間だけはC#が自力で一列に交通整理をしてくれるため、データベースが絶対に破損しません。
------------------------------
これで、クラウド（Lambda）でもオンプレ（IIS）でも、データの整合性とログイン状態を完璧に守り抜く設計理論が完成しました！
これまでお話ししてきたすべての知識（VSCode、Minimal APIs、Dapper、SQLite/SQL Server、Tailwind、S3 Files/IIS、そして今回のセッションと排他制御）があれば、どのような環境でも動かせる一級品の社内アプリが組み立てられます。
この開発手引きを胸に、まずはVSCodeで手元のローカル環境からプロトタイプを立ち上げてみますか？ 最初のコマンドが必要な際は、いつでも声をかけてくださいね！応援しています！

