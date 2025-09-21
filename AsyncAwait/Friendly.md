Friendly（Codeer.Friendly）は、.NET アプリケーションの自動テストを行うためのライブラリです。特に WinForms や WPF アプリケーションの GUI テストに優れています。別プロセスで動作するアプリケーションを操作できる点が特徴的です。

## Friendlyの主な特徴

Friendlyは対象アプリケーションのプロセスにアタッチし、リフレクションを使ってオブジェクトを直接操作できます。これにより、座標指定のクリック操作ではなく、より確実で保守性の高いテストが書けます。

## 基本的な使用例

### 1. セットアップとアプリケーション接続

```csharp
using Codeer.Friendly;
using Codeer.Friendly.Windows;
using Codeer.Friendly.Dynamic;

// アプリケーションを起動してアタッチ
var app = new WindowsAppFriend(Process.Start("MyApplication.exe"));

// または既存のプロセスにアタッチ
var app = new WindowsAppFriend(Process.GetProcessesByName("MyApplication")[0]);
```

### 2. フォームとコントロールの取得

```csharp
// メインフォームを取得
var mainForm = app.Type().System.Windows.Forms.Application.OpenForms[0];

// コントロールを名前で取得
dynamic textBox = mainForm.Controls["textBox1"];
dynamic button = mainForm.Controls["button1"];

// または型とプロパティで検索
var textBox = mainForm.Controls.Cast<Control>()
    .Where(c => c.GetType().Name == "TextBox" && c.Name == "textBox1")
    .First();
```

### 3. コントロール操作の例

```csharp
// テキストボックスに値を設定
textBox.Text = "テスト入力";

// ボタンをクリック
button.PerformClick();

// コンボボックスの選択
dynamic comboBox = mainForm.Controls["comboBox1"];
comboBox.SelectedIndex = 2;

// チェックボックスの操作
dynamic checkBox = mainForm.Controls["checkBox1"];
checkBox.Checked = true;
```

### 4. WPFアプリケーションの場合

```csharp
using Codeer.Friendly.Windows;
using Codeer.Friendly.WPFm;

var app = new WindowsAppFriend(Process.Start("MyWPFApp.exe"));

// WPFのメインウィンドウを取得
var mainWindow = WindowControl.FromZTop(app);

// 名前でコントロールを検索
var textBox = new WPFTextBox(mainWindow.IdentifyFromName("textBox1"));
var button = new WPFButtonBase(mainWindow.IdentifyFromName("button1"));

// 操作
textBox.EmulateChangeText("テスト入力");
button.EmulateClick();
```

### 5. 実際のテストメソッドの例

```csharp
[Test]
public void ログイン画面のテスト()
{
    // アプリケーション起動
    var app = new WindowsAppFriend(Process.Start("LoginApp.exe"));
  
    try
    {
        // ログイン画面の取得
        var loginForm = app.Type().System.Windows.Forms.Application.OpenForms[0];
      
        // コントロールの取得
        dynamic userNameTextBox = loginForm.Controls["txtUserName"];
        dynamic passwordTextBox = loginForm.Controls["txtPassword"];
        dynamic loginButton = loginForm.Controls["btnLogin"];
      
        // 入力操作
        userNameTextBox.Text = "testuser";
        passwordTextBox.Text = "password123";
      
        // ログインボタンクリック
        loginButton.PerformClick();
      
        // 結果の検証
        // 新しいフォームが開かれることを確認
        Thread.Sleep(1000); // 画面遷移を待機
        Assert.AreEqual(2, app.Type().System.Windows.Forms.Application.OpenForms.Count);
    }
    finally
    {
        // アプリケーション終了
        app.Type().System.Windows.Forms.Application.Exit();
    }
}
```

### 6. カスタムクラスのメソッド呼び出し

```csharp
// アプリケーション内のカスタムクラスのメソッドを直接呼び出し
var businessLogic = app.Type().MyNamespace.BusinessLogic();
var result = businessLogic.CalculateTotal(100, 0.1);

Assert.AreEqual(110, (double)result);
```

### 7. データグリッドの操作例

```csharp
// DataGridViewの操作
dynamic dataGridView = mainForm.Controls["dataGridView1"];

// セルの値を取得
var cellValue = dataGridView.Rows[0].Cells[1].Value;

// セルに値を設定
dataGridView.Rows[0].Cells[1].Value = "新しい値";

// 行を選択
dataGridView.CurrentCell = dataGridView.Rows[0].Cells[0];
```

## Friendlyの利点

1. **プロセス間操作**: 別プロセスのアプリケーションを直接操作できる
2. **座標に依存しない**: コントロールを直接操作するため、レイアウト変更に強い
3. **高速**: Win32 APIを使った座標クリックよりも高速
4. **デバッグしやすい**: .NETのデバッガでステップ実行が可能

Friendlyを使用することで、より安定で保守しやすいGUIテストを作成できます。特に業務アプリケーションの自動テストにおいて威力を発揮します。
