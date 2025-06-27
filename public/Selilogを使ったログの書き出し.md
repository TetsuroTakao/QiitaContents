---
title: Serilogを使ったログの書き出し
tags:
  - 'Serilog'
  - '構造化ログ'
  - 'Generic Host'
  - 'C#'
  - 'Options Pattern'
private: false
updated_at: '2025年6月20日'
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
# 事前準備

Visual Studio Codeおよび.NET SDKをインストールし、Pathが通っていることを確認してください。<details><summary>Path設定</summary>Windwosスタートボタン（タスクバーのWindowsのアイコン）を右クリック - コンテキストメニュー：[システム] - 関連リンク：[システムの詳細設定] - [詳細設定]タブ：[環境変数]ボタン - [%ユーザー名%のユーザー環境変数]：[編集]ボタン - 「Code.exe」のパス定義
![環境変数](https://blog.processtune.com/wp-content/uploads/2025/06/環境変数の編集.png "環境変数名の編集")</details>任意のフォルダで右クリック後、Windowsターミナルを起動して「code .」を入力しVisual Studio Codeを起動します。※ここでは「SerilogSample」フォルダで作業を進めます。

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

## Serilogの構造化ログ

ここでは、簡単なサンプルとして.NET Aspireのスターターテンプレートを使って、構造化されたログを書き出してみます。.NET Aspireはコンテナを使った開発を行います（[.NET Aspire のセットアップとツール](https://learn.microsoft.com/ja-jp/dotnet/aspire/fundamentals/setup-tooling?tabs=windows&pivots=vscode&wt.mc_id=DT-MVP-4029060#container-runtime)）ので、DockerまたはPodmanをインストールしてください。また、Visual Studio CodeのExtensionとして[C# Dev Kit](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit)をインストールしておきます。

<details><summary>.NET Aspireのツール</summary>

![Docker Desdtop](https://blog.processtune.com/wp-content/uploads/2025/06/PodmanDesktop.png "Docker")
![Podman Desktop](https://blog.processtune.com/wp-content/uploads/2025/06/PodmanDesktop.png "Podman")
![C# Dev Kit](https://blog.processtune.com/wp-content/uploads/2025/06/CSharpDevKid.png "C# Dev Kit")
</details>
サンプルのフォルダとは別のフォルダでVisual Studio Codeを起動し、[統合ターミナル]で以下のコマンドを入力します。

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

Debugについては、launch.jsonとtasks.jsonを%新しいプロジェクトのルートフォルダ%\.vscodeに作成します。

```csharp:%新しいプロジェクトのルートフォルダ%\.vscode\launch.json
{
    "configurations": [
        {
            "type": "coreclr",
            "request": "launch",
            "name": "Launch AspireSample.AppHost",
            "program": "${workspaceFolder}/AspireSample.AppHost/bin/Debug/net9.0/AspireSample.AppHost.dll",
            "args": [],
            "cwd": "${workspaceFolder}/AspireSample.AppHost",
            "stopAtEntry": false,
            "preLaunchTask": "build"
        }
    ]
}
```

```csharp:%新しいプロジェクトのルートフォルダ%\.vscode\tasks.json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "build",
            "dependsOn": "dotnet build",
            "command": "dotnet",
            "args": [
                "build",
                "${workspaceFolder}/AspireSample.AppHost/AspireSample.AppHost.csproj",
                "--configuration",
                "Debug",
                "--no-restore"
            ],
            "type": "process",
        }
    ]
}
```

tasks.jsonのargsパラメーターについては[オフィシャルブログ「dotnet build」](https://learn.microsoft.com/ja-jp/dotnet/core/tools/dotnet-build)を参照ください。
Visual Studio Codeの[アクティビティバー]の[Run and Debug]を選択します。launch.jsonとtasks.jsonが%新しいプロジェクトのルートフォルダ%\.vscodeに作成してbuildする前は[Run and Debug]ボタンが表示されますが、[Run and Debug]ボタンを選択するとbuildが開始され、Aspireの管理画面のリンクが[統合ターミナル]に表示されます。

<details><summary>Run and Debugペイン</summary>
<img width="45%" title="Build後のRun and Debug" alt="Build後のRun and Debug" src="https://blog.processtune.com/wp-content/uploads/2025/06/VSCodeRunAndDebugWithoutButton.png">
<img width="45%" title="Build前のRun and Debug" alt="Build前のRun and Debug" src="https://blog.processtune.com/wp-content/uploads/2025/06/VSCodeRunAndDebugWithButton.png">
</details>
起動後に[デバッグコンソール]に出力されるリンクをクリックするとブラウザに管理画面が表示されます。

<details><summary>Aspireの実行</summary>
![Aspire管理画面](https://blog.processtune.com/wp-content/uploads/2025/06/AspireManageConsole.png "Aspire管理画面")

Webフロント側のエンドポイントを選択してAspireのWebアプリ画面を表示します。

![Aspire Web App](https://blog.processtune.com/wp-content/uploads/2025/06/AspireWebApp.png "Web App")

[Weather]メニューを選択して、API経由で値を取得してWeb画面に表示させることで分散ログを出力します。

![Aspire Web API](https://blog.processtune.com/wp-content/uploads/2025/06/AspireWebAPI.png "Web API")

管理画面に戻って[トレース]を選択すると、WebフロントのプロセスからAPIが呼び出されていることを確認できます。

![Aspire トレース画面](https://blog.processtune.com/wp-content/uploads/2025/06/ManageConsoleTrace.png "トレース画面")
</details>

このAspireソリューションに対して、SerilogSampleの時と同じようにファイルに出力します。Nugetパッケージを追加して、%新しいプロジェクトのルートフォルダ%\AspireSample.AppHost\program.csに以下のコードを追加します。




## オプションズ・パターン
[Options pattern in .NET](https://learn.microsoft.com/ja-jp/dotnet/core/extensions/options?wt.mc_id=DT-MVP-4029060)を使うと構造化した設定値をクラスにマップできるので、構造化ログの書き出しに使うと可視性の良いトレースログを出力することができます。構造化ログとオプションズ・パターンの組合せは、特にマイクロサービスなど複数のプロセス間通信を必要とするアプリケーションやメッセージング・ミドルウェア分散アプリケーションなどの複数ドメイン間をまたぐようなアプリケーション、APIサービスを使ったマッシュアップ・アプリケーションなど分散トレーシングが有用なアプリケーションのログの出力時に使うことができます。
