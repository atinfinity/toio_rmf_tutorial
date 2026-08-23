# トラブルシューティング(シミュレーション)

このチュートリアルをシミュレーションで進める際に遭遇しやすい、
**コマンドやコードの誤りではない**症状と対処をまとめる。ダッシュボード
(rmf-web)まわりは [docs/DASHBOARD.md](https://github.com/atinfinity/toio_rmf_bringup/blob/main/docs/DASHBOARD.md)、
起動引数の詳細は [README](https://github.com/atinfinity/toio_rmf_bringup/blob/main/README.md) を参照。

---

## `cancel_task` がプロンプトに戻らない

**症状**: `cancel_task -id <task_id>` を実行すると、そのまま制御が戻らず
固まったように見える。

**原因**: 不具合ではなく `rmf_demos_tasks` の `cancel_task` の実装仕様。
キャンセル要求(`ApiRequest`)を publish したあと、応答待ちのため spin に
入ったまま自力で終了しない。

**対処**: **キャンセル要求はすでに送信済み**なので、`Ctrl-C` で抜けてよい。
抜けたあとに対象ロボットが巡回をやめてチャージャーへ戻れば成功。
`rmf_task_dispatcher` / `fleet_adapter` のログに
`Canceling go_to_place ...` → `Goal canceled` が出ることでも確認できる。
なお `cancel_task` はこのチュートリアルで唯一 `--use_sim_time` を付けない
コマンドである(章7参照)。

---

## `/fleet_states` を `echo` しても何も出ない

**症状**: `ros2 topic echo /fleet_states --once` が
`does not appear to be published yet` のまま、あるいは何も表示せず終わる。

**原因**: `--once` は購読直後の1メッセージだけを待つため、DDSの
ディスカバリ完了前だと取りこぼしやすい。

**対処**: `--once` を外して echo し続ける:

```bash
ros2 topic echo /fleet_states
```

数秒待てば各ロボットの `name` / `battery_percent` / 位置が流れてくる。
必要な値を1回見たいだけなら、表示されたところで `Ctrl-C` する。

---

## 2回目以降の起動でロボットが登録されない / `/fleet_states` が空

同一マシンでシミュレーションを**起動し直した2回目以降**に起きやすい、
一見「マット(例: A4)固有のバグ」に見える症状。実際は環境の後片付け不足が
原因で、コードやマットの問題ではない。

**症状(いずれか)**:

- `fleet_adapter` のログに
  `Fleet [toio] does not have any robots to accept task` /
  入札で `did not receive any bids` が出て、タスクが実行されない
- `/fleet_states` にロボットが1台も出てこない
- 別端末の `tf2_echo` から `map` などのフレームが見えない
  (`frame does not exist`)。一方で Nav2 自体は `Managed nodes are active`
  まで進んでいる

**原因**: 前回起動の**ゾンビプロセスの残存**と、
**FastDDS の共有メモリ枯渇**。特に `gz sim` を強制終了(`kill -9`)すると、
`static_transform_publisher` などの子プロセスが親から切り離されて生き残り、
グローバルな `/tf` に**古い世代の衝突する変換**を流し続ける。さらに
`/dev/shm` に `fastrtps_*` セグメントが大量に溜まると、新しい参加者の
ホスト内ディスカバリが成立しなくなる。

**対処**: 次の起動前に、プロセスと共有メモリを完全に掃除して、
別の `ROS_DOMAIN_ID` で起動し直す:

```bash
# 1) ROS/gz 関連プロセスを PID 指定で確実に停止する
pids=$(ps -eo pid,args | grep -iE '/opt/ros/jazzy/lib|gz sim|toio_rmf|mock_workcells|rmf_|fleet_adapter|ros_gz|static_transform' \
       | grep -v grep | awk '{print $1}')
[ -n "$pids" ] && kill -9 $pids

# 2) FastDDS の共有メモリを掃除する
rm -f /dev/shm/*fastrtps* /dev/shm/*fastdds* /dev/shm/sem.*

# 3) 掃除できたか確認(0 と、少数であること)
ps -eo pid,args | grep -iE '/opt/ros/jazzy/lib|gz sim|static_transform' | grep -v grep
ls /dev/shm | wc -l

# 4) 別ドメインで起動し直す
export ROS_DOMAIN_ID=80
ros2 launch toio_rmf_bringup toio_rmf.launch.py mat:=a3 run_sim:=true use_sim_time:=true
```

掃除が済んでいれば、`fleet_adapter` のログに
`Successfully added robot [toio1] to the fleet [toio]`
(と `toio2`)が出て、`/fleet_states` に2台とも現れる。

> [!NOTE]
> `pkill -f "a|b|c"` の**選言(`|`)は効かない**。`pkill -f` のパターンは
> 基本正規表現として1つの文字列に照合されるため、`|` は区切りとして
> 解釈されず何にもマッチしない。上のように `ps | grep -E | awk` で
> PID を集めて `kill` するのが確実。

---

## 起動直後の最初のタスク要求が消える(実機)

**症状**: 実機で `Managed nodes are active` が出た直後にタスクを投げると、CLI は
成功したように見えるのに**ロボットが動かない**。2回目以降は普通に通る。

**原因**: 活性化直後は購読側の接続確立が間に合わず、**最初のCLI要求が届かない
ことがある**([toio_rmf_bringup#55](https://github.com/atinfinity/toio_rmf_bringup/issues/55))。

**対処**:

- `Managed nodes are active` から**約25秒待って**から最初のタスクを投げる。
- 投げたら、フリートアダプタのログに `Direct request … queued`(指名時)か、
  ディスパッチャの `Add Task`(入札時)が出たことを確認する。出ていなければ
  もう一度投げる(2回目は通る)。

sim では起きにくいが、実機の毎回の立ち上げで意識しておくとよい([章11](11_real_robot.md))。

## バッテリが減らない / ChargeBattery が発火しない

これは不具合ではなくシミュレーションの制約。実機の `battery_state` が
sim には無いため `battery_percent` は 100.0% のまま動かず、自動充電
(ChargeBattery)は sim では発火しない。詳細と裏取りの手順は
[章7](07_battery_charge.md)に記載。
