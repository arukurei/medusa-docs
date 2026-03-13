# GraphLogic

**GraphLogic** システムは、グラフの「脳」として機能します。これにより、数百のノードに対して個別に条件を記述することなく、その挙動を自動化できます。以下では、独自のルールの作成方法、優先順位の仕組み、および注意すべきポイントについて詳しく解説します。

`GraphLogic` を継承したすべてのロジック・スクリプトは、パイプライン内のフィルターとして機能します。グラフが更新されるとき、各 **Atom (アトム)** はこのルールのパイプラインを通過します。

## 1. `get_permission` メソッド：誰に権限があるか？
このメソッドは、「このルールはこの Atom についてどう判断するか？」という問いにのみ答えます。Atom の状態を直接変更するのではなく、`Return` 型の値を返します：

*   **`Return.BLOCK` (優先順位 1):** 強力な拒否（否決）。少なくとも 1 つのルールが `BLOCK` を返すと、他のルールが「許可」していても、その Atom は決して取得可能にはなりません。「互換性のないスキル」やシナリオ上の制限に使用します。
*   **`Return.ALLOW` (優先順位 2):** 許可。ルールは「この Atom は開放可能である」と判断します。Atom が `AVAILABLE`（取得可能）になるには、拒否（BLOCK）がない状態で、少なくとも 1 つの許可が必要です。
*   **`Return.PASS` (優先順位 3):** 中立。ルールは「どちらでもよい。他のルールに任せる」と判断します。これがデフォルト値です。

!!!example
    グラフ上に `ALLOW` を返すルールが 1 つも存在しない場合、誰も明示的な許可を与えないため、すべての Atom は `LOCKED`（ロック中）のままになります。

---
## 2. `apply_status` メソッド：視覚状態の管理
`get_permission` がステータスを返すだけなのに対し、`apply_status` メソッドは Atom のプロパティ（`status`、`modulate`、`visible` など）を実際に変更する場所です。


**応用ロジックの例:**
```gdscript
@tool
class_name HiddenFogLogic extends GraphLogic

func apply_status(atom: Atom, graph: Graph, progress: ProgressDB = null) -> bool:
	# もし「秘密 (secret)」タグを持ち、開放された隣接ノードがない場合
	if atom.tags.has(&"secret") and not graph.has_unlocked_neighbor(atom, progress):
		atom.status = Atom.Status.HIDDEN
		atom.visible = false # 物理演算は動作し続けるが、見た目は非表示にする
		return false # チェーンを中断し、他のルールが LOCKED や AVAILABLE に変更するのを防ぐ
	
	atom.visible = true # 条件を満たさない場合は表示
	return true # 次のルール（例：ProgressLogic）がステータスを詳細化するのを許可
```

!!! warning
    **順序が重要:** `graph_logics` 配列内のルールは、上から下へと実行されます。

!!! note
     このメソッドは `bool` を返します。
        `false` を返すと、グラフはこのルールが決定的であると判断し、その Atom に対する以降のルールのチェックを**停止**します。
        `true` を返すと、グラフはリスト内の次のルールのチェックを続行します。

---
## 3. ヒントとテクニック
`atom.get_all_tags()` メソッドを使用してください。これは、`Atom` ノードに手動で設定されたタグと、`AtomData` リソース内にあるタグを結合します。これにより、特定のアイコンだけでなく、リソースのクラス全体に対して機能するルールを作成できます。

#### ロジック内での ProgressDB の使用
ロジックは `progress` データベースに直接アクセスできます。取得済みかどうかだけでなく、レベルもチェックできます。

```gdscript
# get_permission メソッド内
if progress.get_level(&"base_intelligence") < 5:
    return Return.BLOCK # このスキルに必要な知力が不足している
```

#### `graph.has_unlocked_neighbor()` メソッド
これはスキルツリーにおける「黄金のメソッド」です。すべての親（または `ConnectionDir.OUTBOUND` を指定した場合は子）を自動的にチェックし、そのうちの少なくとも 1 つが進捗データベースで開放されているかどうかを判定します。

## グラフ内のルールの推奨順序
スキルツリーを安定して動作させるには、`graph_logics` を以下のように配置します：

1.  **BlockLogics** (いかなる条件でも許可されないものを禁止する)。
2.  **SpecialRules** (ツリーの根ノード用の `TagLogic` など)。
3.  **ConnectionLogic** (接続の確認：次のスキルが購入可能か？)。
4.  **ProgressLogic** (最終確認：このスキルは既に購入済みか？)。

この順序により、購入済みのスキルは常に `UNLOCKED` のままになり、接続がまだ届いていない場合は `LOCKED` になり、シナリオ上の `BlockLogic` がブロックした場合は正しく `DISABLED` になります。
