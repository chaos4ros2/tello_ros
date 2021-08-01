# `tello_ros`

`tello_ros`はTelloとTello EDUドローン用のROS2ドライバーである。

## パッケージ

There are 4 ROS packages:
* `tello_driver`はドローンにつながるためのC++ ROSノード
* `tello_msgs`はROSメッセージのセット
* `tello_description`にロボット記述ファイル（URDF）が含まれている
* `tello_gazebo`でTelloドローンを[Gazebo](http://gazebosim.org/)上でシミレーションできる、詳しくはそのパッケージ内のREADME.mdを参照してください

## インタフェース

### 概略

このドライバーがシンプルに作られるようにデザインされて、ROSエコシステムにTelloドローンを簡単に統合できるようになっている。

ドライバーはTello SDKを通じてドローンと通信する、そしていくつかの利点がある：
* SDKは文書化されていて、開発活動もかなりアクティブのおかげで安定しつつある。
* TSDKはテキストベースのため、`tello_ros`が任意の文字列を送ることでSDKにシンプルかつフルアクセスできる。

多くのTelloコマンド（(e.g., `takeoff` と `land`）は長く続く、droneが`ok`あるいは`error`を返すことで終了とする。
ドライバーはコマンドを初期化するためのROSサービス`tello_command`を提供し、そしてそれに対応するROSトピック
`tello_response`でコマンドの終了を示す。
※ 訳者注：tello_commandが[tello_action](https://github.com/chaos4ros2/tello_ros/commit/c4e39e71deb570de7425d4a23437009967a34658)に変更されている。

ROSの慣例に従い、ドライバーは`cmd_vel`トピックを通じて`Twist`メッセージのやり取りをする。
それらが`rc`コマンドに変換されてドローンに送られる。
速度は[-1.0, 1.0]から[-100, 100]の間で任意にマップされる。この仕組みは将来変わるかもしれない。

ドライバーはテレメトリーデータを分析し、`flight_data`トピックへ回送する。
テレメトリーデータはドローンが正しく接続されている良いインジケータとなっている。

ドライバーは映像ストリームを解析して`image_raw`トピックに画像を送る。
カメラの情報が`camera_info`トピックに送られる。

Telloドローンは洗練されたビジュアルオドメトリシステムと組込みIMUセンサーモジュールを持っているが、
内部システムへのアクセスは最小限に制限されている。ドライバーはオドメトリを配信しない。

追記事項:
* 毎回実行できるコマンドは一つだけ。
* もしコマンド（`rc`以外）は実行中であれば、配信される`cmd_vel`メッセージが無視される。
* Telloドローンとドライバーは`rc`コマンドにレスポンスを送らない。
* ドライバーは起動時に`command`と`streamon`コマンドを送信することでテレメトリーとビデオを初期化する。
* もしテレメトリーあるいはビデオが停止した場合はドライバーは`command`と`streamon`を送ることで再起動するよう試みる。
* `cmd_vel`メッセージ内のロール(`Twist.angular.x`)とピッチ(`Twist.angular.y`)が無視される。
* ドライバーは状態を記録しない、なのでドローンは地面の上でも`rc`コマンドを送信して構わない。
ドローンはただそれを無視する。
* `tello_action`サービスを通じて任意の文字列をドローンに送ることができる。
* Telloドローンは15秒以内に何もコマンドを受信しなかったら自動着陸する。
そのためドライバーは12秒後に`rc 0 0 0 0`コマンドを送信してそれを回避する。

### サービス

* `~tello_action` tello_msgs/TelloAction

### 購読トピック

* `~cmd_vel` [geometry_msgs/Twist](http://docs.ros.org/api/geometry_msgs/html/msg/Twist.html)

### 配信トピック

* `~tello_response` [std_msgs/String](http://docs.ros.org/api/std_msgs/html/msg/String.html)
* `~flight_data` tello_msgs/FlightData
* `~image_raw` [sensor_msgs/Image](http://docs.ros.org/api/sensor_msgs/html/msg/Image.html)
* `~camera_info` [sensor_msgs/CameraInfo](http://docs.ros.org/api/sensor_msgs/html/msg/CameraInfo.html)

### パラメーター

ドローンは一つしかない場合は以下のデフォルト値は正しく動作する。

 名前         |  説明 |  デフォルト
--------------|--------------|----------
`drone_ip`    | コマンドをこのIPアドレスに送る |  `192.168.10.1`
`drone_port`  | このUDPポートにコマンドを送る | `8889`
`command_port`| このUDPポートから来たコマンドが送られる | `38065`
`data_port`   | フライトデータ（Tello状態）がこのUDPポートに送信される  | `8890`
`video_port`  | 画像データがこのUDPポートに送信される |  `11111`

## インストール

### 1. Linux環境をセットアップする

Ubuntu 18.04の実機あるいは仮想環境をセットアップする。ffmpeg 3.4.4およびOpenCV 3.2を含める必要がある。

asioのインストールも必要：
~~~
sudo apt install libasio-dev
~~~

### 2. ROSの環境をセットアップする

`ros-eloquent-desktop`オプション付きで[ROS2 Eloquent Elusorをインストールする](https://index.ros.org/doc/ros2/Installation/)。

If you install binaries, be sure to also install the 
[development tools and ROS tools](https://github.com/ros2/ros2/wiki/Linux-Development-Setup#install-development-tools-and-ros-tools)
from the source installation instructions.
もしバイナリー（パッケージ）でインストールする場合、ソースコードからインストールする手順に記載されている
[開発ツールとROSツール](https://github.com/ros2/ros2/wiki/Linux-Development-Setup#install-development-tools-and-ros-tools)もインストールしてください。

以下の追加パッケージをインストールする：
~~~
sudo apt install ros-eloquent-cv-bridge ros-eloquent-camera-calibration-parsers
~~~

### 3. `tello_ros`をインストールする

`tello_ros`をダウンロードし、コンパイルしてインストールする：
~~~
mkdir -p ~/tello_ros_ws/src
cd ~/tello_ros_ws/src
git clone https://github.com/clydemcqueen/tello_ros.git
git clone https://github.com/ptrmu/ros2_shared.git
cd ..
source /opt/ros/eloquent/setup.bash
# Gazeboをインストールしてない場合はビルド時にtello_gazeboをスキップしてください：
colcon build --event-handlers console_direct+ --packages-skip tello_gazebo
~~~

## Teleop

ドライバーは有線のXBox Oneコントローラーでドローンを操作するためのシンプルなlaunchファイルを提供する。

ドローンをオンにしてからwi-fiを通じて`TELLO-XXXXX`に接続した後、ROSを起動する:
~~~
cd ~/tello_ros_ws
source install/setup.bash
ros2 launch tello_driver teleop_launch.py
~~~

XBox Oneの**menu**ボタンで離陸し、**view**ボタンで着陸する。

もしXBox Oneコントローラーを持ってない場合はROS2 CLIを通じてコマンドを送ることができる：
~~~~
ros2 service call /tello_action tello_msgs/TelloAction "{cmd: 'takeoff'}"
ros2 service call /tello_action tello_msgs/TelloAction "{cmd: 'rc 0 0 0 20'}"
ros2 service call /tello_action tello_msgs/TelloAction "{cmd: 'land'}"
ros2 service call /tello_action tello_msgs/TelloAction "{cmd: 'battery?'}"
~~~~

また、`cmd_vel`メッセージを送ることもできる：
~~~~
ros2 topic pub /cmd_vel geometry_msgs/Twist  # Sends rc 0 0 0 0
ros2 topic pub /cmd_vel geometry_msgs/Twist "{linear: {x: 0.0, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.2}}"
~~~~

## 確認済みデバイス

* Tello
  * Firmware v01.04.35.01, SDK v1.3
* Tello EDU
  * Firmware v02.04.69.03, SDK v2.0

## バージョンとブランチ

ROS2は急速に変換する間に`tello_ros`はいくつかのプロジェクトを同時に進行してきた。
すべての関連のプロジェクトは（以下のような）似たようなルールでブランチで分けている。
* `master`ブランチはROS2の最新バージョンに対応する（このREADMEを書く時点ではEloquent）
* 古いROS2のバージョンに対応するブランチがいくつかがあるかも、たとえば`crystal`あるいは`dashing`。

以下のプロジェクトとブランチが同時にテストされている：

* ROS Dashing:
  * git clone https://github.com/ptrmu/ros2_shared.git
  * git clone https://github.com/ptrmu/fiducial_vlam.git
  * git clone https://github.com/clydemcqueen/tello_ros.git -b dashing
  * git clone https://github.com/clydemcqueen/flock2.git -b dashing

* ROS2 Eloquent with fiducial_vlam:
  * git clone https://github.com/ptrmu/ros2_shared.git
  * git clone https://github.com/ptrmu/fiducial_vlam.git
  * git clone https://github.com/clydemcqueen/tello_ros.git
  * git clone https://github.com/clydemcqueen/flock2.git

* ROS2 Eloquent with fiducial_vlam_sam:
  * git clone https://github.com/ptrmu/ros2_shared.git
  * git clone https://github.com/ptrmu/fiducial_vlam_sam.git
  * git clone https://github.com/clydemcqueen/sim_fiducial.git
  * git clone https://github.com/clydemcqueen/tello_ros.git
  * git clone https://github.com/clydemcqueen/flock2.git

## 感謝

h264decoderパッケージの参照先：https://github.com/DaWelter/h264decoder

## リソース

* [Tello User Manual 1.4](https://dl-cdn.ryzerobotics.com/downloads/Tello/Tello%20User%20Manual%20v1.4.pdf)
* [SDK 1.3](https://terra-1-g.djicdn.com/2d4dce68897a46b19fc717f3576b7c6a/Tello%20%E7%BC%96%E7%A8%8B%E7%9B%B8%E5%85%B3/For%20Tello/Tello%20SDK%20Documentation%20EN_1.3_1122.pdf)
下の正誤表も参照
* [SDK 2.0](https://dl-cdn.ryzerobotics.com/downloads/Tello/Tello%20SDK%202.0%20User%20Guide.pdf)
下の正誤表も参照
* [Tello EDU Mission Pad Guide (SDK 2.0)](https://dl-cdn.ryzerobotics.com/downloads/Tello/Tello%20Mission%20Pad%20User%20Guide.pdf)
Tello EDU用
* [Tello Pilots Developer Forum](https://tellopilots.com/forums/tello-development.8/)
は良い開発コミュニティー

#### Tello SDK正誤表

* Telloドローンは`rc`コマンドにレスポンスしない（SDKによると`ok`あるいは`error`を返すだそうだ）