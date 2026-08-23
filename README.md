# toioで学ぶ Open-RMF フリート処理

ROS 2中級者が、Open-RMFの**フリート処理**(複数ロボットへのタスク割当・入札・
交通調停・充電管理)を、手を動かしながら段階的に理解するためのチュートリアル。

> [!NOTE]
> 実行環境(launch・スクリプト・設定)は
> [toio_rmf_bringup](https://github.com/atinfinity/toio_rmf_bringup) にある。
> このリポジトリはそのチュートリアル部分(旧 `docs/tutorial/`)を独立させたもの。

「1台をA地点に動かす」から始めて、「2台が入札で仕事を奪い合い、狭いレーンで
譲り合い、勝手に充電へ帰る」ところまでを1本の動線で登っていく。全ステップを
**toio_gazeboシミュレーション**で完結させ、最後の章だけ実機への移行差分を扱う。

![toio Open-RMF: フリートアクション(左 RViz2 / 右 Gazebo)](docs/images/09_fleet_action.gif)
*本チュートリアルの舞台 ── 左が RViz2(RMFスケジュール可視化)、右が toio_gazebo の
A3マットと2台のキューブ。この上で入札・交通調停・充電・アクション(LED)を学ぶ。
各章に実行中のスクリーンショットと、左RViz/右Gazeboを並べた動画を載せている。*

## 対象読者と前提

- **対象**: ROS 2中級者。ノード / トピック / アクション / launch / TF は理解済み。
  Open-RMFは初見でよい。
- **前提環境**: Ubuntu 24.04 + ROS 2 Jazzy。`~/dev_ws` に
  [toio_rmf_bringup](https://github.com/atinfinity/toio_rmf_bringup) の
  ワークスペースを構築済みであること(未構築なら[章0](docs/00_setup.md)から)。
- **実機は不要**: 章0〜10はシミュレーションのみで完結する。実機(toioキューブと
  Bluetooth)が要るのは[章11](docs/11_real_robot.md)だけ。

## このチュートリアルの読み方

各章は次の型で進む。特に **「観察する」** を飛ばさないこと ── フリート処理は
「動いた」だけでは中身が見えない。RMFの内部状態(入札・予約・タスク状態)を
ログやRVizで覗きながら進めるのが、このチュートリアルの主眼だ。

1. **狙い** ── この章で何ができるようになるか
2. **動かす** ── 投入するコマンド
3. **観察する** ── RMFの内部で何が起きているかをログ / RViz で見る(肝)
4. **理解する** ── そこで働いている概念
5. **確認課題** ── 理解できたか自分で試す小課題

## 全章共通のルール

- **端末A** で環境全体を起動しっぱなしにし、**端末B** でタスクを投入する。
  各端末で最初に環境をsourceする:
  ```bash
  source /opt/ros/jazzy/setup.bash
  source ~/dev_ws/install/setup.bash
  ```
- 端末Aの起動コマンドは全章共通で、シミュレーションを明示的に有効化する:
  ```bash
  ros2 launch toio_rmf_bringup toio_rmf.launch.py mat:=a3 run_sim:=true use_sim_time:=true
  ```
  > このパッケージの既定は**実機運用**(`run_sim:=false` / `use_sim_time:=false`)。
  > シミュレーションでは両方を `true` にする必要がある。
- **タスク投入コマンドには必ず末尾に `--use_sim_time` を付ける**
  (`cancel_task` だけは非対応 ── [章7](docs/07_battery_charge.md)で扱う)。
  この「`--use_sim_time` の有無」以外は、実機でもコマンドはそのまま通る。
  つまり**ここで学んだタスク操作は、章11でそのまま実機に持っていける**。
- 特記なき限り **A3マット**(6頂点・全レーン双方向)を使う。2台同時運用に
  余裕があるため学習に向く。狭い**A4マット**(一方通行ループ)は
  [章6](docs/06_traffic.md)と[章11](docs/11_real_robot.md)で扱う。

## 章立て

| # | 章 | 学ぶこと | 環境 |
|---|---|---|---|
| 0 | [環境構築とスモークテスト](docs/00_setup.md) | ワークスペース構築、初回の完走確認 | sim |
| 1 | [Open-RMFとは(概要と用語)](docs/01_overview.md) | Open-RMFの目的とRMF固有の用語 | ── |
| 2 | [RMFの全体像を掴む](docs/02_architecture.md) | RMFコア / フリートアダプタ / Nav2 の三層 | sim |
| 3 | [1台を動かす(go_to_place)](docs/03_go_to_place.md) | タスク投入 → Nav2委譲、位置報告 | sim |
| 4 | [巡回と帰還(patrol)](docs/04_patrol.md) | navグラフ、周回、finishing_request | sim |
| 5 | [2台と入札(bidding)](docs/05_bidding.md) | 入札・落札・タスク割当 | sim |
| 6 | [交通調停(traffic)](docs/06_traffic.md) | 大域スケジュールと局所回避の二層 | sim |
| 7 | [バッテリと自動充電](docs/07_battery_charge.md) | ChargeBattery、閾値、キャンセル | sim |
| 8 | [搬送とワークセル(delivery)](docs/08_delivery.md) | dispenser / ingestor の分業 | sim |
| 9 | [フリートアクション](docs/09_fleet_action.md) | perform_action、LED・効果音 | sim |
| 10 | [可視化とダッシュボード](docs/10_visualization.md) | RViz、rmf-web、マーカーの読み方 | sim |
| 11 | [実機へ(sim→real)](docs/11_real_robot.md) | 位置報告 / 起動順序 / Dock の差分 | 実機 |

## 関連ドキュメント(このチュートリアルの土台)

- [README](https://github.com/atinfinity/toio_rmf_bringup/blob/main/README.md) ── 起動方法とlaunch引数の一覧
- [docs/SETUP.md](https://github.com/atinfinity/toio_rmf_bringup/blob/main/docs/SETUP.md) ── 環境構築の詳細と実機検証手順
- [docs/TASKS.md](https://github.com/atinfinity/toio_rmf_bringup/blob/main/docs/TASKS.md) ── 各タスクの内部シーケンス図解
- [docs/DASHBOARD.md](https://github.com/atinfinity/toio_rmf_bringup/blob/main/docs/DASHBOARD.md) ── rmf-webダッシュボードの構築

このチュートリアルは「動線と観察」に徹し、コマンドの網羅的な引数一覧や内部
シーケンス図は上記の各ドキュメントへ委ねる。詰まったら該当章から辿ること。

シミュレーション特有の「コマンドは正しいのに引っかかる」症状
(`cancel_task` が戻らない、`/fleet_states` が空、2回目起動で登録されない 等)は
[TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) にまとめてある。
