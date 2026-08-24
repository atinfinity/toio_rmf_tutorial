# 章12: 実機で1台を動かす(go_to_place / patrol)

← [前章: 実機へ](11_real_robot.md) | [目次](index.md) | 次章: [実機で入札と交通調停 →](13_real_bidding_traffic.md)

対応する第1部: [章3 1台を動かす](03_go_to_place.md) / [章4巡回と帰還](04_patrol.md)

## 狙い

- simで最初にやった「1台を1回動かす」を実機で繰り返し、**コマンドもログの読み方もsimと変わらない**ことを体で確かめる
- RMFが見ている位置が `/toioN/toio/pose` 由来に変わったことを確認する
- A4の一方通行ループを実機で一周させ、navグラフの形を手触りで覚える

## simとの違い

| | sim(章3・4) | 実機(この章) |
|---|---|---|
| コマンド | `--use_sim_time` 付き | 外す |
| 頂点 | A3(`patrol_A`〜`patrol_D`) | A4(`patrol_A` / `patrol_B` のみ) |
| 位置の出どころ | TF `map → toio1/base_link` | `/toio1/toio/pose`(マットに印刷された位置コード = Position ID) |
| 経路の選び方 | 等長経路が2本あり毎回変わりうる | ループが1本なので常に同じ向きに回る |

## 動かす

[章11](11_real_robot.md)の手順で端末1・2を起動し、25秒待ってから端末3で:

```bash
# 指名して1台だけ動かす(章3の「特定の1台を指名する」)
ros2 run rmf_demos_tasks dispatch_go_to_place -p patrol_B -F toio -R toio1
```

`--use_sim_time` が無いだけで、章3と同じコマンド。投げたら端末2のログに `Direct request … queued` が出たことを確認する(出なければもう一度)。

続けて巡回。A3の `patrol_A patrol_D` は、A4では `patrol_A patrol_B`:

```bash
ros2 run rmf_demos_tasks dispatch_patrol -p patrol_A patrol_B -n 3
```

![go_to_placeで1台が目的地へ移動(sim の画像を流用)](images/03_go_to_place.gif)
*第1部のsim動画を流用。左のRViz2の見え方は実機でも同じ。右のGazeboは実機には無く、代わりに目の前のマットでキューブが動く。*

## 観察する

### 1. 位置が `/toio1/toio/pose` から来ていることを確かめる

章3では `tf2_echo map toio1/base_link` で位置を見た。実機ではアダプタがTFでなくtoio_ros2のposeトピックを購読している:

```bash
ros2 topic echo /toio1/toio/pose
```

走行中に値が更新され、`/fleet_states`([章2](02_architecture.md))のロボット位置と一致する。**キューブを手で持ち上げてマットから離すとposeの更新が止まる** ── マットのPosition IDを読んで位置を出しているため。これが[章16](16_real_troubles.md)で扱う「マット境界で位置を失う」現象の正体。

### 2. 一方通行ループを目で追う

`patrol_A patrol_B` の3周は、A3の往復(章4)と違ってループを同じ向きに回り続ける。`patrol_B → patrol_A` の復路も、来た道を戻らず `approach_1` を経由して回り込む。RVizの緑の帯がループをなぞるのを見ながら、[章11](11_real_robot.md)のnavグラフ図と照らす。

### 3. 完了後の帰還と、チャージャーへの最終区間

3周終わると `finishing_request` でチャージャーへ帰る(章4と同じ)。ただし最後の数cmはNav2でなくキューブ内蔵の走行で停まる。チャージャー頂点に設定されたDockイベントによるもので、実機だけの機能。詳しくは[章14](14_real_battery_charge.md)で扱うので、ここでは「最後にピタッと止まる」のを見ておく。

### 4. タスクの一生をログで追う

端末2の `rmf_task_dispatcher` と `toio_fleet_adapter` のログに、章3で読んだ「BidNotice → BidResponse → 落札 → NavigateToPose → completed」が同じ順で流れる。ログの読み方はsimと1行も変わらない。

## 理解する

差し替わったのは③だけで、コマンドもログもsimと同じ。違うのは位置の出どころと、走っている実体だけだ。ただし**位置の出どころが変わると、失い方も変わる**。TFはGazeboが出し続けるが、実機のposeはマット上にいる間しか出ない。フリート運用で「位置ロスト」が現実の問題になる入口がここで、章16で改めて扱う。

## 確認課題

1. `-R toio2` に変えて `go_to_place patrol_A` を投げ、toio2だけが動くことを確認する。`/toio2/toio/pose` も同時にechoしておく。
2. patrol中にキューブを一瞬持ち上げて戻す。`/toio1/toio/pose` が止まって再開するのを見る。RMFの `/fleet_states` の位置はその間どうなるか。

← [前章: 実機へ](11_real_robot.md) | [目次](index.md) | 次章: [実機で入札と交通調停 →](13_real_bidding_traffic.md)
