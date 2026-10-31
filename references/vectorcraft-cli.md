# VectorCraft CLI：編集性を保つ操作メモ

Windows版で確認した手順。アプリの更新でスキーマが変わり得るため、インストール済みCLIの出力を優先する。ユーザー名・過去の案件パス・固定IDに依存しない。

## 接続と安全な実行

まず実行環境のPATHからVectorCraft CLIを探す。PowerShellでは`Get-Command vectorcraft-cli.exe -ErrorAction SilentlyContinue`、macOS/Linuxでは`command -v vectorcraft-cli`を使う。見つからない場合は、その環境のアプリのインストール先や公式の案内からCLIの場所を確認する。ユーザー名・ドライブ・インストール先を固定しない。以下はPATHから取得できた場合のPowerShell例。

```powershell
$vectorcraftCli = (Get-Command vectorcraft-cli.exe -ErrorAction Stop).Source
& $vectorcraftCli --help
& $vectorcraftCli commands
& $vectorcraftCli info 'input.svg'
```

CLIの`run`はヘッドレス処理。GUIで開いている未保存のドキュメントを編集したことにはならない。アクティブな未保存作業を対象にする依頼では、接続できるアプリAPIかGUIでその状態を扱う。

複数のJSON引数は、Pythonの`subprocess.run`に引数配列として渡すとPowerShellの引用符問題を避けられる。環境の利用可能なPythonを使い、特定のランタイムのパスを固定しない。

```python
import json
import subprocess

def run_vectorcraft(cli, source, commands):
    args = [str(cli), 'run', '--in', str(source)]
    for command, params in commands:
        args += ['--cmd', command, '--params',
                 json.dumps(params, ensure_ascii=False, separators=(',', ':'))]
    process = subprocess.run(args, capture_output=True,
                             encoding='utf-8', errors='replace')
    rows = [json.loads(line) for line in process.stdout.splitlines() if line.strip()]
    if process.returncode or any('error' in row or row.get('ok') is False for row in rows):
        raise RuntimeError(process.stdout + process.stderr)
    warnings = [warning for row in rows if isinstance(row.get('result'), dict)
                for warning in row['result'].get('warnings', [])]
    return rows, warnings  # 必ず呼出側で警告を読み、欠落や非対応を解決・説明する
```

現在のCLIでは`run`の結果はJSON Lines。依存する操作の戻り値を読んでから次の操作を組み立てる。ID取得が必要なら作業用`.vectorcraft`に保存し、次の呼び出しでそれを再開する。

## 構造を調べる・保存する

| コマンド | 用途・確認した引数例 |
|---|---|
| `document.inspect` | `{"depth":4,"childLimit":100}`。`layers`を再帰的に読み、`kind`、`name`、`id`、`locked`、`children`を調べる |
| `select.set` | `{"ids":[取得したID]}` |
| `select.none` | `{}` |
| `object.setProps` | `{"ids":[...],"name":"側面_01"}` |
| `layer.setProps` | `{"id":取得したID,"name":"05 側面","locked":false}` |
| `layer.setCurrent` | `{"id":レイヤーID}` |
| `document.save` | `{"path":"出力.vectorcraft","compress":true}` |
| `document.export` | SVGは`{"path":"出力.svg","outlineText":true,"preserveEditing":false,"decimals":5}`、PNGは`{"path":"確認.png","scale":3,"background":"transparent"}`など、要件に合わせる |

`outlineText`は互換用SVGで必要な場合の設定。編集用のライブテキストまで先に破壊しない。透過PNGは適切なアートボードで書き出す。

## 実レイヤーを作る

確認したVectorCraftでは、SVGルート直下の名前付き`g`が**トップレベルの実Layer**として読み込まれた。したがって、すべてが「Layer 1」配下のGroupだと思い込んで変換すると、最初の用途レイヤーを誤って分解してしまう。必ず取り込み後に`document.inspect`する。

- SVG側の名前には`id`と`data-name`を付ける。SVGの順序は背面から前面、レイヤーパネルは通常その逆。番号と重なり順を合わせる。
- すでに`kind: Layer`ならそのまま活用する。
- 案ごとの親Layerの中に用途別Groupがある場合、`layer.collectInNew {"ids":[groupId]}`でサブレイヤーへ移せる。戻り値の`id`を取得して、そのレイヤーを明示的に命名する。
- その後、元の**通常Groupだけ**を`select.set`し`object.ungroup`すると、不要なラッパーを外せる。複合パスやClip Groupは同じように解除しない。
- 背景・番号をロックする際は、対象LayerのIDを指定する。選択や現在のレイヤーが変わるので、暗黙の対象に依存しない。
- 同じ場所に重なるレイヤーを、番号合わせだけのために並べ替えない。非重複の複数案の親Layerは、`layer.move`や`layer.reverse`で番号順に整理できる。

## 重なった側面を整理する

前面形状を`F`、厚み表現の複製面を`S_i`とすると、見える側面は次の形を基準にできる。

`S = union(S_i) - F`

1. 作業コピーで側面だけを選択し、`object.pathfinder.unite {}`。
2. 前面の複製をその上に置き、統合側面と前面複製を選んで`object.pathfinder.minusFront {}`。元の前面は残す。
3. 必要な場合のみ`object.path.simplify {"tolerance":0.02,"cornerAngle":30,"straightLines":false}`。これは数百pxのアートで使った出発点であり、固定の正解値ではない。
4. `before`/`after`の点数と描画を比較する。統合結果に離れた面が残る場合は、面ごとの独立パスへ分離して命名する。一つのレイヤーに複数パスがあってよい。穴を構成する輪郭は一緒に保持する。

離れたパーツだけを含むことが確認できた複合パスは、`object.compoundPath.release`で解除できる。解除後は各子Pathに名前を付け、レイヤー数を増やす必要はない。親の透明度などが解除で失われる場合があるため、先に`transparency.info`等で親の属性を確認し、必要な属性を分離後のパスへ引き継ぐ。半透明部分や境界の重なりは必ず画像差分で確認する。中央の穴・アイコンの輪郭・クリップ用の複合パスに対して、同じ解除操作を一括適用しない。

多数の面の選択順や`minusFront`の前後関係は描画で確認する。階調が重要なら編集可能なグラデーションを使う。統合したからという理由だけで、承認済みの色を大きく変えない。

ガラスの表面の透明度だけ下げても、不透明な側面の全面複製が背後に残っていると透明にならない。側面を可視範囲に限定する。大量の半透明複製も重なりで濃くなる。

## 細い線・文字・パースの落とし穴

- `object.path.outlineStroke`で局所的な針状の突起が出た場合、元パスの始点と終点の近傍を調べる。ほぼ同位置の点への微小な線分と`Z`が連続すると、マイターの突起が生じることがある。原本は保持し、原因が確認できた箇所だけ作業コピーで修正する。一律の座標丸めや、全パスの終端削除はしない。
- パースには`object.distort`など、現在のカタログで使える機能を選ぶ。全体の変形を二重適用しない。見た目の維持が必要な細線は、必要に応じて変形前にアウトライン化し、アイコンの輪郭を目視する。
- `select.object.textObjects`が0件なら`type.createOutlines`を呼ばない。空選択ではコマンドが利用不可になる。
- 変形でテキストをアウトライン化する場合は、文字の順番・内容・元フォントを把握できる形で保持する。

## ガラス・屈折の整理

例：7つの材質レイヤー「アイコン／文字／ハイライト／反射縁／ガラス面／厚み／影」。背景のあるプレビューでは「屈折／番号／背景」を追加する。用途が異なればこの数に合わせる必要はない。

- 反射縁はクリップ内の少数のストローク、広い反射は塗りグラデーション、側面は連続した面ごとのパスとして扱える。離れた反射片や側面を一つの複合パスへ戻さない。
- クリップは保持し、その中に「下地／主線／補助線」などを命名する。離れた線は独立パスにし、主線・補助線の一括変更には名前付きGroupを使う。色や線幅が共通という理由だけで、離れた線を一つの複合パスにしない。
- 屈折の折れ線を簡略化する際は線だけを対象にする。開いたサブパスの端点を閉じたり、別の線と結合したりしない。
- グラデーション各点の透明度、フィルター、変形、合成順を維持する。各レイヤーへ分けるときに同じ親変形を二重に掛けない。
- `effect.list`で対応効果と適用状態を確認できる。`effect.expandAppearance`は必要な場合だけ使用し、編集可能なぼかしを無条件に画像へ変換しない。

## 仕上がりの判定

ネイティブを再読み込みし、各用途がLayerになっていること、対象の選択が容易なこと、過剰な重複が残っていないことを調べる。構造整理ではPNGの差分比較が有効。純粋な階層変更なら同一描画を目指し、曲線簡略化の微小差は全体と拡大表示で確認する。PNGや差分数値だけで編集性を判定しない。
