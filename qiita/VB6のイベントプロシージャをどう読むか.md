# VB6 のイベントプロシージャをどう読むか（VB.NET / C# 経験者向け）

GUIツールなしでのコードリーディングの機会があり備忘録
VB6 では、Form_Load` や `Command1_Click` といった「イベントプロシージャ」が登場しますが、VB.NET / C# の経験があれば、対応関係を押さえるだけで理解が楽になります

## 1. 読み方の基本（規則）
- 形式：**`<コントロール名>_<イベント名>`**
  - 例：`Form_Load`（フォームのロード時）, `Command1_Click`（ボタンのクリック時）, `Text1_Change`（テキスト変更時）
- マッピングの実体：**.frm はテキスト**。GUI定義とコードが同居し、命名規則で暗黙に結びつきます（画像等は `.frx` にバイナリ）
- 用語メモ：VB6 では正式には **「イベントプロシージャ」**。.NET 以降は **「イベントハンドラ」** が一般的。

```vb
' VB6 の典型例
Private Sub Form_Load()
  ' 初期化
End Sub

Private Sub Command1_Click()
  ' ボタンクリック時の処理
End Sub
```

## 2. 代表的なイベントの読み方と .NET 対応
他.NET言語 との読み方比較
※表作成にはGPT5の力を借りました

| VB6 | 実行タイミング | VB.NET（Handles） | C#（+=） |
|---|---|---|---|
| `Form_Load` | フォーム初期化（表示前）。コントロール操作OK | `Form1_Load Handles MyBase.Load` | `this.Load += Form1_Load;` |
| `Form_Activate` | フォーカス獲得時。軽い処理だけ | `Activated Handles MyBase.Activated` | `this.Activated += Form1_Activated;` |
| `Form_QueryUnload` | 閉じる直前の可否判断（キャンセル可） | `FormClosing Handles MyBase.FormClosing` | `this.FormClosing += Form1_FormClosing;` |
| `Form_Unload` | クローズ後処理（保存・解放） | `FormClosed Handles MyBase.FormClosed` | `this.FormClosed += Form1_FormClosed;` |
| `Form_Resize` | サイズ変更時のレイアウト調整 | `Resize Handles MyBase.Resize` | `this.Resize += Form1_Resize;` |
| `Command1_Click` | ボタンクリック | `Button1_Click Handles Button1.Click` | `this.button1.Click += Button1_Click;` |
| `Text1_Change` | テキスト値が変わるたび | `TextBox1_TextChanged Handles TextBox1.TextChanged` | `this.textBox1.TextChanged += TextBox1_TextChanged;` |
| `Text1_KeyPress` | 文字入力時（`KeyAscii`） | `KeyPress Handles TextBox1.KeyPress` | `this.textBox1.KeyPress += TextBox1_KeyPress;` |
| `Form_Paint` | 再描画時（重い処理禁止） | `Paint Handles MyBase.Paint` | `this.Paint += Form1_Paint;` |

## 3. GUI とコードの対応（.frm の中身を読む）
`.frm` では GUI もコードもテキストで並びます。IDE がなくても対応関係を読み起こせます。

```vb
' GUI 定義（抜粋）
Begin VB.CommandButton Command1
   Caption         =   "実行"
   Height          =   375
   Left            =   120
   Top             =   120
   Width           =   1335
End

' イベントプロシージャ
Private Sub Command1_Click()
    MsgBox "Hello VB6"
End Sub
```

- `Begin ... End` が **画面項目**、`Private Sub ... End Sub` が **イベント**
- .NET の `InitializeComponent()` + ハンドラ結合（`Handles` / `+=`）が、VB6 では **命名規則**(`<コントロール名>_<イベント名>`)で表現されている