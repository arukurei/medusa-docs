# GraphLines

Medusaにおけるライン（線）は、ノードではなく**リソース**です。これにより、シーンツリーに不要なオブジェクトを増やすことなく、同じラインスタイルを使用して数千の接続を同時に描画できます。

1.  **集中管理ループ:** **Graph (グラフ)** ノードが `_draw()` メソッド内で、登録されたすべての **Atom (アトム)** を走査します。
2.  **パラメータの取得:** 「親 -> 子」のペアごとに、中心座標、**Atom** の半径（線がアイコンの中心ではなく縁から始まるようにするため）、および透明度レベルなどのデータを収集します。
3.  **リソースの呼び出し:** **Graph** は指定された **GraphLine** リソースの `draw()` メソッドを呼び出し、`context` を渡します。

---
## 描画コンテキスト (Drawing Context)
`draw()` メソッドは `context` ディクショナリを受け取り、ラインが接続の状態を「理解」できるようにします：

*   **`connection` (bool):** 物理的な接続かどうか。
*   **`navigation` (bool):** 操作用（ゲームパッドなど）のナビゲーション接続かどうか。
*   **`alpha` (float):** 現在の透明度（**Atom** のアニメーションに依存）。
*   **`highlighted` (bool):** ラインを光らせるべきか。（接続された **Atom** のいずれかにホバーした際に `true` になります）。
*   **`conn_mutual` (bool):** 双方向の接続かどうか。（両端に矢印を描画する際に重要です）。
*   **`is_primary` (bool):** 2つの **Atom** 間の同じラインが二重に描画されるのを防ぐためのフラグ。

---
## 接続モード (Connection Mode)
Medusaのほとんどのラインスタイル（Circuit, Soft, Dotted）には **Connection Mode** パラメータがあります：

1.  **OFF:** ラインを描画しません。
2.  **MERGED (統合):** **Atom** AとBの間に往復の接続がある場合、それらを1本の線として描画します。双方向の場合、自動的に両端に矢印が表示されます。
3.  **SEPARATE (個別):** わずかなオフセット（`separate_stride`）を持たせた2本の並行線を描画します。ノード間の「対立」や「複雑な交換」を表現するのに最適です。

---
## パフォーマンスと最適化

数百の動的なラインをレンダリングするのは負荷がかかる場合があります。そのため、3つのレベルの最適化を導入しています：

1.  **頂点キャッシュ (Vertex Caching):** 矢印や菱形などの形状は毎フレーム計算されません。その形状は一度生成されると `PackedVector2Array` に保存されます。再計算はラインの長さや幅が変更されたときにのみ行われます。
2.  **描画トランスフォーム (Draw Transform):** すべての点をワールド座標で計算する代わりに、`draw_set_transform` を使用します。これにより、GodotエンジンはGPUを効率的に使用してセグメントを描画できます。
3.  **メモリ管理:** 点や色の配列（`PackedColorArray`）は呼び出し間で再利用されます。これにより、低スペックのハードウェアで60 FPS動作させた場合でも、ガベージコレクション (GC) の負荷をほぼゼロに抑えます。

---
## カスタムラインの作成

**GraphLine** リソースを継承するだけで、独自の接続スタイルを作成できます：

```gdscript
@tool
class_name MyPulseLine extends GraphLine

@export var color: Color = Color.CYAN

func draw(canvas: Control, start: Vector2, end: Vector2, context: Dictionary = {}) -> void:
    # 1. 方向と長さを計算
    var direction = (end - start).normalized()
    var time = Time.get_ticks_msec() * 0.005
    
    # 2. 時間に応じてポイントを修正（波のエフェクト）
    var mid_point = start.lerp(end, 0.5) + direction.orthogonal() * sin(time) * 20.0
    
    # 3. Godotの標準的なCanvasItemメソッドで描画
    canvas.draw_line(start, mid_point, color, 2.0)
    canvas.draw_line(mid_point, end, color, 2.0)
```

作成したリソースを **Graph** ノードの `graph_lines` フィールドにドラッグ＆ドロップするだけで、すべてのスキルが脈打つ波のような線で接続されます。
