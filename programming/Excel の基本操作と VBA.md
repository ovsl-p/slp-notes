---
title: Excel の基本操作と VBA
aliases: [Excel の基本操作と VBA, Excel, エクセル, VBA, マクロ, Excel ショートカット]
domain: programming
tags: [excel, vba, shortcut, spreadsheet, office]
created: 2026-11-14
updated: 2026-11-14
status: growing
prerequisites: [セル参照と数式, 相対参照と絶対参照, ワークシートとブックの違い]
related: []
sources:
  - Microsoft サポート（Excel のキーボード ショートカット）
  - Microsoft Learn（VBA リファレンス）
---

# Excel の基本操作と VBA

## ひとことで

ショートカットは「移動・選択・入力」の3種類だけ覚えれば足り、マクロは「どの範囲を」「どう変えるか」をVBAで書く。

## なぜ効くか / どこで使うか

基本設計の資料は、シートが増える、列が増える、改訂が入る、の繰り返しになる。手で毎回やっている作業は、ショートカットとマクロでほぼ消せる。

特に効くのは、シートをまたぐ移動、数式の参照元をたどる作業、改訂履歴の更新になる。

## 前提となる知識

| 知っておくこと | なぜ必要か |
| --- | --- |
| セル参照（A1形式） | マクロで範囲を指定するときの基礎になる |
| 相対参照と絶対参照（`$`） | `F4` で切り替える。コピーしたときの挙動が変わる |
| ワークシートとブックの違い | `Worksheets` はシート、`Workbook` はファイル全体 |

## 本体の解説

### 1. よく使うショートカット

移動と選択（設計書では一番使う）:

| キー | 動作 |
| --- | --- |
| `Ctrl` + 矢印 | データの端まで移動 |
| `Ctrl` + `Shift` + 矢印 | 端まで範囲選択 |
| `Ctrl` + `PageUp` / `PageDown` | シートの切り替え |
| `Ctrl` + `Home` | A1 に戻る |
| `F5`（`Ctrl` + `G`） | セル番地を指定して移動 |
| `Ctrl` + `[` / `Ctrl` + `]` | 参照元 / 参照先へジャンプ（数式の追跡） |

入力と編集:

| キー | 動作 |
| --- | --- |
| `F2` | セルの編集 |
| `Ctrl` + `Enter` | 選択範囲すべてに同じ値を入れる |
| `Alt` + `Enter` | セル内で改行 |
| `Ctrl` + `D` / `Ctrl` + `R` | 上 / 左のセルをコピーして埋める |
| `Ctrl` + `;` | 今日の日付（改訂日入れに便利） |
| `Ctrl` + `Shift` + `;` | 現在時刻 |

書式と表示:

| キー | 動作 |
| --- | --- |
| `Ctrl` + `1` | セルの書式設定 |
| `Ctrl` + `Shift` + `3` | 日付形式 |
| `Ctrl` + `Alt` + `V` → `V` | 値だけ貼り付け（書式を持ち込まない） |
| `Ctrl` + `Shift` + `L` | オートフィルタの切り替え |
| `Alt` + `Shift` + `→` / `←` | 行のグループ化 / 解除（章の折りたたみ） |
| `F4` | 数式の参照を絶対参照に切り替え（`$A$1` → `A$1` → `$A1`） |

### 2. マクロの基本

`Alt` + `F11` で VBA エディタ、`Alt` + `F8` でマクロ一覧になる。マクロを含むブックは `.xlsm` で保存する。

最小の形はこれだけ。

```vb
Sub サンプル()
    Dim ws As Worksheet
    Dim lastRow As Long

    Set ws = ThisWorkbook.Worksheets("Sheet1")
    lastRow = ws.Cells(ws.Rows.Count, 1).End(xlUp).Row   ' A列の最終行
    ws.Range("B2:B" & lastRow).Value = "未確認"
End Sub
```

押さえるポイントは3つ。

`Cells(Rows.Count, 1).End(xlUp).Row` は、A列の最終行を下から上に探す定番の書き方になる。`UsedRange` は書式だけの空セルまで拾うので、最終行の判定には向かない。

`Application.ScreenUpdating = False` を処理の前に入れ、最後に `True` に戻すと、大量のセルを触るときに目に見えて速くなる。

`On Error GoTo` でエラー処理を入れる。シート名の取り違えは実行時にしか分からないので、最初に一度は確認させる。

### 3. よく書くマクロ

目次シートを自動で作る。基本設計では、シート名の一覧とリンクがそのまま目次になる。

```vb
Sub 目次を作る()
    Dim ws As Worksheet
    Dim toc As Worksheet
    Dim i As Long

    On Error Resume Next
    Set toc = ThisWorkbook.Worksheets("目次")
    On Error GoTo 0

    If toc Is Nothing Then
        Set toc = ThisWorkbook.Worksheets.Add(Before:=ThisWorkbook.Worksheets(1))
        toc.Name = "目次"
    End If

    toc.Cells.Clear
    toc.Range("A1").Value = "シート一覧"
    i = 3
    For Each ws In ThisWorkbook.Worksheets
        If ws.Name <> toc.Name Then
            toc.Hyperlinks.Add _
                Anchor:=toc.Cells(i, 1), _
                Address:="", _
                SubAddress:="'" & ws.Name & "'!A1", _
                TextToDisplay:=ws.Name
            i = i + 1
        End If
    Next ws
End Sub
```

全シートの同じ位置に改訂日を入れる。

```vb
Sub 改訂日を入れる()
    Dim ws As Worksheet
    Application.ScreenUpdating = False
    For Each ws In ThisWorkbook.Worksheets
        If ws.Name <> "目次" Then
            ws.Range("H2").Value = Date
        End If
    Next ws
    Application.ScreenUpdating = True
End Sub
```

複数シートの表を1枚に集める（列構成が同じ場合だけ使う）。

```vb
Sub 全シートを集計()
    Dim ws As Worksheet, dst As Worksheet
    Dim lastRow As Long, dstRow As Long

    Set dst = ThisWorkbook.Worksheets.Add
    dst.Name = "集計_" & Format(Now, "yyyymmdd_hhmm")

    For Each ws In ThisWorkbook.Worksheets
        If ws.Name <> dst.Name And ws.Name <> "目次" Then
            lastRow = ws.Cells(ws.Rows.Count, 1).End(xlUp).Row
            If lastRow >= 2 Then
                dstRow = dst.Cells(dst.Rows.Count, 1).End(xlUp).Row + 1
                ws.Range("A2:A" & lastRow).EntireRow.Copy
                dst.Cells(dstRow, 1).PasteSpecial xlPasteValues
            End If
        End If
    Next ws
    Application.CutCopyMode = False
End Sub
```

## 具体例・コード

基本設計でよく出る作業と、その打ち方の対応。

| やりたいこと | 手順 |
| --- | --- |
| 表の値だけを別シートに移す | `Ctrl` + `Alt` + `V` → `V` |
| 数式の参照元を確認する | `Ctrl` + `[` で参照元へ飛ぶ |
| 章ごとに折りたたむ | 行を選んで `Alt` + `Shift` + `→` |
| 改訂日を今日にする | `Ctrl` + `;` |
| 数式を一括で確認する | `Ctrl` + `` ` `` で数式表示、もう一度で戻る |

## つまずきポイント

- `UsedRange` で最終行を取る。書式だけのセルまで含まれるので、実際のデータより下を指すことがある。
- 相対参照のまま数式をコピーする。`F4` で絶対参照にしないと、行や列がずれる。
- マクロを `.xlsx` で保存する。マクロは `.xlsm` でないと保存されない。
- シート名を決め打ちで書く。シート名が変わると実行時エラーになるので、名前を定数にするか確認を入れる。
- 処理中に画面更新を切らない。数千行を触ると待ち時間が目に見えて変わる。

## 疑問・TODO

- [ ] テーブル（ListObject）を使えば、最終行の計算が要らなくなるか確認する
- [ ] Power Query で複数シートの統合を置き換えられるか試す
- [ ] 改訂履歴を自動で追記するマクロを書く

## 関連ノート

- [[Kubernetes の概要]] — 業務システムの構成を読む側の知識として

## 出典

- Microsoft サポート: Excel のキーボード ショートカット
- Microsoft Learn: Visual Basic for Applications リファレンス
