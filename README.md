# roadviewer-data

**日本道路名マップ**(iOS)が同梱する道路データベース
(roads.db / intersections.db)の公開リポジトリです。

このデータベースは [OpenStreetMap](https://www.openstreetmap.org/) のデータを
加工して作成した派生データベースであり、
[Open Database License (ODbL) 1.0](https://opendatacommons.org/licenses/odbl/1-0/)
に基づき公開しています。

> © OpenStreetMap contributors
>
> Contains information from OpenStreetMap, which is made available
> under the Open Database License (ODbL).

- アプリ: 日本道路名マップ(App Store リンクは公開後にここへ追加)<!-- TODO: アプリのリンク -->
- ライセンス全文: [LICENSE](LICENSE)
- スキーマ詳細: [SCHEMA.md](SCHEMA.md)

## データのダウンロード

DB ファイル本体はサイズの都合上 git 管理せず、
**[Releases](../../releases) からダウンロード**してください。

| ファイル | サイズ | 内容 |
|---|---|---|
| roads.db | 約 280 MB | 全国の道路 111,490 本 / セグメント 718,658 本 |
| intersections.db | 約 27 MB | 全国の交差点 132,453 箇所 |

roads.db は GitHub の 1 ファイル 100 MB 制限を超えるため、リポジトリには
含めていません(Git LFS は無料枠の帯域 1 GB/月 を数回のダウンロードで
使い切るため不採用。Releases は 1 ファイル 2 GB まで無料・帯域制限なし)。

## データの内容

OpenStreetMap の道路データ(日本全国)を、本アプリ向けに
「**1 本の道路**」単位へ名寄せ・統合した SQLite データベースです。

### roads.db

| テーブル | 行数 | 内容 |
|---|---|---|
| roads | 111,490 | 道路(名寄せ・統合後の論理単位)。名称・ふりがな・priority・道路クラス等 |
| road_segments | 718,658 | 描画用セグメント(OSM way 単位のジオメトリ) |
| road_aliases | 16,949 | 別名(検索用) |
| road_refs | 19,663 | 路線番号(国道20号・E4 等) |
| road_admins | 129,734 | 道路が通過する都道府県・市区町村・行政区(通過順) |
| road_tags | 50,955 | 分類タグ |
| road_topo_links | 7,454 | 無名 link way の本線帰属(交差点ビルド用中間データ) |
| road_overview | 944 | 広域ズーム用の簡略化ジオメトリ(2 段階) |
| roads_rtree / segments_rtree | — | R*Tree 空間インデックス |

### intersections.db

| テーブル | 行数 | 内容 |
|---|---|---|
| intersections | 132,453 | 交差点(名称・座標・信号有無・kind: 交差点/JCT/ランプ) |
| road_intersections | 244,285 | 道路⇔交差点の対応 |

各テーブル・各列の定義は [SCHEMA.md](SCHEMA.md) を参照してください。

## 加工内容の概要

OSM 生データ(日本抽出)からの独自パイプラインによる主な加工:

1. **道路の名寄せ・統合** — `route=road` リレーションによるグループ化を第一とし、
   リレーションに属さない way は名前の正規化(NFKC・括弧注記除去・
   「○○県道N号」プレフィックス除去等)+ 端点近接(200 m)でマージ。
   さらに同名+同 ref、同名+bbox 重複、共有 way の多い道路グループを後処理で統合
2. **priority 付与** — 道路クラス・総延長・接続性・知名度シグナル
   (国道/高速リレーション所属等)から表示優先度(0–100)をデータ駆動で算出
3. **行政界判定** — OSM の `boundary=administrative` リレーションから
   都道府県・市区町村・政令市行政区ポリゴンを構築し、各道路の通過行政区域を
   通過順に記録(road_admins)
4. **簡略化ジオメトリ生成** — 広域ズーム描画用に Douglas-Peucker 法で
   2 段階(fine / coarse)の簡略化ポリラインを生成(road_overview)
5. **属性整備** — ふりがな(name:ja-Hira)、路線番号ラベル、wikipedia / wikidata、
   速度制限レンジ、道路種別(feature_type)、高速の本線/ランプ/IC/SA 区別
   (highway_part)等の付与
6. **クレンジング** — 計画段階(planned/proposed)の架空ルート、バス専用路、
   サーキット、SA/PA 場内通路等の非道路の除外。退化名(「線」等)への
   正規化事故ガード、誤統合の検出
7. **交差点抽出** — way のノード共有に基づくトポロジーベース抽出
   (立体交差は自動排除)。中央分離帯等で分裂したノードのクラスタリング、
   信号ノードからの名称取得、JCT/ランプ判別

## データの限界

- **元データは OpenStreetMap** です。OSM 上の入力漏れ・誤り(道路名の欠落、
  名称の表記ゆれ、行政界の誤差等)はそのまま反映されます
- 名寄せ・統合はヒューリスティックであり、本来別の道路が 1 本に統合される、
  または同一道路が分裂して収録される場合があります
- `name` タグのない道路は原則収録されません(高速のランプ等、一部例外を除く)
- 交差点名は OSM の信号・交差点ノードの name タグに依存し、網羅的ではありません
- priority・is_major 等のスコアは本アプリの表示用に設計した独自指標であり、
  公的な道路重要度を示すものではありません
- ナビゲーション・測量等、正確性が要求される用途には適しません

## 更新方針

- OSM データの更新に追従した定期的な再ビルドは予定していますが、
  頻度は保証しません。各リリースに元データの取得時期を記載します
- スキーマは後方互換(列の追加のみ)を基本としますが、
  破壊的変更がある場合はリリースノートに明記します
- 誤りの報告は Issue へお願いします。ただしデータ自体の誤りは
  OpenStreetMap 本体の編集で直すことが最善です(修正は次回ビルドに反映されます)

## ライセンス

このデータベースは [Open Database License (ODbL) 1.0](LICENSE) で提供します。

- 出典: © OpenStreetMap contributors
  (https://www.openstreetmap.org/copyright)
- 本データベースを利用・再配布する場合は ODbL の条件
  (出典表示・同ライセンスでの共有)に従ってください
