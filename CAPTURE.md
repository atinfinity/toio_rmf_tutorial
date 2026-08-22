# チュートリアル用メディアの撮影手順

`images/` に置いているスクリーンショット・動画(GIF)を、
toio_gazeboシミュレーションから撮り直す/追加する手順。すべてホストの
X11ディスプレイ上でGazebo・RVizを動かし、画面を取り込む。

## 前提ツール

```bash
sudo apt-get install -y ffmpeg imagemagick scrot xdotool wmctrl fonts-noto-cjk
```

- `ffmpeg` … 画面録画(x11grab)とGIF変換
- `import`(imagemagick)… ウィンドウ単位のスクリーンショット
- `xdotool` / `wmctrl` … ウィンドウの前面化・座標取得
- `fonts-noto-cjk` … 端末風PNG(`05_bidding_log.png`)の日本語描画

## sim起動(録画対象)

```bash
source /opt/ros/jazzy/setup.bash
source ~/dev_ws/install/setup.bash
ros2 launch toio_rmf_bringup toio_rmf.launch.py mat:=a3 run_sim:=true use_sim_time:=true
```

RVizの表示がマット向けに正しく出るには **`rmf_visualization` の small-maps
パッチ**が必要(未適用だと footprint/vicinity の巨大な円柱がnavグラフを覆う)。
手順は [docs/SETUP.md](https://github.com/atinfinity/toio_rmf_bringup/blob/main/docs/SETUP.md) の「rmf_visualization パッチ」を参照。

## ウィンドウ単位のスクリーンショット

```bash
export DISPLAY=:1
# 対象ウィンドウIDを調べる(RViz / Gazebo Sim)
wmctrl -lG | grep -iE 'rviz|gazebo'
# 前面化して取り込む
WID=0xXXXXXXXX
xdotool windowactivate $WID; xdotool windowraise $WID; sleep 0.5
import -window $WID images/00_setup_rviz.png
```

## 画面録画 → GIF

`ffmpeg` の x11grab で対象ウィンドウの矩形を録画する。**幅・高さは偶数**に
すること(libx264 は奇数サイズを拒否する。RViz既定 1853x1025 → `1852x1024`)。

```bash
# 位置とサイズ: xdotool getwindowgeometry <WID> で確認
ffmpeg -y -f x11grab -framerate 15 -video_size 1852x1024 -i :1.0+81,118 \
  -t 16 -pix_fmt yuv420p /tmp/clip.mp4
# mp4 -> GIF(パレット生成で発色を良く。横幅は 926 程度に縮小)
ffmpeg -y -i /tmp/clip.mp4 -vf "fps=10,scale=926:-2:flags=lanczos,palettegen" /tmp/pal.png
ffmpeg -y -i /tmp/clip.mp4 -i /tmp/pal.png \
  -lavfi "fps=10,scale=926:-2:flags=lanczos[x];[x][1:v]paletteuse" images/04_patrol.gif
```

動きを確実に写すには「タスク投入 → 数秒待って走り出してから録画開始」。
タスク完了後に撮ると静止画になる(GIFがほとんど変化せず数KBほどのファイルに
なっていたら、これが原因)。

## 合成GIF(左RViz / 右Gazebo)

`04_patrol.gif` / `06_traffic.gif` は **RViz と Gazebo を左右に並べた合成GIF**。
「スケジュール(RViz)」と「実際のキューブの動き(Gazebo)」を同時に見せるため。
`ffmpeg` が使えない環境向けに、Python だけで撮影・合成する手順も用意した。

```bash
pip install --user mss imageio python-xlib   # 画面グラブ / GIF / ウィンドウ操作
```

1. **Gazebo ウィンドウを前面化**して連番PNGにグラブ(`python-xlib` の
   `_NET_ACTIVE_WINDOW` で raise → `mss` で矩形grab → 3Dビュー部分を crop)。
   ヘッドレス寄りの環境でも、実GPU付きの X(`:1`)なら Gazebo GUI は描画される。
2. タスク投入 → 数秒(交通調停なら約5秒)待って走り出してから **10fps で
   180フレーム(=18秒)** グラブ。`traffic` は片道で終わると後半が静止するので、
   2台を **逆向きに周回**(例: `patrol_B patrol_C` と `patrol_C patrol_B`)させて
   全編クロスし続ける画にする。
3. 既存の RViz GIF を左パネル、Gazebo連番を右パネルに置いて横並び合成し、
   ラベル(`RViz2` / `Gazebo`)を焼き込む(Pillow)。
4. **GIF軽量化のコツ**: 静止部の微小レンダノイズで色indexがブレるとフレーム間
   圧縮が効かず数MBに膨れる。`ImageOps.posterize(5)` でノイズを丸め、共有パレット
   ・**ディザ無効**・**`disposal=1`**(前フレームに差分だけ上書き)で保存すると、
   180フレームでも 1〜1.5MB に収まる。

> RViz 側は既存の(small-mapsパッチ適用済みの)綺麗なGIFを再利用し、Gazebo側
> だけ撮り直して合成している。2つのパネルは別走行のため厳密なフレーム同期は
> していない(どちらも同じ挙動を映す図として並べている)。

## RViz のカメラ(フィット/センタリング)

`rviz/toio_rmf.rviz` の `Views > Current`(TopDownOrtho)で決まる:

| キー | 値 | 意味 |
|---|---|---|
| `Scale` | `2200` | px/m相当。大きいほど拡大。A3マットが画面に収まる値 |
| `X` / `Y` | `0.2` / `-0.15` | 注視点(マット中央) |

A4マットで撮るときは `Scale` を上げ気味に、`X`/`Y` をA4の中央へ合わせる。

## 既知の注意点

- **タスク開始直後にnavグラフ/床面図が一度消えることがある**。navgraph
  visualizer が起動直後に DELETEALL を送り、RVizがそれを受けてクリアするため
  (publisher側のlatchデータは生きている)。**撮影前にRVizを一度リロードして
  おくと**、再購読でnavグラフが復活し、以後はタスク中も残って安定する。
- Gazeboウィンドウはマップを見失わないので、「実行中の様子」を確実に撮るには
  Gazebo側が手堅い。

## 各メディアの撮り方(対応表)

| ファイル | 対象 | 撮り方 |
|---|---|---|
| `00_setup_gazebo.png` | Gazebo | 起動直後、2台がチャージャー上の全景 |
| `00_setup_rviz.png` | RViz | 同上のnavグラフ全景(idle) |
| `03_go_to_place.gif` | Gazebo | `dispatch_go_to_place` で1台を指名し、目的地に着いて停止するまでを録画 |
| `04_patrol_rviz.png` | RViz | patrol投入後、スケジュール経路帯が出た瞬間 |
| `04_patrol.gif` | RViz+Gazebo(合成) | patrol走行を左RViz/右Gazeboで並べた合成GIF(下記「合成GIF」参照) |
| `05_bidding_log.png` | 端末風PNG | `dispatch_patrol` の `-R`有/無 の実出力を並べて描画(`scripts`外の生成物) |
| `06_traffic_rviz.png` | RViz | 2台に別タスクを投入、経路帯が交錯した瞬間 |
| `06_traffic.gif` | RViz+Gazebo(合成) | 2台の交差を左RViz/右Gazeboで並べた合成GIF(下記「合成GIF」参照) |
| `08_delivery.gif` | Gazebo | deliveryを投入し、pickup→dropoffの移動(各地点で約3秒停止)を録画 |
| `10_footprint_vicinity.png` | RViz | `ScheduleMarkers` の `participant location 0/1` を表示に切り替え、稼働中の円が出た状態 |
| `10_dashboard_robots.png` | ブラウザ | rmf-webのRobotsタブ。別途コンテナ起動が要る([docs/DASHBOARD.md](https://github.com/atinfinity/toio_rmf_bringup/blob/main/docs/DASHBOARD.md)) |

### まだ用意していない(必要なら追加)

- `07_battery.*` … シミュレーションでは残量が100%に固定され、ChargeBatteryも
  発火しない([章7](07_battery_charge.md)で実測確認)。撮るには実機が必要

## 図版(SVG)の原本

`images/navgraph_a3.svg` / `navgraph_a4.svg` / `initial_placement_a4.svg` は
撮影物ではなく手描きのSVGで、原本は
[toio_rmf_bringup の docs/images/](https://github.com/atinfinity/toio_rmf_bringup/tree/main/docs/images)
にある(bringup 側の README・SETUP.md・TASKS.md からも参照されるため)。
変更するときは bringup 側を更新し、このリポジトリへコピーして同期する。
