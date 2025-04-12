---
title: Bevyでテトリスを作る
tags:
  - game
  - Rust
  - Bevy
private: false
updated_at: '2025-04-12T19:55:26+09:00'
id: bfa3396880424ff4a380
organization_url_name: null
slide: false
ignorePublish: false
---
この記事では、Bevyを使ったテトリスの作り方について説明しています。

テトリスとは、7種類のテトリミノがフィールド上部からランダムに1種類ずつ落下してきて、
うまいこと`10x20`のフィールドの横のラインを揃えるとブロックが消えてポイントが加算される。

これを繰り返して高いスコアを目指す落ちものパズルゲームです。

今回実装したテトリスは以下のようなものです。

![ittoku-tetris.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/922938/2bd29b0f-df3c-4fa6-a3fd-7bc2f5966b8e.gif)

ここでのテトリスでは、本当に最低限の機能を持つゲームとなっています。
なのでゲーム性などを持たせようとするなら追加の機能が必要となるでしょう。

## ソースコード

ゲーム制作に使用したソースコードは以下のURLから入手することができます。

https://github.com/ittokunvim/bevy-tetris

## バージョン

ゲーム制作に使用した｀`Bevy`のバージョンは以下の通りです。
以下のバージョン以外だと動作しない可能性が高いのでご注意ください。

```toml
bevy = "0.15.1"
```

## Cargoを追加

ではここからゲーム制作を開始していきます。

まずは作業を行うディレクトリを作成します。
次に`Bevy`を使用するために`Cargo`というパッケージマネージャを追加します。
ここで動作を確認しておきます。`Hello, World!`がコンソールに表示されたら成功です。

```sh
# ディレクトリを作成
mkdir ittoku-tetris
# Cargoを追加
cargo init
# 動作を確認
cargo run
```

## アセットをダウンロード

次に今回使用するアセットをあらかじめ用意しておきます。

以下のURLに飛んで、下の方の`assets.zip`をクリックしてダウンロードを行います。

ダウンロードが終わったら、ファイルを開いて解凍し、作成したプロジェクト内に移動します。

https://github.com/ittokunvim/bevy-tetris/releases/tag/v0.1.0

## Bevyを追加

次に`Cargo`に`Bevy`を追加します。

`Bevy`を追加するために以下のコマンドを実行します。

```sh
cargo add bevy@0.15.1
```

次に`src/main.rs`に`Bevy`のセットアップコードを追加します。

ここでは、タイトル、画面サイズ、背景色、アセットのパス、カメラなどが設定されています。

```rust
use bevy::prelude::*;

// mod block;
// mod blockdata;
// mod field;
// mod key;

// mod gameover;

const GAMETITLE: &str = "テトリス";
const WINDOW_SIZE: Vec2 = Vec2::new(640.0, 480.0);
const BACKGROUND_COLOR: Color = Color::srgb(0.1, 0.1, 0.1);
const PATH_FONT: &str = "fonts/misaki_gothic.ttf";
const PATH_IMAGE_RETRY: &str = "images/retry.png";
const PATH_SOUND_BGM: &str = "bevy-tetris/bgm.ogg";

const GRID_SIZE: f32 = 20.0;
const BLOCK_SPEED: f32 = 0.5;
const FIELD_SIZE: Vec2 = Vec2::new(10.0 * GRID_SIZE, 20.0 * GRID_SIZE);
const FIELD_POSITION: Vec3 = Vec3::new(0.0, 0.0, -10.0);

#[derive(Event)]
struct MoveEvent(Direction);

#[derive(Event)]
struct RotationEvent(Direction);

#[derive(Event, Default)]
struct SpawnEvent;

#[derive(Event, Default)]
struct FixEvent;

#[derive(Copy, Clone, PartialEq, Debug)]
enum Direction {
    Left,
    Right,
    Bottom,
}

#[derive(Resource, Deref, DerefMut)]
struct FallingTimer(Timer);

#[derive(States, Default, Debug, Clone, PartialEq, Eq, Hash)]
enum AppState {
    #[default]
    InGame,
    Gameover,
}

impl FallingTimer {
    fn new() -> Self {
        Self(Timer::from_seconds(BLOCK_SPEED, TimerMode::Repeating))
    }

    fn update_timer(seconds: f32) -> Timer {
        Timer::from_seconds(seconds, TimerMode::Repeating)
    }
}

fn main() {
    App::new()
        .add_plugins(DefaultPlugins
            .set(WindowPlugin {
                primary_window: Some(Window {
                    resolution: WINDOW_SIZE.into(),
                    title: GAMETITLE.to_string(),
                    ..Default::default()
                }),
                ..Default::default()
            })
        )
        .init_state::<AppState>()
        .insert_resource(ClearColor(BACKGROUND_COLOR))
        .insert_resource(Time::<Fixed>::from_seconds(1.0 / 60.0))
        .add_event::<MoveEvent>()
        .add_event::<RotationEvent>()
        .add_event::<SpawnEvent>()
        .add_event::<FixEvent>()
        .insert_resource(FallingTimer::new())
        // .add_plugins(field::FieldPlugin)
        // .add_plugins(key::KeyPlugin)
        // .add_plugins(block::BlockPlugin)
        // .add_plugins(gameover::GameoverPlugin)
        .add_systems(Startup, setup)
        .run();
}

fn setup(
    mut commands: Commands,
    asset_server: Res<AssetServer>,
) {
    // camera
    commands.spawn(Camera2d::default());
    // bgm
    let sound = AudioPlayer::new(asset_server.load(PATH_SOUND_BGM));
    let settings = PlaybackSettings::LOOP;
    commands.spawn((sound, settings));
}
```

では以下のコマンドを実行して動作を確認してみましょう。
画面が現れて、音楽も流れたら成功です。

```sh
cargo run
```

## フィールドを生成

次にブロックの動かす範囲を指定するフィールドを追加します。

`src/field.rs`を作成し、以下のコードを配置します。

```rust
use bevy::prelude::*;

use crate::{
    FIELD_SIZE,
    FIELD_POSITION,
    AppState,
};

const FIELD_COLOR: Color = Color::srgb(0.6, 0.6, 0.6);

#[derive(Component)]
struct Field;

fn setup(
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    mut materials: ResMut<Assets<ColorMaterial>>,
) {
    // field
    let shape = meshes.add(Rectangle::new(FIELD_SIZE.x, FIELD_SIZE.y));
    commands.spawn((
        Mesh2d(shape),
        MeshMaterial2d(materials.add(FIELD_COLOR)),
        Transform::from_xyz(FIELD_POSITION.x, FIELD_POSITION.y, FIELD_POSITION.z),
        Field,
    ));
}

fn despawn(
    mut commands: Commands,
    query: Query<Entity, With<Field>>,
) {
    for entity in &query {
        commands.entity(entity).despawn();
    }
}

pub struct FieldPlugin;

impl Plugin for FieldPlugin {
    fn build(&self, app: &mut App) {
        app
            .add_systems(OnEnter(AppState::InGame), setup)
            .add_systems(OnExit(AppState::Gameover), despawn)
        ;
    }
}
```

そして`src/main.rs`の以下のコードのコメントをはずします。

```rust
// ...

mod field;

// ...
fn main() {
    App::new()
        // ...
        .add_plugins(field::FieldPlugin)
        // ...
}
```

では`cargo run`を実行して動作を確認してみましょう。
ゲーム画面に四角いフィールドが描画されたら成功です。

## キーを設定

次は押されたキーに応じてイベントを振り分けます。

ここではブロックが、右左下に移動、右左回転するイベントを振り分けています。

`src/key.rs`を作成して、以下のコードを記述します。

```rust
use bevy::prelude::*;

use crate::{
    BLOCK_SPEED,
    MoveEvent,
    RotationEvent,
    Direction,
    FallingTimer,
    AppState,
};

const KEY_BLOCK_LEFT_1: KeyCode = KeyCode::ArrowLeft;
const KEY_BLOCK_LEFT_2: KeyCode = KeyCode::KeyA;
const KEY_BLOCK_RIGHT_1: KeyCode = KeyCode::ArrowRight;
const KEY_BLOCK_RIGHT_2: KeyCode = KeyCode::KeyD;
const KEY_BLOCK_BOTTOM_1: KeyCode = KeyCode::ArrowDown;
const KEY_BLOCK_BOTTOM_2: KeyCode = KeyCode::KeyS;
const KEY_BLOCK_ROTATION_LEFT: KeyCode = KeyCode::KeyZ;
const KEY_BLOCK_ROTATION_RIGHT: KeyCode = KeyCode::ArrowUp;

fn move_event(
    mut events: EventWriter<MoveEvent>,
    mut timer: ResMut<FallingTimer>,
    keyboard_input: Res<ButtonInput<KeyCode>>,
) {
    let mut closure = |direction: Direction| {
        events.send(MoveEvent(direction));
        if direction == Direction::Bottom {
            timer.0 = FallingTimer::update_timer(BLOCK_SPEED / 2.0);
        }
    };
    for key in keyboard_input.get_just_pressed() {
        match key {
            &KEY_BLOCK_LEFT_1   | &KEY_BLOCK_LEFT_2   => closure(Direction::Left),
            &KEY_BLOCK_RIGHT_1  | &KEY_BLOCK_RIGHT_2  => closure(Direction::Right),
            &KEY_BLOCK_BOTTOM_1 | &KEY_BLOCK_BOTTOM_2 => closure(Direction::Bottom),
            _ => {},
        }
    }
    for key in keyboard_input.get_just_released() {
        if key == &KEY_BLOCK_BOTTOM_1 || key == &KEY_BLOCK_BOTTOM_2 {
            timer.0 = FallingTimer::update_timer(BLOCK_SPEED);
        }
    }
}

fn rotation_event(
    mut events: EventWriter<RotationEvent>,
    keyboard_input: Res<ButtonInput<KeyCode>>,
) {
    let mut closure = |direction: Direction| {
        events.send(RotationEvent(direction));
    };
    for key in keyboard_input.get_just_pressed() {
        match key {
            &KEY_BLOCK_ROTATION_LEFT  => closure(Direction::Left),
            &KEY_BLOCK_ROTATION_RIGHT => closure(Direction::Right),
            _ => {},
        };
    }
}

pub struct KeyPlugin;

impl Plugin for KeyPlugin {
    fn build(&self, app: &mut App) {
        app
            .add_systems(Update, (
                move_event,
                rotation_event,
            ).run_if(in_state(AppState::InGame)))
        ;
    }
}
```

そして`src/main.rs`の以下のコードのコメントをはずします。

```rust
// ...

mod key;

// ...
fn main() {
    App::new()
        // ...
        .add_plugins(key::KeyPlugin)
        // ...
}
```

では`cargo run`を実行して動作を確認してみましょう。
エラーが出なければ成功です。


## ブロックのデータを設定

次にテトリスのブロックのデータを設定します。

ここではマップ、ブロックの形、ブロックの色を定義しています。

ブロックマップは縦24マス、横10マスに定義されています。
この値はブロックを削除するときに使用されます。
縦のマスがなぜ24マスかというと、ブロックの位置が決められるときに、ブロックがフィールド上部からはみ出す可能性があるからです。
ブロックがはみ出ても値を保持できるように4つ余分に設定しています。

ブロックの形は`I,J,L,O,S,T,Z`の7種類で、それぞれに回転した後のブロックの位置を定義しています。

ブロックの色は`I,J,L,O,S,T,Z`の7種類、用意しています。

`src/blockdata.rs`を作成し、以下のコードを記述します。

```rust
use bevy::prelude::*;

pub const BLOCK_MAP: [[usize; 10]; 24] = [
    [0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0],
];
pub const I_BLOCK: [[usize; 16]; 4] = [
    [
        0,0,0,0,
        1,2,3,4,
        0,0,0,0,
        0,0,0,0,
    ],
    [
        0,0,1,0,
        0,0,2,0,
        0,0,3,0,
        0,0,4,0,
    ],
    [
        0,0,0,0,
        0,0,0,0,
        1,2,3,4,
        0,0,0,0,
    ],
    [
        0,1,0,0,
        0,2,0,0,
        0,3,0,0,
        0,4,0,0,
    ],
];
pub const J_BLOCK: [[usize; 16]; 4] = [
    [
        1,0,0,0,
        2,3,4,0,
        0,0,0,0,
        0,0,0,0,
    ],
    [
        0,1,2,0,
        0,3,0,0,
        0,4,0,0,
        0,0,0,0,
    ],
    [
        0,0,0,0,
        1,2,3,0,
        0,0,4,0,
        0,0,0,0,
    ],
    [
        0,1,0,0,
        0,2,0,0,
        3,4,0,0,
        0,0,0,0,
    ],
];
pub const L_BLOCK: [[usize; 16]; 4] = [
    [
        0,0,1,0,
        4,3,2,0,
        0,0,0,0,
        0,0,0,0,
    ],
    [
        0,1,0,0,
        0,2,0,0,
        0,3,4,0,
        0,0,0,0,
    ],
    [
        0,0,0,0,
        1,2,3,0,
        4,0,0,0,
        0,0,0,0,
    ],
    [
        1,2,0,0,
        0,3,0,0,
        0,4,0,0,
        0,0,0,0,
    ],
];
pub const O_BLOCK: [[usize; 16]; 4] = [
    [
        0,0,0,0,
        0,1,2,0,
        0,3,4,0,
        0,0,0,0,
    ],
    [
        0,0,0,0,
        0,1,2,0,
        0,3,4,0,
        0,0,0,0,
    ],
    [
        0,0,0,0,
        0,1,2,0,
        0,3,4,0,
        0,0,0,0,
    ],
    [
        0,0,0,0,
        0,1,2,0,
        0,3,4,0,
        0,0,0,0,
    ],
];
pub const S_BLOCK: [[usize; 16]; 4] = [
    [
        0,0,0,0,
        0,1,2,0,
        3,4,0,0,
        0,0,0,0,
    ],
    [
        0,1,0,0,
        0,2,3,0,
        0,0,4,0,
        0,0,0,0,
    ],
    [
        0,2,1,0,
        3,4,0,0,
        0,0,0,0,
        0,0,0,0,
    ],
    [
        0,1,0,0,
        0,2,3,0,
        0,0,4,0,
        0,0,0,0,
    ],
];
pub const T_BLOCK: [[usize; 16]; 4] = [
    [
        0,1,0,0,
        2,3,4,0,
        0,0,0,0,
        0,0,0,0,
    ],
    [
        0,1,0,0,
        0,2,3,0,
        0,4,0,0,
        0,0,0,0,
    ],
    [
        0,0,0,0,
        1,2,3,0,
        0,4,0,0,
        0,0,0,0,
    ],
    [
        0,1,0,0,
        2,3,0,0,
        0,4,0,0,
        0,0,0,0,
    ],
];
pub const Z_BLOCK: [[usize; 16]; 4] = [
    [
        0,0,0,0,
        1,2,0,0,
        0,3,4,0,
        0,0,0,0,
    ],
    [
        0,0,1,0,
        0,2,3,0,
        0,4,0,0,
        0,0,0,0,
    ],
    [
        1,2,0,0,
        0,3,4,0,
        0,0,0,0,
        0,0,0,0,
    ],
    [
        0,0,1,0,
        0,2,3,0,
        0,4,0,0,
        0,0,0,0,
    ],
];
pub const I_COLOR: Color = Color::srgb(0.0, 0.0, 1.0);
pub const J_COLOR: Color = Color::srgb(0.0, 1.0, 0.0);
pub const L_COLOR: Color = Color::srgb(0.0, 1.0, 1.0);
pub const O_COLOR: Color = Color::srgb(1.0, 0.0, 0.0);
pub const S_COLOR: Color = Color::srgb(1.0, 0.0, 1.0);
pub const T_COLOR: Color = Color::srgb(1.0, 1.0, 0.0);
pub const Z_COLOR: Color = Color::srgb(1.0, 1.0, 1.0);

```

そして`src/main.rs`の以下のコードのコメントをはずします。

```rust
// ...

mod blockdata;

// ...
```

では`cargo run`を実行して動作を確認してみましょう。
エラーが出なければ成功です。

## ブロックを実装

次は本命のブロックの実装を行なっていきます。

追加するブロックの主な処理は、生成、移動、回転、削除の4種類です。

`src/block/mod.rs`を作成し、以下のコードを記述します。

ここではブロックの処理に関する様々な定数、リソース、コンポーネント、メソッドなどが定義されています。

そしてフィールド上部の真ん中の2マスにブロックが固定されたらゲームオーバーになる処理もここで定義されています。

```rust
use bevy::prelude::*;

use crate::{
    GRID_SIZE,
    FIELD_SIZE,
    FIELD_POSITION,
    SpawnEvent,
    FixEvent,
    AppState,
};
use crate::blockdata::{
    BLOCK_MAP,
    I_BLOCK,
};

// mod clear;
// mod movement;
// mod rotation;
// mod spawn;

const MAX_BLOCK_COUNT: usize = 4;
const MAX_COLLISION_COUNT: usize = 3;
const BLOCK_SIZE: f32 = GRID_SIZE - 1.0;
const BLOCK_POSITION: Vec3 = Vec3::new(
    FIELD_POSITION.x + GRID_SIZE / 2.0 - GRID_SIZE * 2.0,
    FIELD_POSITION.y + GRID_SIZE / 2.0 + FIELD_SIZE.y / 2.0 - GRID_SIZE * 1.0,
    10.0,
);
const FIELD_LEFT_TOP: Vec2 = Vec2::new(
    FIELD_POSITION.x - FIELD_SIZE.x / 2.0 + GRID_SIZE / 2.0, 
    FIELD_POSITION.y + FIELD_SIZE.y / 2.0 - GRID_SIZE / 2.0,
);

/// ブロック回転時に用いるリソース
///
/// idには[usize; 16]で定義されているindexが格納される
/// posには回転時に軸となるXYZ軸が定義される
#[derive(Resource)]
struct RotationBlock {
    id: usize,
    pos: Vec3,
}

/// ブロック削除時に用いるリソース
///
/// 値は[[usize; 10]; 24]で定義されており
/// フィールド内の各ブロック座標が0 or 1で格納されている
#[derive(Resource)]
struct BlockMap([[usize; 10]; 24]);

/// 移動、回転するブロックを識別するコンポーネント
///
/// 値には1~4に定義されているブロックのIDが格納される
#[derive(Component)]
struct PlayerBlock(usize);

/// 移動、回転しないブロックを識別するコンポーネント
///
/// ブロック削除時に使用される
#[derive(Component)]
struct Block;

impl RotationBlock {
    // リソースを初期化
    fn new() -> Self {
        RotationBlock {
            id: 0,
            pos: BLOCK_POSITION,
        }
    }
    /// 渡されたブロックIDの回転後のブロックの位置を返すメソッド
    ///
    /// # Arguments
    /// * id - 回転後のブロックの位置を取得するためのブロックID
    ///
    /// # Returns
    /// * Vec3 - 回転後のブロックの位置
    ///
    /// # Panics
    /// * idが見つからない場合
    fn position(&self, id: usize) -> Vec3 {
        // ブロックIDが有効範囲内かチェック
        assert!(self.id < I_BLOCK.len());
        // 回転後のブロックの位置を見つける
        for (index, value) in I_BLOCK[self.id].iter().enumerate() {
            if id == *value {
                // ブロックの新しい位置を計算して返す
                let (x, y, z) = (
                    self.pos.x + GRID_SIZE * ((index % 4) as f32),
                    self.pos.y - GRID_SIZE * ((index / 4) as f32),
                    self.pos.z,
                );
                return Vec3::new(x, y, z);
            }
        }
        // ブロックIDが見つからなかったらパニック
        panic!("id not found: {}", id);
    }
}

impl BlockMap {
    /// 渡されたブロックの座標からブロックマップに値を代入し
    /// そのブロックマップを返すメソッド
    ///
    /// # Arguments
    /// * pos - ブロックの座標
    ///
    /// # Returns
    /// * [[usize; 10]; 24] - 更新されたブロックマップ
    ///
    /// # Panics
    /// * 指定された座標が見つからない場合
    fn insert(&self, pos: Vec2) -> [[usize; 10]; 24] {
        let mut block_map = self.0;
        // ブロック座標にブロックマップを追加
        for y in 0..block_map.len() {
            for x in 0..block_map[0].len() {
                let current_pos = Vec2::new(
                    FIELD_LEFT_TOP.x + GRID_SIZE * x as f32, 
                    FIELD_LEFT_TOP.y + GRID_SIZE * 4.0 - GRID_SIZE * y as f32,
                );
                if current_pos == pos {
                    block_map[y][x] = 1;
                    return block_map
                }
            }
        }
        panic!("pos no found: {}", pos);
    }
    /// 渡された削除するブロックの列のIDを参照して
    /// 消されるブロックをブロックマップに更新し
    /// ブロック削除後のブロックマップを返すメソッド
    ///
    /// # Arguments
    /// * index - 削除するブロックの列のID
    ///
    /// # Returns
    /// * [[usize; 10]; 24] - 更新されたブロックマップ
    fn clearline(&self, index: usize) -> [[usize; 10]; 24] {
        let mut block_map = self.0;
        // clear index line
        block_map[index] = [0; 10];
        // shift down one by one
        for i in (1..=index).rev() {
            block_map[i] = block_map[i - 1];
        }
        // clear top line
        block_map[0] = [0; 10];
        block_map
    }
}

fn setup(mut events: EventWriter<SpawnEvent>) { events.send_default(); }

/// ゲームオーバーを管理する関数
/// `FixEvent`を受け取り、固定されたブロックから
/// ゲームオーバーになるかどうかチェックします
///
fn gameover(
    mut events: EventReader<FixEvent>,
    mut next_state: ResMut<NextState<AppState>>,
    query: Query<&Transform, With<PlayerBlock>>,
) {
    // イベントをチェック
    if events.is_empty() {
        return;
    }

    // イベントをクリア
    events.clear();

    // ゲームオーバーかどうか判定する
    for transform in &query {
        let pos = transform.translation;
        if pos.y >= FIELD_LEFT_TOP.y {
            if pos.x == FIELD_LEFT_TOP.x + GRID_SIZE * 5.0
            || pos.x == FIELD_LEFT_TOP.x + GRID_SIZE * 6.0 {
                next_state.set(AppState::Gameover);
                return;
            }
        }
    }
}

fn despawn(
    mut commands: Commands,
    query: Query<Entity, With<Block>>,
) {
    for entity in &query {
        commands.entity(entity).despawn();
    }
}

fn reset(
    mut rotation_block: ResMut<RotationBlock>,
    mut block_map: ResMut<BlockMap>,
) {
    *rotation_block = RotationBlock::new();
    *block_map = BlockMap(BLOCK_MAP);
}

pub struct BlockPlugin;

impl Plugin for BlockPlugin {
    fn build(&self, app: &mut App) {
        app
            .insert_resource(RotationBlock::new())
            .insert_resource(BlockMap(BLOCK_MAP))
            .add_systems(OnEnter(AppState::InGame), setup)
            .add_systems(Update, (
                // spawn::block_spawn,
                // movement::block_falling,
                // rotation::block_rotation,
                // movement::block_movement,
                gameover,
                // clear::block_clear,
            ).chain().run_if(in_state(AppState::InGame)))
            .add_systems(OnExit(AppState::Gameover), despawn)
            .add_systems(OnExit(AppState::Gameover), reset)
        ;
    }
}
```

そして`src/main.rs`の以下のコードのコメントをはずします。

```rust
// ...

mod block;

// ...
fn main() {
    App::new()
        // ...
        .add_plugins(block::BlockPlugin)
        // ...
}
```

では`cargo run`を実行して動作を確認してみましょう。
エラーが出なければ成功です。

### 生成処理を実装

次にブロックの生成処理を実装していきます。

この処理ではまずゲーム開始時にブロックをフィールド上部に生成し、
ブロックが固定されたら再度、ブロックを生成するといったことをしています。

`src/block/spawn.rs`を作成し、以下の記述を行います。

```rust
use bevy::prelude::*;

use crate::{
    GRID_SIZE,
    SpawnEvent,
};
use crate::block::{
    BLOCK_POSITION,
    BLOCK_SIZE,
    RotationBlock,
    PlayerBlock,
    Block,
};
use crate::blockdata::{
    I_BLOCK,
    I_COLOR,
};

/// ブロック生成イベントを処理する関数
/// `SpawnEvent`を受け取り、新しいブロックを生成してフィールドに配置します
///
pub fn block_spawn(
    mut events: EventReader<SpawnEvent>,
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    mut materials: ResMut<Assets<ColorMaterial>>,
    mut rotation_block: ResMut<RotationBlock>,
    query: Query<&Transform, With<Block>>,
) {
    // イベントをチェック
    if events.is_empty() {
        return;
    }

    // イベントをクリア
    events.clear();

    // RotationBlockをリセット
    *rotation_block = RotationBlock::new();

    // PlayerBlockを生成
    let shape = meshes.add(Rectangle::new(BLOCK_SIZE, BLOCK_SIZE));
    let mut init_position = BLOCK_POSITION;

    for (index, value) in I_BLOCK[0].iter().enumerate() {
        // ブロックの値が0であればスキップ
        if *value == 0 {
            continue;
        }

        // ブロックの位置を計算
        let mut position = Vec3::new(
            init_position.x + GRID_SIZE * ((index % 4) as f32),
            init_position.y - GRID_SIZE * ((index / 4) as f32),
            init_position.z,
        );

        // ブロックが同士が被らないように位置を計算
        for transform in &query {
            if position == transform.translation {
                position.y += GRID_SIZE;
                init_position.y += GRID_SIZE;
            }
        }

        // PlayerBlockを生成
        commands.spawn((
            Mesh2d(shape.clone()),
            MeshMaterial2d(materials.add(I_COLOR)),
            Transform::from_xyz(position.x, position.y, position.z),
            PlayerBlock(*value),
        ));
    }
}
```

そして`src/block/mod.rs`の以下のコードのコメントをはずします。

```rust
// ...

mod spawn;

// ...
impl Plugin for BlockPlugin {
    fn build(&self, app: &mut App) {
        app
            // ...
            .add_systems(Update, (
                spawn::block_spawn,
                // ...
            ).chain().run_if(in_state(AppState::InGame)))
    }
}
```

では`cargo run`を実行して動作を確認してみましょう。
フィールド上部にブロックが描画されていれば成功です。

### 移動処理を実装

次はブロックの移動処理を実装していきます。

ここではブロックが落下する処理、キーを押したときにブロックが移動する処理、
ブロックがフィールドからはみ出る場合やブロックに当たる場合、移動させない処理などを追加しています。

`src/block/movement.rs`を作成し、以下のコードを記述します。

```rust
use bevy::prelude::*;

use crate::{
    GRID_SIZE,
    FIELD_SIZE,
    FIELD_POSITION,
    MoveEvent,
    FixEvent,
    Direction,
    FallingTimer,
};
use crate::block::{
    RotationBlock,
    PlayerBlock,
    Block,
};

/// ブロックの落下を管理する関数
/// `FallingTimer`を使用して一定間隔でブロックを下に移動させる
///
pub fn block_falling(
    mut timer: ResMut<FallingTimer>,
    mut events: EventWriter<MoveEvent>,
    time: Res<Time>,
) {
    // タイマーを進める
    timer.tick(time.delta());

    // タイマーが終わったかチェック
    if !timer.just_finished() {
        return;
    }

    // ブロックを下に移動させるイベントを送信
    events.send(MoveEvent(Direction::Bottom));
}

/// ブロックの移動を管理する関数
/// `MoveEvent`を受け取り、ブロックの位置を更新し、
/// 必要に応じてブロックを固定する
///
pub fn block_movement(
    mut move_events: EventReader<MoveEvent>,
    mut fix_events: EventWriter<FixEvent>,
    mut player_query: Query<&mut Transform, (With<PlayerBlock>, Without<Block>)>,
    mut rotation_block: ResMut<RotationBlock>,
    block_query: Query<&Transform, With<Block>>,
) {
    for event in move_events.read() {
        let direction = event.0;

        // フィールドの衝突をチェック
        for player_transform in &mut player_query {
            let player_x = player_transform.translation.x;
            let player_y = player_transform.translation.y;

            match direction {
                Direction::Left => {
                    if player_x - GRID_SIZE < FIELD_POSITION.x - FIELD_SIZE.x / 2.0 {
                        return;
                    }
                }
                Direction::Right => {
                    if player_x + GRID_SIZE > FIELD_POSITION.x + FIELD_SIZE.x / 2.0 {
                        return;
                    }
                }
                Direction::Bottom => {
                    if player_y - GRID_SIZE < FIELD_POSITION.y - FIELD_SIZE.y / 2.0 {
                        // ブロックがそこに達した場合、ブロックを固定
                        fix_events.send_default();
                        return;
                    }
                }
            }

            // ブロックの衝突をチェック
            for block_transform in &block_query {
                let block_x = block_transform.translation.x;
                let block_y = block_transform.translation.y;

                match direction {
                    Direction::Left => {
                        if player_x - GRID_SIZE == block_x && player_y == block_y {
                            return;
                        }
                    }
                    Direction::Right => {
                        if player_x + GRID_SIZE == block_x && player_y == block_y {
                            return;
                        }
                    }
                    Direction::Bottom => {
                        if player_x == block_x && player_y - GRID_SIZE == block_y {
                            // ブロックが底に達した場合、ブロックを固定
                            fix_events.send_default();
                            return;
                        }
                    }
                }
            }
        }

        // 現在のブロック位置を更新
        match direction {
            Direction::Left   => rotation_block.pos.x -= GRID_SIZE,
            Direction::Right  => rotation_block.pos.x += GRID_SIZE,
            Direction::Bottom => rotation_block.pos.y -= GRID_SIZE,
        }
        // ブロックを移動
        for mut transform in &mut player_query {
            match direction {
                Direction::Left   => transform.translation.x -= GRID_SIZE,
                Direction::Right  => transform.translation.x += GRID_SIZE,
                Direction::Bottom => transform.translation.y -= GRID_SIZE,
            }
        }
    }
}
```

そして`src/block/mod.rs`の以下のコードのコメントをはずします。

```rust
// ...

mod movement;

// ...
impl Plugin for BlockPlugin {
    fn build(&self, app: &mut App) {
        app
            // ...
            .add_systems(Update, (
                // ...
                movement::block_falling,
                // ...
                movement::block_movement,
                // ...
            ).chain().run_if(in_state(AppState::InGame)))
    }
}
```

では`cargo run`を実行して動作を確認してみましょう。
ブロックが落下して、移動することができたら成功です。

### 回転処理を実装

次はブロックの回転処理を実装していきます。

ここではブロックが回転する処理、フィールドとブロックの衝突判定、
もし衝突判定でブロックが3マス以上動くことになるなら、ブロックを回転させない処理などが書かれています。

`src/block/rotation.rs`を作成し、以下のコードを記述します。

```rust
use bevy::prelude::*;

use crate::{
    GRID_SIZE,
    FIELD_SIZE,
    FIELD_POSITION,
    RotationEvent,
    Direction,
    FallingTimer,
};
use crate::block::{
    MAX_BLOCK_COUNT,
    MAX_COLLISION_COUNT,
    RotationBlock,
    PlayerBlock,
    Block,
};

/// ブロックの回転を管理する関数
/// `RotationEvent`を受け取り、ブロックの位置を更新し、
/// 必要に応じてブロックの衝突を処理します
///
pub fn block_rotation(
    mut events: EventReader<RotationEvent>,
    mut timer: ResMut<FallingTimer>,
    mut player_query: Query<(&PlayerBlock, &mut Transform), (With<PlayerBlock>, Without<Block>)>,
    mut rotation_block: ResMut<RotationBlock>,
    block_query: Query<&Transform, With<Block>>,
) {
    for event in events.read() {
        let direction = event.0;
        let mut count = 0;
        let mut collision_x = 0.0;
        let mut collision_y = 0.0;

        // タイマーをリセット
        timer.reset();

        // 現在のブロックIDを更新
        rotation_block.id = match direction {
            Direction::Right => (rotation_block.id + 1) % MAX_BLOCK_COUNT,
            Direction::Left  => (rotation_block.id + MAX_BLOCK_COUNT - 1) % MAX_BLOCK_COUNT,
            _ => rotation_block.id,
        };

        // 衝突をチェック
        for (player, mut _player_transform) in &mut player_query {
            while count < MAX_COLLISION_COUNT {
                // 回転時のブロックの位置を取得
                let position = rotation_block.position(player.0);

                // フィールド左側の衝突判定
                if position.x < FIELD_POSITION.x - FIELD_SIZE.x / 2.0 {
                    rotation_block.pos.x += GRID_SIZE;
                    collision_x += GRID_SIZE;
                    count += 1;
                }
                // フィールド右側の衝突判定
                else if position.x > FIELD_POSITION.x + FIELD_SIZE.x / 2.0 {
                    rotation_block.pos.x -= GRID_SIZE;
                    collision_x -= GRID_SIZE;
                    count += 1;
                }
                // フィールド下側の衝突判定
                else if position.y < FIELD_POSITION.y - FIELD_SIZE.y / 2.0 {
                    rotation_block.pos.y += GRID_SIZE;
                    collision_y += GRID_SIZE;
                    count += 1;
                }
                // ブロック同士の衝突判定
                else if block_query.iter().any(|block_transform|
                    position == block_transform.translation
                ) {
                    rotation_block.pos.y += GRID_SIZE;
                    collision_y += GRID_SIZE;
                    count += 1;
                }
                // 衝突がなければループを抜ける
                else { break; }
            }
        }

        // もし衝突判定が規定回数以上あった場合、回転を行わない
        if count >= MAX_COLLISION_COUNT {
            // 現在のブロックIDをリセット
            rotation_block.id = match direction {
                Direction::Right => (rotation_block.id + MAX_BLOCK_COUNT - 1) % MAX_BLOCK_COUNT,
                Direction::Left  => (rotation_block.id + 1) % MAX_BLOCK_COUNT,
                _ => rotation_block.id,
            };
            // 現在のブロック位置をリセット
            rotation_block.pos.x -= collision_x;
            rotation_block.pos.y -= collision_y;
            return;
        }
        // ブロックを回転させる
        for (player, mut player_transform) in &mut player_query {
            player_transform.translation = rotation_block.position(player.0);
        }
    }
}
```

そして`src/block/mod.rs`の以下のコードのコメントをはずします。

```rust
// ...

mod rotation;

// ...
impl Plugin for BlockPlugin {
    fn build(&self, app: &mut App) {
        app
            // ...
            .add_systems(Update, (
                // ...
                rotation::block_rotation,
                // ...
            ).chain().run_if(in_state(AppState::InGame)))
    }
}
```

では`cargo run`を実行して動作を確認してみましょう。
ブロックが回転することができたら成功です。

### 削除処理を実装

次はブロックを削除する処理を実装していきます。

ここでは`BlockMap`を参照してブロックが横一列に揃っている場合、
その列を削除し、削除した列より上のブロックを削除した列の分、下に下げる処理を実装しています。

`src/block/clear.rs`を作成し、以下のコードを記述します。

```rust
use bevy::prelude::*;

use crate::{
    GRID_SIZE,
    SpawnEvent,
    FixEvent,
};
use crate::block::{
    FIELD_LEFT_TOP,
    BlockMap,
    PlayerBlock,
    Block,
};

/// ブロックの削除を管理する関数
/// `FixEvent`を受け取り、プレイヤーブロックを固定ブロックに変換し、
/// ブロックマップを更新して、ラインが揃った場合にブロックを削除します。
///
pub fn block_clear(
    mut fix_events: EventReader<FixEvent>,
    mut commands: Commands,
    mut player_query: Query<(Entity, &mut Transform), (With<PlayerBlock>, Without<Block>)>,
    mut block_query: Query<(Entity, &mut Transform), (With<Block>, Without<PlayerBlock>)>,
    mut block_map: ResMut<BlockMap>,
    mut spawn_events: EventWriter<SpawnEvent>,
) {
    // イベントをチェック
    if fix_events.is_empty() {
        return;
    }

    // イベントをクリア
    fix_events.clear();

    // PlayerBlockをBlockに変換
    for (player_entity, player_transform) in &player_query {
        commands.entity(player_entity).remove::<PlayerBlock>();
        commands.entity(player_entity).insert(Block);

        // BlockMapを更新
        let pos = player_transform.translation.truncate();
        block_map.0 = block_map.insert(pos);
    }

    let map = block_map.0;

    // ブロックを削除
    for (index, row) in map.iter().enumerate() {
        if *row == [1; 10] {
            let y = FIELD_LEFT_TOP.y + GRID_SIZE * 4.0 - GRID_SIZE * index as f32;
            block_map.0 = block_map.clearline(index);

            // プレイヤーブロックをチェック
            for (player_entity, mut player_transform) in &mut player_query {
                if player_transform.translation.y == y {
                    commands.entity(player_entity).despawn();
                }
                if player_transform.translation.y > y {
                    player_transform.translation.y -= GRID_SIZE;
                }
            }

            // 固定ブロックをチェック
            for (block_entity, mut block_transform) in &mut block_query {
                if block_transform.translation.y == y {
                    commands.entity(block_entity).despawn();
                }
                if block_transform.translation.y > y {
                    block_transform.translation.y -= GRID_SIZE;
                }
            }
        }
    }

    // ブロックを生成するイベントを送信
    spawn_events.send_default();
}
```

そして`src/block/mod.rs`の以下のコードのコメントをはずします。

```rust
// ...

mod clear;

// ...
impl Plugin for BlockPlugin {
    fn build(&self, app: &mut App) {
        app
            // ...
            .add_systems(Update, (
                // ...
                clear::block_clear,
            ).chain().run_if(in_state(AppState::InGame)))
    }
}
```

では`cargo run`を実行して動作を確認してみましょう。

ここで一通りのブロックの実装は完了です。
ブロックを操作して横一列に揃えてみましょう。
ブロックが消えたら成功です。

## ゲームオーバーを実装

最後にゲームオーバーになった時のセットアップと処理を実装していきます。

ここではゲームオーバー画面の描画、リトライボタンを押したら再度ゲームを開始することができる処理を追加しています。

`src/gameover.rs`を作成し以下のコードを記述します。

```rust
use bevy::prelude::*;

use crate::{
    WINDOW_SIZE,
    PATH_FONT,
    PATH_IMAGE_RETRY,
    AppState,
};

const BOARD_WIDTH: Val = Val::Px(360.0);
const BOARD_HEIGHT: Val = Val::Px(270.0);
const BOARD_LEFT: Val = Val::Px(WINDOW_SIZE.x / 2.0 - 360.0 / 2.0);
const BOARD_TOP: Val = Val::Px(WINDOW_SIZE.y / 2.0 - 270.0 / 2.0);
const BOARD_PADDING: Val = Val::Px(16.0);
const BOARD_COLOR: Color = Color::srgb(0.9, 0.9, 0.9);

const GAMEOVER_TEXT: &str = "ゲームオーバー";
const GAMEOVER_FONT_SIZE: f32 = 24.0;
const GAMEOVER_COLOR: Color = Color::srgb(0.1, 0.1, 0.1);

const RETRY_SIZE: Vec2 = Vec2::new(24.0, 24.0);
const RETRY_BACKGROUND_COLOR_HOVER: Color = Color::srgb(0.8, 0.8, 0.8);

const BORDER_SIZE: Val = Val::Px(4.0);
const BORDER_COLOR: Color = Color::srgb(0.5, 0.5, 1.0);
const BORDER_RADIUS: Val = Val::Px(10.0);

#[derive(Component)]
struct Gameover;

impl Gameover {
    /// ゲームオーバー画面のルートノードを生成します
    ///
    /// Returns:
    /// * `Self`: Gameoverのインスタンス。
    /// * `Node`: 幅と高さが100%のルートノード。
    fn from_root() -> (Self, Node) {
        (
            Self,
            Node {
                width: Val::Percent(100.0),
                height: Val::Percent(100.0),
                ..Default::default()
            }
        )
    }

    /// ゲームオーバー画面の背景を生成します。
    ///
    /// Returns:
    /// * `Self`: Gameoverのインスタンス。
    /// * `Node`: 背景のサイズ、場所、並び方などが定義されたノード。 
    /// * `BackgroundColor`: 背景色
    /// * `BorderColor`: ボーダーの色
    /// * `BorderRadius`: ボーダーのラディウス
    fn from_board() -> (Self, Node, BackgroundColor, BorderColor, BorderRadius) {
        (
            Self,
            Node {
                width: BOARD_WIDTH,
                height: BOARD_HEIGHT,
                border: UiRect::all(BORDER_SIZE),
                position_type: PositionType::Absolute,
                left: BOARD_LEFT,
                top: BOARD_TOP,
                padding: UiRect::all(BOARD_PADDING),
                justify_content: JustifyContent::SpaceBetween,
                align_items: AlignItems::Center,
                flex_direction: FlexDirection::Column,
                ..Default::default()
            },
            BackgroundColor(BOARD_COLOR),
            BorderColor(BORDER_COLOR),
            BorderRadius::all(BORDER_RADIUS),
        )
    }

    /// ゲームオーバーメッセージを表示するテキストを生成します。
    ///
    /// Params:
    /// * `font`: テキストに使用するフォント
    ///
    /// Returns:
    /// * `Self`: Gameoverのインスタンス。
    /// * `Text`: ゲームオーバーメッセージのテキスト。
    /// * `TextFont`: フォントスタイル。
    /// * `TextColor`: テキストの色
    fn from_text(font: Handle<Font>) -> (Self, Text, TextFont, TextColor) {
        (
            Self,
            Text::new(GAMEOVER_TEXT),
            TextFont {
                font: font.clone(),
                font_size: GAMEOVER_FONT_SIZE,
                ..Default::default()
            },
            TextColor(GAMEOVER_COLOR),
        )
    }

    /// ゲームオーバー画面に表示する「リトライ」ボタンを生成します。
    ///
    /// Returns:
    /// * `Self`: Gameoverのインスタンス。
    /// * `Node`: リトライボタンを表すノード。
    /// * `BorderColor`: ボーダーの色
    /// * `BorderRadius`: ボーダーのラディウス
    /// * `Button`: ボタンコンポーネント
    fn from_retry() -> (Self, Node, BorderColor, BorderRadius, Button) {
        (
            Self,
            Node {
                width: Val::Px(RETRY_SIZE.x * 2.0),
                height: Val::Px(RETRY_SIZE.y * 2.0),
                border: UiRect::all(BORDER_SIZE),
                justify_content: JustifyContent::Center,
                align_items: AlignItems::Center,
                ..Default::default()
            },
            BorderColor(BORDER_COLOR),
            BorderRadius::all(BORDER_RADIUS),
            Button,
        )
    }

    /// ゲームオーバー画面に表示するリトライアイコンを生成します。
    ///
    /// Params:
    /// * `image`: リトライアイコン
    ///
    /// Returns:
    /// * `Self`: Gameoverのインスタンス。
    /// * `ImageNode`: 画像のノード
    /// * `Node`: リトライアイコンのサイズ、レイアウトを表すノード。
    fn from_retry_icon(image: Handle<Image>) -> (Self, ImageNode, Node) {
        (
            Self,
            ImageNode::new(image.clone()),
            Node {
                width: Val::Px(RETRY_SIZE.x),
                height: Val::Px(RETRY_SIZE.y),
                ..Default::default()
            },
        )
    }
}

/// 構造:
/// * root
///   * board
///     * gameover text
///     * retry
///       * icon
fn setup(
    mut commands: Commands,
    asset_server: Res<AssetServer>,
) {
    let font = asset_server.load(PATH_FONT);
    let image = asset_server.load(PATH_IMAGE_RETRY);

    commands
        // ルートノードを生成
        .spawn(Gameover::from_root())
        .with_children(|parent| {
            // ボードノードを生成
            parent.spawn(Gameover::from_board())
                .with_children(|parent| {
                    // ゲームオーバーテキストノードを生成
                    parent.spawn(Gameover::from_text(font));
                })
                .with_children(|parent| {
                    // リトライボタンノードを生成
                    parent.spawn(Gameover::from_retry())
                        .with_children(|parent| {
                            // リトライアイコンノードを生成
                            parent.spawn(Gameover::from_retry_icon(image));
                        });
                });
        });
}

fn update(
    mut interaction_query: Query<
    (&Interaction, &mut BackgroundColor),
    (Changed<Interaction>, With<Button>),
    >,
    mut next_state: ResMut<NextState<AppState>>,
) {
    // 全てのインタラクション状態を持つボタンに対して処理を行う
    for (interaction, mut color) in &mut interaction_query {
        match *interaction {
            // ボタンが押された時の処理
            Interaction::Pressed => {
                next_state.set(AppState::InGame);
            }
            // ボタンがホバーされた時の処理
            Interaction::Hovered => {
                *color = RETRY_BACKGROUND_COLOR_HOVER.into();
            }
            // ボタンに何もされていない時の処理
            Interaction::None => {
                *color = BOARD_COLOR.into();
            }
        }
    }
}

fn despawn(
    mut commands: Commands,
    query: Query<Entity, With<Gameover>>,
) {
    for entity in &query {
        commands.entity(entity).despawn();
    }
}

pub struct GameoverPlugin;

impl Plugin for GameoverPlugin {
    fn build(&self, app: &mut App) {
        app
            .add_systems(OnEnter(AppState::Gameover), setup)
            .add_systems(Update, update.run_if(in_state(AppState::Gameover)))
            .add_systems(OnExit(AppState::Gameover), despawn)
        ;
    }
}
```

そして`src/main.rs`の以下のコードのコメントをはずします。

```rust
// ...

mod gameover;

// ...
fn main() {
    App::new()
        // ...
        .add_plugins(gameover::GameoverPlugin)
        // ...
}
```

では`cargo run`を実行して動作を確認してみましょう。
ゲームオーバーになったときに、ゲームオーバー画面が表示されて、
リトライボタンを押したときにゲームが再度プレイできたら成功です。

ここまでできたらゲームは完成です。お疲れ様でした！

## まとめ

いかがだったでしょうか？
巷ではテトリスの制作は簡単と言われていますが、Bevyで作った感想は非常に難しかったです。
特にテトリミノのような変形した形をBevyでは描画することができず、4つのブロックを1つのテトリミノとして定義し、操作する点など難しかったです。
これさえなければもっと少ない工数で実装できたのになぁと感じた次第です。

ソースコードは上記の「ソースコード」の箇所のURLから入手することが可能です。
まだこのプロジェクトは終わりではないので、これから機能をもっと実装していく予定です。
もしよろしければ、何か気になる点や、おかしな点などを見つけたら、
イシューを投げてもらっても全然OKです！

ここまでみていただきありがとうございました！
