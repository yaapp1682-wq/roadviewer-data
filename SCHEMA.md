# スキーマリファレンス

roads.db / intersections.db (SQLite 3) の全テーブル・全列の定義。
実DBの DDL から抽出したものです。

## 共通事項

- 座標はすべて WGS84 (緯度 `lat` / 経度 `lon`、度単位)
- 文字列名は NFKC 正規化済み(全角英数→半角等)
- `geometry_blob` はリトルエンディアン Float32 の `(lat, lon)` ペア列
  (1頂点 = 8バイト)。`road_overview.geometry_blob` のみ、複数折れ線を
  `(NaN, NaN)` ペアで区切る

---

## roads.db

### roads — 道路(名寄せ・統合後の論理単位)

| 列 | 型 | 説明 |
|---|---|---|
| road_id | INTEGER PK | 道路ID |
| canonical_road_id | INTEGER | 正規道路ID(統合時の参照用) |
| display_name | TEXT | 表示名(例: 山手通り) |
| normalized_name | TEXT | 正規化名(検索キー。NFKC・括弧注記除去・県道プレフィックス除去等) |
| road_type | TEXT | 道路種別(road_class と同値の旧列) |
| priority | INTEGER | 表示優先度(0–100。道路クラス・延長・接続性・知名度シグナルから算出) |
| user_priority | INTEGER | ユーザー側上書き用 priority |
| is_major | INTEGER | 主要道フラグ(0/1。国道・高速 relation 所属や知名度シグナルによるデータ駆動判定) |
| is_highway | INTEGER | 高速道路フラグ(0/1) |
| representative_lanes | INTEGER | 代表車線数 |
| importance_connection_score | INTEGER | 接続重要度スコア |
| intersection_complexity | INTEGER | 交差点複雑度 |
| bbox_min_lat / bbox_max_lat / bbox_min_lon / bbox_max_lon | REAL | 道路全体のバウンディングボックス |
| center_lat / center_lon | REAL | 中心座標 |
| total_length_meters | REAL | 総延長(メートル) |
| ref | TEXT | 路線番号(例: 20, E4) |
| wikipedia | TEXT | OSM wikipedia タグ(例: ja:山手通り) |
| road_class | TEXT | `general` / `motorway` / `trunk` |
| relation_id | INTEGER | 由来の OSM relation ID(NULL = 名前マージで生成) |
| yomi | TEXT | ふりがな(例: やまてどおり。relation の name:ja-Hira 由来) |
| maxspeed_min / maxspeed_max | INTEGER | 最低/最高速度制限 (km/h) |
| wikidata | TEXT | 道路固有の Wikidata QID(例: Q11579612)。wikipedia が空の場合の sitelink 解決用 |
| feature_type | TEXT | 道路種別。NULL = 通常道路。構造系: `bridge` / `tunnel` / `slope` / `footbridge` / `ped_bridge` / `overpass` / `underpass` / `steps` / `level_crossing` / `roundabout`。非自動車系: `cycleway` / `greenway` / `promenade` / `pedestrian`。別枠系(デフォルト非表示推奨): `private_road`(私道・車道系) / `passage`(自由通路・連絡通路等・歩行者系) |
| highway_part | TEXT | 高速(is_highway=1)の本線/付帯区別。NULL = 本線。`ramp`(出入口・ランプ・連結路) / `ic_jct`(IC・JCT) / `pa_sa`(PA・SA) / `tollgate`(料金所)。一般道は常に NULL |

インデックス: `idx_roads_name(normalized_name)`, `idx_roads_priority(priority)`,
`idx_roads_class(road_class)`, `idx_roads_rel(relation_id)`

### road_segments — セグメント(OSM way 単位のジオメトリ)

| 列 | 型 | 説明 |
|---|---|---|
| segment_id | INTEGER PK AUTOINCREMENT | セグメントID |
| road_id | INTEGER | 帰属する roads.road_id |
| osm_way_id | INTEGER | 元の OSM way ID |
| name | TEXT | way の name タグ |
| highway | TEXT | OSM highway タグ(motorway / trunk / primary / secondary / tertiary 等) |
| ref | TEXT | way の ref タグ |
| network | TEXT | way の network タグ |
| oneway | INTEGER | 一方通行(0/1) |
| bridge | INTEGER | 橋(0/1) |
| tunnel | INTEGER | トンネル(0/1) |
| layer_value | INTEGER | OSM layer タグ(立体交差の階層) |
| geometry_blob | BLOB | Float32 (lat, lon) ペア列 |
| node_count | INTEGER | 頂点数 |
| bbox_min_lat / bbox_max_lat / bbox_min_lon / bbox_max_lon | REAL | セグメントのバウンディングボックス |
| maxspeed | TEXT | 速度制限(OSM 生値) |
| length_meters | REAL | セグメント長(メートル) |
| is_ramp | INTEGER | 無名ランプ(motorway_link / trunk_link)フラグ。描画専用、検索・priority 対象外 |

インデックス: `idx_segs_road(road_id)`, `idx_segs_osm(osm_way_id)`

### road_aliases — 別名(検索用)

| 列 | 型 | 説明 |
|---|---|---|
| road_id | INTEGER | roads.road_id |
| alias | TEXT | 別名(alt_name / official_name / loc_name / old_name、通称等) |

### road_refs — 路線番号

| 列 | 型 | 説明 |
|---|---|---|
| road_id | INTEGER | roads.road_id |
| ref | TEXT | 番号(例: "20", "317", "E4") |
| ref_label | TEXT | 表示ラベル(例: "国道20号", "東京都道317号") |
| ref_type | TEXT | `national` / `prefectural` / `expressway` / `unknown` |

### road_admins — 通過行政区域

| 列 | 型 | 説明 |
|---|---|---|
| road_id | INTEGER | roads.road_id |
| pref | TEXT | 都道府県名(例: 東京都。NFKC 正規化済み) |
| city | TEXT | 市区町村名(例: 新宿区 / 八王子市 / 横浜市。admin_level=7) |
| ward | TEXT | 政令指定都市の行政区名(例: 港北区。admin_level=8)。政令市以外は NULL |
| seq | INTEGER | 通過順(0始まり。segment 順で最初に通過した順。名前マージ道路は近似順) |

主キー: `(road_id, seq)`。インデックス: `idx_admins_city(city)`

### road_tags — タグ

| 列 | 型 | 説明 |
|---|---|---|
| road_id | INTEGER | roads.road_id |
| tag | TEXT | 分類タグ |

### road_topo_links — 平面系無名 link の帰属情報(中間データ)

交差点ビルド(build_intersections_db.py)専用。アプリは参照しない(描画されない)。

| 列 | 型 | 説明 |
|---|---|---|
| osm_way_id | INTEGER | link way の OSM way ID(primary/secondary/tertiary_link) |
| road_id | INTEGER | 帰属先本線の roads.road_id |
| highway | TEXT | 元の highway 種別 |

### road_overview — 広域ズーム用の簡略化ジオメトリ

| 列 | 型 | 説明 |
|---|---|---|
| road_id | INTEGER | roads.road_id |
| level | TEXT | `fine`(表示スパン 0.5〜2°向け、Douglas-Peucker 許容 0.001度) / `coarse`(スパン 2°超向け、同 0.0025度) |
| geometry_blob | BLOB | Float32 (lat, lon) ペア列。複数折れ線は NaN ペア区切り |
| point_count | INTEGER | 実頂点数合計(NaN 区切りを含まない) |

主キー: `(road_id, level)`

### roads_rtree / segments_rtree — 空間インデックス

SQLite R*Tree 仮想テーブル。列: `road_id`(または `segment_id`),
`min_lat`, `max_lat`, `min_lon`, `max_lon`。
付随する `*_node` / `*_parent` / `*_rowid` テーブルは R*Tree の内部実装です。

---

## intersections.db

### intersections — 交差点

| 列 | 型 | 説明 |
|---|---|---|
| intersection_id | INTEGER PK AUTOINCREMENT | 交差点ID |
| normalized_name | TEXT | 正規化名(「交差点」サフィックス除去、JCT/IC 表記ゆれ統一等) |
| display_name | TEXT NOT NULL | 表示名 |
| lat / lon | REAL NOT NULL | 座標 |
| road_class | TEXT NOT NULL | 接続道路から決まるクラス |
| has_signal | INTEGER | 信号有無(0/1) |
| is_major | INTEGER | 主要交差点フラグ(信号有無 + 接続道路数 + 接続道路の priority/is_major から判定) |
| road_count | INTEGER | 接続道路数 |
| kind | TEXT | `crossing`(平面交差点) / `jct`(JCT・IC) / `ramp`(出入口・料金所) |
| osm_node_ids | TEXT | 構成 OSM ノード ID の JSON 配列 |

インデックス: `idx_intersections_name(normalized_name)`, `idx_intersections_class(road_class)`

### road_intersections — 道路⇔交差点の対応

| 列 | 型 | 説明 |
|---|---|---|
| road_id | INTEGER NOT NULL | roads.db の roads.road_id |
| intersection_id | INTEGER NOT NULL | intersections.intersection_id |

制約: `UNIQUE(road_id, intersection_id)`。
インデックス: `idx_road_intersections_road`, `idx_road_intersections_intersection`
