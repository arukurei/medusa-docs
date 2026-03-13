# 物理 (Physics)

Medusaの物理エンジンは、標準的な `RigidBody` エンジンではありません。これは、リアルタイムで数千のUI要素を処理するために最適化された**グラフシミュレーション専用システム**です。

このシステムは**データ指向アプローチ (Data-Oriented approach)** に基づいて構築されています。位置、速度、接続に関するすべてのデータがフラットな配列にパックされているため、FPSを低下させることなく、数百のノードを反発させるなどの複雑な計算が可能です。

---
## 1. シミュレーションアーキテクチャ

オブジェクトが「個別に」存在する標準的な物理とは異なり、Medusaでは **Graph (グラフ)** ノードを介して計算が集中管理されます。

フレームごとに、**Graph** はすべてのフォースに渡される `context` オブジェクトを生成します。これには以下の内容が含まれます：

*   `spatial_grid`: **空間グリッド**（近傍ノードの高速検索用）。
*   `centers` と `scales`: すべての **Atom (アトム)** のワールド座標とスケール。
*   `radii`: 有効な衝突半径（接続による **Atom** の成長を考慮）。
*   `delta`: フレーム時間。

---
## 2. 計算フェーズ (GraphForce)

各フォース（`GraphForce` を継承したもの）は、2つの処理段階を経ます：

#### フェーズ 1: 質量計算 (`calculate_mass`)
この段階で、フォースは各 **Atom** がどの程度の「重さ」を持つべきかを決定します。

!!! example
    **InertiaForce** は接続ごとに質量を追加します。これにより、多数の接続を持つ「ハブ」ノードはより安定して動きが遅くなり、単一の **Atom** は軽く俊敏な動きを保ちます。

#### フェーズ 2: フォースの適用 (`apply_force`)
ここでは、変位ベクトルの数学的計算が行われます。フォースは位置を直接変更するのではなく、速度アキュムレータに値を追加します。

```gdscript
# カスタムフォース内での記述
func apply_force(registry, forces, context):
    var dist = center_a.distance_to(center_b)
    # ベクトルを計算し、特定の Atom インデックスに追加します
    forces[index_a] += direction * power
```

---
## 3. 標準フォースライブラリ

生き生きとした整然としたスキルツリーを作成するために、Medusaは6つの基本フォースを提供しています：

1.  **GravityForce (重力):**
    **Atom** をそれぞれの「引力ターゲット」に引き寄せます。自由な **Atom** は **Graph** の中心に、**GraphBranch (グラフブランチ)** 内の **Atom** はそのブランチの中心に引き寄せられます。
2.  **RepulsionForce (反発力):**
    磁石の同極のように機能し、**Atom** 同士の重なりを防ぎます。
3.  **SpringForce (バネ接続):**
    接続された **Atom** 間に弾力性のある張力を作成します。これらを特定の `target_length`（目標距離）に保とうとします。
4.  **InertiaForce (慣性):**
    動きへの抵抗を制御します。急激な変化による **Graph** の「揺れ」を防ぎ、動きをよりスムーズで物理的にします。
5.  **ClusterRepulsionForce (クラスター反発):**
    **GraphBranch** 全体を単一の物理体として扱います。ブランチ全体を押し出すことで、巨大なスキルクラスター同士が重ならないようにします。
6.  **VortexForce (渦):**
    自由な **Atom** を中心の周りで軌道運動させます。ダイナミックに「回転する」星図のようなスキルレイアウトを作成できます。

!!! note
    RepulsionForceは**二乗反比例の法則**に基づいています。**Atom**が近づくほど、反発力は強くなります。

---
## 4. 最適化システム
物理シミュレーションはリソースを大量に消費します。Medusaは、CPUの負荷を抑えるためにいくつかの技術を採用しています。

#### 空間グリッド (Spatial Grid)
**Graph** は画面を見えないセル（サイズは `spatial_grid_size` で設定）に分割します。すべての **Atom** が**すべての**他の **Atom** と衝突判定を行う（O(n²) の複雑さ）代わりに、自身のセルと隣接するセルの近傍のみをチェックします。これにより、パフォーマンスを維持したまま巨大な **Graph** を構築できます。

#### 熱システム (Thermal System: Sleep & Heat)
Medusaはノードの「スリープ（休止）」システムを実装しています：

1.  **冷却 (Cooling):** **Atom** の速度が `sleep_threshold`（スリープしきい値）を下回ると、「冷却温度」が蓄積されます。数秒後（`sleep_delay`）、**Atom** は「スリープ」状態になり、物理計算が停止します。
2.  **加熱 (Heat):** **Atom** にマウスをホバーしたり、ドラッグしたり、あるいは動いている隣接ノードが衝突したりすると、ノードは即座に「加熱」され、目を覚まします。
3.  **エネルギー伝達:** 目覚めた **Atom** は、接続を通じて「熱（エネルギー）」を隣接ノードに伝え、ブランチ全体を動かすことができます。

!!! note
    多くのエンジンでは、シーンのスケール（Zoom）を変更すると物理が「爆発」することがあります。Medusaの物理は、親の現在のスケールにフォースを自動的に適応させます。フォースの係数が再計算されるため、ズームレベルに関わらずバネの張力や重力が同じように感じられます。

---
## 5. カスタムフォースの作成

`GraphForce` を継承した新しいリソースを作成し、2つのメソッドを実装します。
以下は、中心から遠ざける「遠心力」の例です：

```gdscript
class_name OutwardForce extends GraphForce

@export var power: float = 500.0

func apply_force(registry: Array, forces: Array, context: Dictionary) -> void:
    var center = context["global_center"]
    var positions = context["centers"]
    
    for item in registry:
        var idx = item.index
        var direction = (positions[idx] - center).normalized()
        forces[idx] += direction * power
```

その後、このリソースを **Graph** ノードの `Physics Forces` 配列に追加するだけで適用されます。

!!! tip
    アトムが近すぎる場合は、 `graph.wake_up_atom(atom)` を呼び出します。反発力が自動的にノードを理想的な距離に配置します。
