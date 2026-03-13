## 自動化

プログラムによるグラフ作成は、 **「データ作成」** 、 **「コア登録」** 、 **「トポロジ構築」** 、 **「ロジック同期」** の4つの段階に分かれています。

## 基礎
アトムは `AtomData` リソースなしでは存在できません。これはProgressDBとGraphLogicシステムのための「パスポート」です。
コードで生成する場合、既存のリソースをロードするか、新しいインスタンスを作成する必要があります。

```gdscript
var new_data = AtomData.new()
new_data.id = &"skill_fireball" # ユニークIDは必須
new_data.title = "ファイアボール"
```

#### 1. アトムの登録 (`add_atom`)

!!! warning
    標準の `add_child()` は使用しないでください。代わりに `graph.add_atom(node, branch)` メソッドを使用します。

*   ノードがGodotのシーンツリーに追加されます。
*   **AtomContext** （グラフの物理エンジン内のオブジェクト）が作成されます。
*   高速な計算のために、アトムに質量、温度、インデックスが割り当てられます。
*   アトムに `AtomData` がある場合、グローバルな `MedusaDB` に自動的に登録されます。

```gdscript
var atom = atom_template.instantiate()
atom.data = new_data
graph.add_atom(atom) # システムにアトムを登録
```

#### 2. トポロジの構築 (`connect_atoms`)
Medusaにおいて、接続（線）は単なる視覚効果ではありません。これらは物理挙動と利用可能ロジックを管理します。 `graph.connect_atoms(source, target)` メソッドは3つの機能を実行します。

1.  **物理:** ノード間に目に見えない「バネ」（ `SpringForce` ）を作成します。
2.  **ロジック:** `target` を `source` の子として、 `source` を `target` の親として記録します。これは `ConnectionsLogic` ににとって重要です。
3.  **視覚:** `GraphLine` にこれらの点座標の間に線を引くよう指示します。

```gdscript
graph.connect_atoms(parent_atom, child_atom)
```

#### 3. 状態の同期 (`sync_with_progress`)
アトムと接続の「網」を構築した後、デフォルトの状態は `LOCKED` です。ツリーを活性化し、どのスキルが利用可能かを確認するには、ロジック同期サイクルを実行する必要があります。

`graph.sync_with_progress(progress_db)` メソッドは、グラフにすべての `GraphLogic` ルールをチェックさせ、すべてのアトムの `status` を更新させます。

---
## 自動化と管理

*   コードで「スターター」スキルを作成するときに、 `&"root"` タグを追加します。
*   グラフで `TagLogic` を設定し、 `&"root"` タグを持つすべてのアトムに対して `AVAILABLE` 状態を許可します。
*   これで `sync_with_progress()` を呼び出した後、スタータースキルが自動的に点灯します。

### 動的な操作（リンクと切断）
ゲーム内の接続が変更される場合（例：動的な関係ネットワーク）、以下のメソッドを使用します。

*   `graph.disconnect_atoms(a, b)` — リンクを即座に削除し、ロзиックマップを更新します。
*   `graph.wake_up_atom(a)` — アトムを強制的に「スリープ」モードから解除します。リンク変更後に物理演算が即座にノードを押し始めるようにするのに役立ちます。

### フォーカスの監視
エディタや詳細情報パネルを作成するには、 `graph._focused_atom` 変数を使用します。 

*   これは、ユーザーが最後に選択したアトムへの参照です。
*   これを使用して、コンテキストメニューを簡単に実装できます（デモのように）： `panel.position = graph._focused_atom.position + offset` 。
