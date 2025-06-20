---
title: Serilogを使ったログの書き出し
tags:
  - 'Serilog'
  - '構造化ログ'
  - 'Generic Host'
  - 'C#'
  - 'Options Pattern'
  - '.NET Aspire'
  - 'Podman'
private: false
updated_at: '2025年6月20日'
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
# 事前準備

Visual Studio Codeおよび.NET SDKをインストールし、Pathが通っていることを確認してください。<details>Windwosスタートボタン（タスクバーのWindowsのアイコン）を右クリック - コンテキストメニュー：[システム] - 関連リンク：[システムの詳細設定] - [詳細設定]タブ：[環境変数]ボタン - [%ユーザー名%のユーザー環境変数]：[編集]ボタン - 「Code.exe」のパス定義
![環境変数](https://blog.processtune.com/wp-content/uploads/2025/06/環境変数の編集.png "環境変数名の編集")</details>任意のフォルダで右クリック-ターミナルを起動して「code .」を入力しVisual Studio Codeを起動します。※ここでは「SerilogSample」フォルダで作業を進めます。

私が簡単にサンプルアプリを作成するために事前に用意してあるリポジトリをダウンロードして使用していただいても構いません（.NET 9.0で作成してあります）。
```dotnetcli:Visual Studio Code Integrated Terminal
%Project root(SerilogSample)% > git clone https://github.com/TetsuroTakao/ConsoleAppBase
```
やっていることは、コンソールアプリ.NETテンプレートを作成して各種パッケージを追加したものです。
```dotnetcli:Visual Studio Code Integrated Terminal
%Project root(SerilogSample)% > dotnet new console
%Project root(SerilogSample)% > dornet new gitignore
%Project root(SerilogSample)% > dotnet add package Microsoft.Extensions.Hosting
%Project root(SerilogSample)% > dotnet add package Microsoft.Extensions.Logging
%Project root(SerilogSample)% > dotnet add package Microsoft.Extensions.Logging.Console
%Project root(SerilogSample)% > dotnet add package Serilog
%Project root(SerilogSample)% > dotnet add package Serilog.Extensions.Logging
%Project root(SerilogSample)% > dotnet add package Serilog.Sinks.File
```
Program.csに以下のコードを付けてあります。Dependency injectionを使用しないロギングのサンプルですのでアプリケーションの起動ログに利用できるようにしてあります。
```csharp:%Project root(SerilogSample)%\Program.cs
#region using
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Configuration;
using Serilog;
using Serilog.Sinks.File;
#endregion

HostApplicationBuilder builder = Host.CreateApplicationBuilder(args);
Log.Logger = new LoggerConfiguration()
    .WriteTo.File("logs/log.txt", rollingInterval: RollingInterval.Day)
    .CreateLogger();
ILoggerFactory factory = LoggerFactory.Create(builder =>
{
    builder.AddConsole();
    builder.AddSerilog(Log.Logger, dispose: true);
});
Microsoft.Extensions.Logging.ILogger logger = factory.CreateLogger<Program>();
builder.Configuration.Sources.Clear();
IHostEnvironment env = builder.Environment;
builder.Configuration.AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
    .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true, reloadOnChange: true)
    .AddUserSecrets<Program>(optional: true, reloadOnChange: true);
logger.LogInformation("Hello, World!");
using IHost host = builder.Build();
await host.RunAsync();
```
:::note info
logsフォルダは.gitignoreで除外してありますので、ダウンロード後プロジェクトルートにlogsフォルダを作成してください。
:::
## Serilogは出力のオプション
.NETのロギングは一般的にlog4netやNLogなど、[.NET Core および ASP.NET Core でのサードパーティ製のログ記録フレームワーク](https://learn.microsoft.com/ja-jp/aspnet/core/fundamentals/logging/?view=aspnetcore-9.0&wt.mc_id=DT-MVP-4029060#third-party-logging-providers)を使うことが多いと思います。Serilogは構造化されたログを出力できるので、そのようなロギングを行いたい場合に使うと有用です。
サンプルのロギングの仕組みとしては、Microsoft.Extensions.Loggingを使ってロギングを行いSerilogを使って出力するという仕組みです。サンプルのコードはこの出力の部分を差し替えられるように作成してあります。
```diff_c:%Project root(SerilogSample)%\プロジェクト名%.csproj
+ <program-name>.csproj
+ <PackageReference Include="NLog" Version="5.5.0" />
+ <PackageReference Include="NLog.Extensions.Logging" Version="5.5.0" />
- <PackageReference Include="Serilog" Version="4.3.0" />
- <PackageReference Include="Serilog.Extensions.Logging" Version="9.0.2" />
- <PackageReference Include="Serilog.Sinks.File" Version="7.0.0" />
```
```diff_c:%Project root(SerilogSample)%\Program.cs
+ using NLog.Extensions.Logging;
+ using NLog.Targets;
- using Serilog;
- using Serilog.Sinks.File;

ILoggerFactory factory = LoggerFactory.Create(builder =>
{
    builder.AddConsole();
+   var nlogConfig = new NLog.Config.LoggingConfiguration();
+   nlogConfig.AddRuleForAllLevels(new FileTarget("logfile") { FileName = $"{env.ContentRootPath}/logs/nlog.txt", Layout = "${longdate}|${level:uppercase=true}|${logger}|${message}" });
+   nlogConfig.AddRuleForAllLevels(new ConsoleTarget("logconsole"));
+   builder.AddNLog(nlogConfig);
-   builder.AddSerilog(Log.Logger, dispose: true);
});
- var logger = factory.CreateLogger<Program>();
+ var logger = NLog.LogManager.GetCurrentClassLogger();
builder.Configuration.Sources.Clear();
builder.Configuration.AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
    .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true, reloadOnChange: true)
    .AddUserSecrets<Program>(optional: true, reloadOnChange: true);
- logger.LogInformation("Hello, World!");
+ logger.Info("Hello, NLog!/" + env.EnvironmentName);
```

## Serilogの構造化ログとオプションズ・パターン
[Options pattern in .NET](https://learn.microsoft.com/ja-jp/dotnet/core/extensions/options?wt.mc_id=DT-MVP-4029060)を使うと構造化した設定値をクラスにマップできるので、構造化ログの書き出しに使うと可視性の良いトレースログを出力することができます。構造化ログとオプションズ・パターンの組合せは、特にマイクロサービスなど複数のプロセス間通信を必要とするアプリケーションやメッセージング・ミドルウェア分散アプリケーションなどの複数ドメイン間をまたぐようなアプリケーション、APIサービスを使ったマッシュアップ・アプリケーションなど分散トレーシングが有用なアプリケーションのログの出力時に使うことができます。
ここでは、簡単なサンプルとして.NET Aspireのスターターテンプレートを使って、構造化されたログを書き出してみます。.NET Aspireはコンテナを使った開発を行います（[.NET Aspire のセットアップとツール](https://learn.microsoft.com/ja-jp/dotnet/aspire/fundamentals/setup-tooling?tabs=windows&pivots=vscode&wt.mc_id=DT-MVP-4029060#container-runtime)）のでDockerまたはPodmanをインストールしてください。また、Visual Studio CodeのExtensionとして[C# Dev Kit](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit)をインストールしておきます。
<detail>
</detail>
サンプルのフォルダとは別のフォルダでVisual Studio Codeを起動し、ターミナルで以下のコマンドを入力します。
```dotnetcli:%新しいプロジェクトのルートフォルダ%
%新しいプロジェクトのルートフォルダ% > dotnet new aspire-starter
```
%新しいプロジェクトのルートフォルダ%\AspireSample.AppHost\appsettings.jsonを追加して以下の内容を書き込みます。以下のpodmanまたはdockerは、OCI（Open Container Initiative）互換コンテナランタイムをインストールした環境にあわせてどちらかを使用してください。.NET Aspireは既定でdockerを使用します。
```csharp:%新しいプロジェクトのルートフォルダ%\AspireSample.AppHost\appsettings.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Aspire.Hosting.Dcp": "Warning"
    }
  },
  "DOTNET_ASPIRE_CONTAINER_RUNTIME": "podman"
  // "DOTNET_ASPIRE_CONTAINER_RUNTIME": "docker"
}
```
Visual Studio Codeのコマンドバー（[F1]キー）で「aspire」を入力します。
