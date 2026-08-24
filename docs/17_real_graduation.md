# 章17: 卒業課題とまとめ

← [前章: 実機特有のトラブルと復帰](16_real_troubles.md) | [目次](index.md)

## 卒業課題 ── 実機検証チェックリスト

[docs/SETUP.md の「検証項目」](https://github.com/atinfinity/toio_rmf_bringup/blob/main/docs/SETUP.md)が、そのまま卒業課題になる。
第1部(sim)・第2部(実機)の各章と対応づけて挑むと、学んだことの答え合わせになる:

| 項目 | 学んだ章(sim → 実機) |
|---|---|
| [ ] 2台のフリート登録と位置報告(RVizが実位置と一致) | [章2](02_architecture.md) → [章12](12_real_go_to_place.md) |
| [ ] 1台ずつの `go_to_place`(順次) | [章3](03_go_to_place.md) → [章12](12_real_go_to_place.md) |
| [ ] patrolタスク完走(1台・3周) | [章4](04_patrol.md) → [章12](12_real_go_to_place.md) |
| [ ] 指名なしの投入で、目的地に応じて落札者が入れ替わる | [章5](05_bidding.md) → [章13](13_real_bidding_traffic.md) |
| [ ] 2台続けて投げ、mutex `ring` で直列化される | [章6](06_traffic.md) → [章13](13_real_bidding_traffic.md) |
| [ ] バッテリ離散値(10%刻み)の実測確認 | [章7](07_battery_charge.md) → [章14](14_real_battery_charge.md) |
| [ ] 低バッテリ時のChargeBattery発行・チャージャー帰還(Dock停止) | [章7](07_battery_charge.md) → [章14](14_real_battery_charge.md) |
| [ ] タスクキャンセル → 再投入 | [章7](07_battery_charge.md) → [章14](14_real_battery_charge.md) |
| [ ] delivery と perform_action(LED・効果音) | [章8](08_delivery.md)・[章9](09_fleet_action.md) → [章15](15_real_delivery_action_viz.md) |
| [ ] BLE切断 → 位置報告停止 → 再接続後の復帰 | ── → [章16](16_real_troubles.md) |
| [ ] マット境界付近の挙動(Position ID読取不能領域に入らない) | ── → [章16](16_real_troubles.md) |

最後の2つはシミュレーションに無い実機特有の項目。BLEの切断・再接続や
マット外での位置ロストは、実機フリート運用で必ず向き合うことになる現実。

## まとめ ── このチュートリアルで登ったもの

```
第1部 シミュレーション編 ── 「仕組み」を学ぶ
  章3    1台を1回動かす        ── タスク→Nav2委譲の縦串
  章4    巡回とnavグラフ        ── フリートの地図
  章5    入札                  ── 誰がやるか(タスク割当)     ┐
  章6    交通調停              ── 道をどう分けるか             ├ フリート処理の核心
  章7    バッテリ・充電         ── いつ休むか(自己管理)       ┘
  章8-9  荷役・演出             ── 移動以外のタスク
  章10   可視化                ── 内部状態を見る

第2部 実機編 ── 「手触り」を得る
  章11   sim→real             ── ③だけ差し替え、①②はそのまま
  章12-13 1台・2台             ── 同じコマンド、A4の形が効く
  章14   本物のバッテリ         ── ChargeBattery が本当に発火する
  章15   音が鳴る・運用卓       ── 指令の行き先が本物になる
  章16   壊れ方と直し方         ── 位置が来ない世界
```

**入札・交通調停・充電**の3つが、Open-RMFのフリート処理の核。この3つが
揃うと、運用者は個々のロボットの世話をせずにフリートを回せる。このチュートリアルでは、
その3つを手のひらサイズのtoioで通しで体験した ── シミュレーションで仕組みを、
実機で手触りを。

第2部で一貫して見たのは、ロボット層③を差し替えても①②は一切変わらない
こと。コマンドもログも同じで、変わったのは位置・バッテリ・LED・音の
「出どころと行き先」だけだった。これが Open-RMF がフリートアダプタという
境界を置いている理由そのもの。

## 次に進むなら

- 別のフリート(toio以外のロボット)を同じRMFコアに繋ぐ ── フリート
  アダプタ(EasyFullControl)を自分で書く
- navグラフを自作する ── `toio_rmf_maps` の建物図・navグラフ定義を読む。
  A4で見た mutex group や Dock の設定がどう書かれているかから入るとよい
- door / lift など、このパッケージが省いたRMF機能(rmf_demos参照)
- 実機編の画像・動画を、実機で撮ったものに差し替える
  ([CAPTURE.md](CAPTURE.md) に実機撮影の手順を足すところから)

← [前章: 実機特有のトラブルと復帰](16_real_troubles.md) | [目次](index.md)
