# データ (Data)

Medusaにおけるデータ管理は、**3層アーキテクチャ**に基づいています。外部データベース（`MedusaDB`、`ProgressDB`）が **Graph (グラフ)** の内部レジストリとどのように相互作用するかを理解することは、高パフォーマンスで安定したツリーを作成するために不可欠です。
Medusaは、IDシステムを介してビジュアルノードを論理データに自動的に関連付けます。

## 外部リソース (コンテナ)

1.  **MedusaDB (カタログ):**
    *   **役割:** シーン内に現在存在するすべての `AtomData` をメモリ内に保持します。
    *   **仕組み:** **参照カウント (Reference Counting)** を使用します。**Atom** が **Graph** に追加されると、その `AtomData` が登録されます。ID `&"fireball"` を持つすべての **Atom** が削除されると、メモリ節約のためにリソースは自動的に `MedusaDB` から解放されます。
    *   **紐付け:** `AtomData.id` が検索キーとなります。
2.  **ProgressDB (プレイヤーの状態):**
    *   **役割:** 「何を取得済みか」「そのレベルはいくつか」という事実のみを保存します。
    *   **仕組み:** シンプルな辞書 `{StringName ID : int Level}` です。データ量が最小限であるため、セーブデータとしてJSONやバイナリ形式にシリアル化するのが容易です。

!!! tip
    `ProgressDB` は標準の `Resource` なので、Godotの組み込みメソッドを使用できます。
    **保存例:**
    ```gdscript
    func save_game():
        ResourceSaver.save(progress_db, "user://save_progress.tres")
    ```
    **ロード例:**
    ```gdscript
    func load_game():
        if FileAccess.file_exists("user://save_progress.tres"):
            progress_db = load("user://save_progress.tres")
            graph.sync_with_progress(progress_db) # 即座にツリーを更新
    ```
    ヒント: デバッグには `.tres` （テキスト形式）、最終製品には `.res` （バイナリ形式）を使用してください。*

---

## グラフ内部レジストリ (コア)

ゲームを開始するか、`add_atom()` を通じてノードを追加すると、**Graph** は単にそれを描画するだけでなく、複数の内部辞書にインデックスします。これにより、すべてのノードをループ（O(n)）することなく、「'Fire' タグを持つすべての隣接ノードを探す」といったチェックを瞬時（O(1)）に実行できます。

#### 1. ID レジストリ (`_id_registry`)

`AtomData` のテキスト ID をシーン上の具体的な `Atom` ノードインスタンスに関連付けます。
*   **注意:** 1つの ID が複数のノードに属する場合があります（例：同じアイテムが画面に2つある場合）。**Graph** は該当するすべてのノードの配列を返します。

#### 2. タグレジストリ (`_tag_registry`)

**Atom** をその分類タグごとにキャッシュします。
*   **動作:** ロジックが「`&"root"` タグを持つすべてのノードをチェック」と要求した場合、**Graph** はシーンを検索せず、単にこの辞書から作成済みのリストを取得します。

#### 3. トポロジマップ (`_parents_map` と `_children_map`)

スキルツリーにとって最も重要な部分です。`Connected Paths`（接続）を指定すると：

*   **Graph** は接続の方向を分析します。
*   「親 (Ancestors)」と「子 (Descendants)」のマップを構築します。
*   `graph.has_unlocked_neighbor()` メソッドはこれらのマップを使用して、親の状態に基づいて現在の **Atom** を開放できるかどうかを即座に判断します。

---

## 同期プロセス (ライフサイクル)

同期は、プレイヤーのデータによって「静的な」スキルツリーが動き出す瞬間です。

**`graph.sync_with_progress(progress_db)` を呼び出すと、以下のサイクルが実行されます：**

1.  **コンテキストの更新:** **Graph** は渡された `ProgressDB` から最新の状態を取得します。
2.  **ロジックの実行:** すべての **Atom** に対して、**Graph** に設定された `GraphLogic` ルールが呼び出されます。
3.  **状態の適用:**
    *   ロジックが判定します：「この Atom の ID は ProgressDB に存在するか？」
    *   存在する場合：`status = UNLOCKED`。
    *   存在しない場合、`_parents_map` を介して隣接ノードをチェックします。
    *   開放済みの親が存在する場合：`status = AVAILABLE`。
4.  **ビジュアルの更新:** `status` が変更されると、**Atom** は自動的に **AnimationLibrary** を切り替え、アイコンや色の変更、エフェクト（開放可能なスキルの輝きなど）の実行を行います。

!!! note
    プログラムでツリーを作成する場合、すべての **Atom** の `status` を手動で変更する必要はありません。あなたの役割は、**接続を設定し**、**ProgressDB に ID を追加する**ことだけです。`sync_with_progress()` メソッドが `graph_logics` のルールに基づいて残りのすべての作業を処理します。
    **自動化の例:**
    ```gdscript
    func _on_buy_button_pressed(atom: Atom):
        progress_db.unlock(atom.data.id) # 1. セーブデータに ID を追加するだけ
        graph.sync_with_progress(progress_db) # 2. グラフが自動的に、どの「新しい」アトムが AVAILABLE になったかを判断します
    ```
