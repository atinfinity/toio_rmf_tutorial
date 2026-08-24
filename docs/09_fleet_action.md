# 章9: フリートアクション(perform_action)

← [前章: 搬送とワークセル](08_delivery.md) | [目次](index.md) | 次章: [可視化とダッシュボード →](10_visualization.md)

## 狙い

- フリートが独自に宣言するカスタム動作(フリートアクション)を単独で実行する
- 前章の delivery が「ワークセル任せでキューブは演技しない」だったのに対し、
  こちらは**キューブ自身にLEDと効果音で動作を表現させる**方法
- フリート設定(YAML)を編集して、ロボットの振る舞いを変える体験をする

## フリートアクションとは

RMFには、標準タスク(patrol / delivery / charge)では表せないフリート独自の
動作を宣言する仕組みがある。フリートアダプタが「うちのロボットは○○が
できます」と申告し、`perform_action` タスクでそれを名指しで実行する。

toioフリートが宣言しているのは2つ:

- `delivery_pickup` … pickup動作の演出
- `delivery_dropoff` … dropoff動作の演出

キューブには搬送機構が無いので、形だけの実装になっている。その場で3秒
保持し、LED(pickup=緑 / dropoff=青)と効果音で「何をしているか」を見せる
だけだ。delivery(章8)がワークセル側で完結してキューブが無反応なのに対し、
これはキューブ側の演出にあたる。

## 動かす

指定した頂点へ移動し、そこでフリートアクションを実行する:

```bash
ros2 run rmf_demos_tasks dispatch_action -s patrol_A -a delivery_pickup --use_sim_time
```

| 引数 | 意味 |
|---|---|
| `-s` | アクションを実行する waypoint 名 |
| `-a` | アクション名(`delivery_pickup` / `delivery_dropoff`) |

dropoff側も試す:

```bash
ros2 run rmf_demos_tasks dispatch_action -s patrol_D -a delivery_dropoff --use_sim_time
```

## 観察する

![フリートアクションのアニメーション(左: RViz2 / 右: Gazebo)](images/09_fleet_action.gif)
*指定頂点へ移動して `perform_action` を実行する様子。右の Gazebo で、pickup 時に
キューブの LED が緑、dropoff 時に青へ色づくのが見える(その間 3 秒保持)。
左の RViz2 では各ロボットが対象 waypoint 上で停止する。*

- ロボットが指定頂点へ移動し、そこで3秒保持する
- LEDが色づく(pickup=緑 / dropoff=青)。toio_gazebo の `ToioLedSystem` が
  フリートアダプタの LED 指令を反映するため、上のGIFのとおりシミュレーションでも
  色は見える
- 効果音は **sim では鳴らない**。`toio_sound` ノードが効果音ID付きの指令を出し、
  ログに `playing sound effect ...` と出るところまでは動くが、音そのものは実機でしか
  鳴らない ── この差が第2部で実機に移る動機のひとつ([章15](15_real_delivery_action_viz.md))
- `rmf_task_dispatcher` のログで、`perform_action` タスクがアクション名付きで
  実行される様子が読める

## カスタマイズする(YAML編集)

保持時間・色・効果音は、フリート設定
`toio_fleet_adapter/config/toio_fleet_config_<mat>.yaml` の `toio.actions`
セクションで変えられる。編集後は端末Aを起動し直すと反映される。`delivery_pickup`
の保持時間を長くする、LEDの色を変える、などを1つ試して、`dispatch_action` で
挙動が変わることを確認する。章7の `recharge_threshold` と同じく、
**フリートの振る舞いは設定ファイルで決まっている**という感覚がここでも強まる。

> どのキー(保持秒・色・効果音)がどれに対応するかは、実ファイルの
> `toio.actions` を開いて確かめること。このチュートリアルはコマンド操作を
> 主眼にしているため、YAMLの全キーはファイル側に委ねる。

## 理解する

delivery(章8)と perform_action(本章)は「荷役の見せ方」が逆で、前者は荷役が
ワークセルの仕事(キューブは移動して待つだけ)、後者は荷役の演出がキューブの仕事。
実運用では本物の荷役(ワークセル)とロボットの演出を組み合わせることになるが、
学習としてはこの2つを分けて理解しておくと混乱しない。フリートアクションは
**標準タスクで足りない動作をフリート側で定義してRMFに載せる拡張ポイント**で、
toioでは「LEDと音」という無害な例だが、実機ロボットなら「アームで掴む」「扉を開ける」
などをここに実装する。章7と本章で設定ファイルを2回いじったとおり、フリートの人格
(いつ充電するか、どう荷役を演出するか)はコードでなくYAMLに書かれている。

## 確認課題

1. `delivery_pickup` と `delivery_dropoff` を両方投げ、頂点での3秒保持を
   観察する(実機なら色と音の違いも)。
2. `toio_fleet_config_<mat>.yaml` の `toio.actions` を1箇所編集(例:保持
   時間を延ばす)し、端末Aを再起動して反映されることを確認する。
3. delivery(章8)と perform_action(本章)で、「荷役をするのは誰か」の
   違いを一文で説明する。

← [前章: 搬送とワークセル](08_delivery.md) | [目次](index.md) | 次章: [可視化とダッシュボード →](10_visualization.md)
