Friendlyが別プロセスのGUIを制御する仕組みについて、技術的な詳細を説明します。

## 基本的な仕組み

Friendlyは**プロセス間通信（IPC）**と**コードインジェクション**を組み合わせて、別プロセスのGUIを制御しています。

### 1. アーキテクチャの概要

```
[テストプロセス]          [対象アプリプロセス]
┌─────────────┐         ┌─────────────────┐
│ Friendly    │         │ 対象アプリ      │
│ Client      │◄────────┤                 │
│             │  IPC    │ ┌─────────────┐ │
│ - リフレク  │         │ │ Friendly    │ │
│   ション呼び出し  │         │ │ Server      │ │
│ - オブジェクト │         │ │ (注入された │ │
│   参照管理   │         │ │  ライブラリ)  │ │
└─────────────┘         │ └─────────────┘ │
                        └─────────────────┘
```

## 詳細な技術実装

### 1. プロセス接続とDLLインジェクション

```csharp
// Friendlyの内部的な処理（簡略化）
public class WindowsAppFriend
{
    public WindowsAppFriend(Process process)
    {
        // 1. 対象プロセスにDLLを注入
        InjectFriendlyDll(process);
      
        // 2. 名前付きパイプでIPC通信を確立
        EstablishConnection(process.Id);
      
        // 3. 対象プロセス内でFriendlyサーバーを初期化
        InitializeFriendlyServer();
    }
  
    private void InjectFriendlyDll(Process process)
    {
        // Win32 APIを使用してDLLインジェクション
        // - CreateRemoteThread
        // - WriteProcessMemory
        // - LoadLibrary
        IntPtr hProcess = OpenProcess(process.Id);
        IntPtr remoteAddr = VirtualAllocEx(hProcess, ...);
        WriteProcessMemory(hProcess, remoteAddr, dllPath, ...);
        CreateRemoteThread(hProcess, LoadLibraryAddress, remoteAddr);
    }
}
```

### 2. IPC通信メカニズム

Friendlyは主に**名前付きパイプ**を使用してプロセス間通信を行います：

```csharp
// 通信の流れ（概念的な実装）
public class FriendlyConnection
{
    private NamedPipeClientStream _pipe;
  
    public object InvokeMethod(string typeName, string methodName, object[] args)
    {
        // 1. メソッド呼び出し情報をシリアライズ
        var request = new MethodInvocationRequest
        {
            TypeName = typeName,
            MethodName = methodName,
            Arguments = args,
            ObjectId = GetObjectId()
        };
      
        // 2. パイプ経由で対象プロセスに送信
        SendRequest(request);
      
        // 3. 実行結果を受信
        return ReceiveResponse();
    }
  
    private void SendRequest(MethodInvocationRequest request)
    {
        byte[] data = SerializeRequest(request);
        _pipe.Write(data, 0, data.Length);
    }
}
```

### 3. 対象プロセス内でのFriendlyサーバー

注入されたFriendlyサーバーは対象プロセス内で以下の処理を行います：

```csharp
// 対象プロセス内で動作するFriendlyサーバー（概念的な実装）
public class FriendlyServer
{
    private readonly Dictionary<int, object> _objectCache = new();
    private readonly NamedPipeServerStream _pipe;
  
    public void ProcessRequests()
    {
        while (true)
        {
            // リクエストを受信
            var request = ReceiveRequest();
          
            try
            {
                // リフレクションを使用してメソッドを実行
                var result = ExecuteMethod(request);
              
                // 結果を送信
                SendResponse(new MethodInvocationResponse 
                { 
                    Result = result, 
                    Success = true 
                });
            }
            catch (Exception ex)
            {
                SendResponse(new MethodInvocationResponse 
                { 
                    Error = ex.Message, 
                    Success = false 
                });
            }
        }
    }
  
    private object ExecuteMethod(MethodInvocationRequest request)
    {
        // 1. 型を取得
        Type targetType = Type.GetType(request.TypeName) 
                         ?? Assembly.GetExecutingAssembly().GetType(request.TypeName);
      
        // 2. オブジェクトインスタンスを取得
        object targetObject = GetCachedObject(request.ObjectId) 
                             ?? Activator.CreateInstance(targetType);
      
        // 3. メソッドを実行
        MethodInfo method = targetType.GetMethod(request.MethodName);
        object result = method.Invoke(targetObject, request.Arguments);
      
        // 4. 結果オブジェクトをキャッシュ
        if (result != null)
        {
            int objectId = CacheObject(result);
            return new ObjectReference { ObjectId = objectId, Type = result.GetType() };
        }
      
        return result;
    }
}
```

### 4. オブジェクト参照管理

Friendlyは複雑なオブジェクト参照を効率的に管理します：

```csharp
public class ObjectManager
{
    private static int _nextObjectId = 1;
    private readonly Dictionary<int, WeakReference> _objects = new();
  
    public int RegisterObject(object obj)
    {
        int id = _nextObjectId++;
        _objects[id] = new WeakReference(obj);
        return id;
    }
  
    public object GetObject(int objectId)
    {
        if (_objects.TryGetValue(objectId, out var weakRef))
        {
            return weakRef.Target; // GCされていなければ取得
        }
        return null;
    }
  
    // ガベージコレクション対応
    public void CleanupDeadReferences()
    {
        var deadKeys = _objects.Where(kvp => !kvp.Value.IsAlive)
                              .Select(kvp => kvp.Key)
                              .ToList();
      
        foreach (var key in deadKeys)
        {
            _objects.Remove(key);
        }
    }
}
```

## GUI制御の具体例

### 1. コントロール操作の実装

```csharp
// テストプロセスでの呼び出し
dynamic textBox = app.Type().System.Windows.Forms.TextBox();

// 実際にはこのような処理が発生
// 1. IPC経由で対象プロセスにリクエスト送信
// 2. 対象プロセスでSystem.Windows.Forms.TextBoxを取得
// 3. プロパティやメソッドを実行
// 4. 結果をIPC経由で返送

textBox.Text = "Hello World"; // → SetProperty("Text", "Hello World")
string text = textBox.Text;   // → GetProperty("Text")
textBox.Focus();             // → InvokeMethod("Focus")
```

### 2. イベントハンドリングの仕組み

```csharp
// 対象プロセス内でのイベント処理
public class EventManager
{
    public void AttachEventHandler(object target, string eventName, int handlerId)
    {
        var eventInfo = target.GetType().GetEvent(eventName);
      
        // デリゲートを動的に作成
        var handler = CreateEventHandler(eventInfo.EventHandlerType, handlerId);
      
        // イベントにハンドラーをアタッチ
        eventInfo.AddEventHandler(target, handler);
    }
  
    private Delegate CreateEventHandler(Type delegateType, int handlerId)
    {
        return Delegate.CreateDelegate(delegateType, this, 
            nameof(OnEventOccurred));
    }
  
    private void OnEventOccurred(object sender, EventArgs e)
    {
        // イベント発生をテストプロセスに通知
        NotifyEventOccurred(sender, e);
    }
}
```

## パフォーマンス最適化

### 1. バッチ処理

```csharp
// 複数の操作を一度に実行して通信回数を削減
public void BatchExecute(params Action[] operations)
{
    var batch = new BatchRequest();
  
    foreach (var operation in operations)
    {
        batch.Add(ConvertToRequest(operation));
    }
  
    // 一度の通信で全て実行
    var results = ExecuteBatch(batch);
  
    ProcessBatchResults(results);
}
```

### 2. 非同期通信

```csharp
public async Task<T> InvokeAsync<T>(string method, params object[] args)
{
    var request = CreateRequest(method, args);
  
    // 非同期でリクエスト送信
    await SendRequestAsync(request);
  
    // 非同期で結果受信
    return await ReceiveResponseAsync<T>();
}
```

## セキュリティ考慮事項

1. **プロセス権限** : 対象プロセスと同等以上の権限が必要
2. **DLLインジェクション** : アンチウイルスソフトに検知される可能性
3. **メモリ保護** : DEP（Data Execution Prevention）やASLRの影響を受ける可能性

## まとめ

Friendlyの別プロセスGUI制御は以下の技術を組み合わせて実現されています：

1. **DLLインジェクション** : Win32 APIを使用して対象プロセスにライブラリを注入
2. **IPC通信** : 名前付きパイプによるプロセス間通信
3. **リフレクション** : .NETのリフレクション機能による動的なオブジェクト操作
4. **オブジェクト参照管理** : 効率的なメモリ管理とガベージコレクション対応

この仕組みにより、座標ベースの操作よりもはるかに確実で高速なGUI自動化が可能になっています。
