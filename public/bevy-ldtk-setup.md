---
title: 【Bevyでゲーム作り】BevyとLDtkで2Dゲームを作る
tags:
  - Rust
  - Bevy
  - ldtk
private: false
updated_at: '2025-03-03T20:43:08+09:00'
id: 9378e8d5ba98e3322b59
organization_url_name: null
slide: false
ignorePublish: false
---
この記事ではRustで作られたゲームエンジン`Bevy`と、2Dレベルエディタである`LDtk`を組み合わせたゲームの作り方について解説しています。

![bevy-ldtk-setup](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/922938/741c5f51-e51e-4210-96b8-255536d591cb.gif)
## この記事は？

`bevy_ecs_ldtk`パッケージを使えばいい感じにBevyでゲームを作れることは書いてたが、
`Bevy`でどうやってゲームの制御面を作れば良いかのことは探してもなかったので、
自分で書いてみました。

## 参考URL

今回記事を書くのに参考にしたURLは以下の通り。

**ゲームエンジン**

https://bevyengine.org/

**レベルエディタ**

https://ldtk.io/

**ldtkクレート**

https://github.com/Trouv/bevy_ecs_ldtk

**Bevy(States)**

https://bevy-cheatbook.github.io/programming/states.html

**bevy_ecs_ldtkチュートリアル**

https://trouv.github.io/bevy_ecs_ldtk/v0.10.0/tutorials/tile-based-game/index.html

## ソースコード

ソースコードはGitHubに保存しています。

https://github.com/ittokunvim/bevy_ldtk_setup

## バージョン

バージョンが違うとおそらく動作しません。ご注意を。

```toml
bevy = "0.14.2"
bevy_ecs_ldtk = "0.10.0"
```

## ディレクトリ構造

最終的なディレクトリ構造、ファイルは以下の通り。

```
bevy_ldtk_setup
├── Cargo.lock
├── Cargo.toml
├── LICENSE
├── README.md
├── assets
│   ├── bevy_ldtk_setup.ldtk
│   ├── fonts
│   │   ├── FiraMono-Medium.ttf
│   │   └── FiraSans-Bold.ttf
│   └── images
│       ├── player.png
│       ├── thumbnail.png
│       └── tileset.png
└── src
    ├── gameover.rs
    ├── ingame.rs
    ├── main.rs
    └── mainmenu.rs
```

## ゲーム概要

迷路ゲームでプレイヤーを操作してゴールを目指すものとなっています。

ゲーム自体は以下の`bevy_ecs_ldtk`のチュートリアルをそのまま使用しています。

上記にURLを貼っているので`bevy_ecs_ldtk`の使い方などが学べるのでおすすめです。
ここでは本筋から外れるので説明はしません。

## 追加した制御面の仕組み

今回私が追加した制御面は、以下の通り。

- スタート画面：ゲーム起動時にタイトル名、画像、`click start ...`テキストを配置し、画面をクリックすることでゲームを開始することができる仕組み。
- ゲームクリア画面：プレイヤーがゴールに到達したらポップアップが出現し、`R`キーを押すことで1からゲームをプレイすることができる仕組み。

スタート画面は、ゲーム起動時にいきなりゲームが始まるのを防ぎます。
この機能実装の目的は、ユーザーにゲーム開始の権利を与えることにあります。

ゲームクリア画面は、ゲームクリアがクリアしたことをユーザーに知らせる役割と、ゲームがループできるようにする役割を持たせています。

これらの機能を実装するために、`bevy`のステート機能を利用しています。
ステート機能導入によってアプリ内の状態によって操作を変更することができるようになります。

詳しくは上記のURLをご覧ください。

この記事では上記の項目について解説していこうと思っています。

### main.rsにステートを追加

`main.rs`にステートを追加して、スタート画面、ゲームクリア画面を追加します。

ステートを追加することによって、ゲーム中、メインメニュー、ゲームクリアによって操作を帰ることができるようになります。

コードは以下の通り。

`src/main.rs`

```rust
use bevy::prelude::*;

mod mainmenu;
mod gameover;

use crate::mainmenu::{
    mainmenu_setup,
    mainmenu_update,
};

// ...

use crate::gameover::{
    gameover_setup,
    gameover_update,
};

const GAMETITLE: &str = "Bevy LDtk Setup";
const WINDOW_SIZE: Vec2 = Vec2::new(800.0, 800.0);
const BG_COLOR: Color = Color::srgb(0.255, 0.251, 0.333);

#[derive(Clone, Copy, Eq, PartialEq, Hash, Debug, Default, States)]
pub enum AppState {
    #[default]
    MainMenu,
    InGame,
    GameOver,
}

fn main() {
    App::new()
        .add_plugins(DefaultPlugins
            .set(WindowPlugin {
                primary_window: Some(Window {
                    resolution: WINDOW_SIZE.into(),
                    title: GAMETITLE.to_string(),
                    ..default()
                }),
                ..default()
            })
            .set(ImagePlugin::default_nearest())
        )
        .init_state::<AppState>()
        .insert_resource(ClearColor(BG_COLOR))
        .insert_resource(Time::<Fixed>::from_seconds(1.0 / 60.0))
        // ldtk setup
        // ...
        // mainmenu
        .add_systems(OnEnter(AppState::MainMenu), mainmenu_setup)
        .add_systems(Update, mainmenu_update.run_if(in_state(AppState::MainMenu)))
        // ingame
        .add_systems(OnEnter(AppState::InGame), ingame_setup)
        .add_systems(Update, (
            move_player_from_input,
            translate_grid_coords_entities,
            cache_wall_locations,
            check_goal,
            // update_ingame,
        ).run_if(in_state(AppState::InGame)))
        // gameover
        .add_systems(OnEnter(AppState::GameOver), gameover_setup)
        .add_systems(Update, gameover_update.run_if(in_state(AppState::GameOver)))
        .run();
}
```

### メインメニュー

以下のファイルではメインメニューのセットアップと、クリック時にゲームを開始できるようにコードが書かれています。

`src/mainmenu.rs`

```rust
use bevy::prelude::*;

use crate::{
    GAMETITLE,
    AppState,
};

const GAMETITLE_FONT_SIZE: f32 = 40.0;
const CLICKSTART_FONT_SIZE: f32 = 30.0;
const FONT_COLOR: Color = Color::srgb(0.9, 0.9, 0.9);

#[derive(Component)]
pub struct Mainmenu;

pub fn mainmenu_setup(
    mut commands: Commands,
    asset_server: Res<AssetServer>,
) {
    // Camera
    commands.spawn(Camera2dBundle::default());
    // Game title
    commands.spawn((
        TextBundle::from_section(
            GAMETITLE,
            TextStyle {
                font: asset_server.load("fonts/FiraSans-Bold.ttf"),
                font_size: GAMETITLE_FONT_SIZE,
                color: FONT_COLOR,
            },
        )
        .with_style(Style {
            position_type: PositionType::Relative,
            justify_self: JustifySelf::Center,
            ..default()
        }),
        Mainmenu,
    ));
    // Background image
    commands.spawn((
        SpriteBundle {
            texture: asset_server.load("images/thumbnail.png"),
            ..default()
        },
        Mainmenu,
    ));
    // Click start
    commands.spawn((
        TextBundle::from_section(
            "click start ...",
            TextStyle {
                font: asset_server.load("fonts/FiraMono-Medium.ttf"),
                font_size: CLICKSTART_FONT_SIZE,
                color: FONT_COLOR,
            },
        )
        .with_style(Style {
            position_type: PositionType::Absolute,
            right: Val::Px(16.0),
            bottom: Val::Px(16.0),
            ..default()
        }),
        Mainmenu,
    ));
}

pub fn mainmenu_update(
    mouse_event: Res<ButtonInput<MouseButton>>,
    mainmenu_query: Query<Entity, With<Mainmenu>>,
    mut commands: Commands,
    mut app_state: ResMut<NextState<AppState>>,
) {
    if mouse_event.just_pressed(MouseButton::Left) {
        // Despawned mainmenu
        for mainmenu_entity in mainmenu_query.iter() {
            commands.entity(mainmenu_entity).despawn();
        }
        // Changed app state
        app_state.set(AppState::InGame);
    }
}
```

### ゲームクリア、リスタート

以下のファイルではゲームクリア時のセットアップと、`R`キー押下時にメインメニューに戻る仕組みを実装しています。

`src/gameover.rs`

```rust
use bevy::prelude::*;
use bevy_ecs_ldtk::prelude::*;

use crate::{
    WINDOW_SIZE,
    AppState,
};

const GAMEOVER_FONT_SIZE: f32 = 40.0;
const FONT_COLOR: Color = Color::srgb(0.1, 0.1, 0.1);
const BG_COLOR: Color = Color::srgb(0.9, 0.9, 0.9);
const BG_SIZE: Vec2 = Vec2::new(160.0, 160.0);
const TEXT_GAP: f32 = 40.0;
const RESTART_FONT_SIZE: f32 = 30.0;

#[derive(Component)]
pub struct Gameover;

pub fn gameover_setup(
    mut commands: Commands,
    asset_server: Res<AssetServer>,
) {
    // Gameover
    commands.spawn((
        TextBundle::from_section(
            "Game Clear !!!",
            TextStyle {
                font: asset_server.load("fonts/FiraSans-Bold.ttf"),
                font_size: GAMEOVER_FONT_SIZE,
                color: FONT_COLOR,
            },
        )
        .with_style(Style {
            position_type: PositionType::Relative,
            top: Val::Px(WINDOW_SIZE.y / 2.0 - GAMEOVER_FONT_SIZE / 2.0 - TEXT_GAP),
            justify_self: JustifySelf::Center,
            ..default()
        }),
        Gameover,
    ));
    // Restart [R]
    commands.spawn((
        TextBundle::from_section(
            "Restart [R]",
            TextStyle {
                font: asset_server.load("fonts/FiraMono-Medium.ttf"),
                font_size: RESTART_FONT_SIZE,
                color: FONT_COLOR,
            },
        )
        .with_style(Style {
            position_type: PositionType::Relative,
            top: Val::Px(WINDOW_SIZE.y / 2.0 - RESTART_FONT_SIZE / 2.0 + TEXT_GAP),
            justify_self: JustifySelf::Center,
            ..default()
        }),
        Gameover,
    ));
    // Gameover background
    commands.spawn((
        SpriteBundle {
            sprite: Sprite {
                color: BG_COLOR,
                custom_size: Some(BG_SIZE),
                ..default()
            },
            transform: Transform {
                translation: Vec3::new(
                    WINDOW_SIZE.x / 4.0,
                    WINDOW_SIZE.y / 4.0,
                    10.0
                ),
                ..default()
            },
            ..default()
        },
        Gameover,
    ));
}

pub fn gameover_update(
    keyboard_input: Res<ButtonInput<KeyCode>>,
    gameover_query: Query<Entity, With<Gameover>>,
    level_selection: ResMut<LevelSelection>,
    mut commands: Commands,
    mut app_state: ResMut<NextState<AppState>>,
) {
    // R pressed
    if keyboard_input.just_pressed(KeyCode::KeyR) {
        // Despawned gameover entities
        for gameover_entity in gameover_query.iter() {
            commands.entity(gameover_entity).despawn();
        }
        // Reset ldtk level
        let indices = match level_selection.into_inner() {
            LevelSelection::Indices(indices) => indices,
            _ => panic!("level selection should always be Indices in this game"),
        };
        indices.level = 0;
        // Moved app state to ingame
        app_state.set(AppState::InGame);
    }
}
```

以下のファイルでは、`bevy_ldtk_setup`のチュートリアルコード内の`check_goal`システムに、
ステートに関するコードを追加して、
プレイヤーがゴールに到達したらゲームクリアに移行する処理を追加しています。

`src/ingame.rs`

```rust
pub fn check_goal(
    level_selection: ResMut<LevelSelection>,
    players: Query<&GridCoords, (With<Player>, Changed<GridCoords>)>,
    goals: Query<&GridCoords, With<Goal>>,
    mut app_state: ResMut<NextState<AppState>>,
) {
    if players
        .iter()
        .zip(goals.iter())
        .any(|(player_grid_coords, goal_grid_coords) | player_grid_coords == goal_grid_coords)
    {
        let indices = match level_selection.into_inner() {
            LevelSelection::Indices(indices) => indices,
            _ => panic!("level selection should always be Indices in this game"),
        };

        if indices.level < MAX_LEVEL_SELECTION - 1 {
            indices.level += 1;
        }
        else {
            app_state.set(AppState::GameOver);
        }
    }
}
```

## まとめ

`bevy_ecs_ldtk`のチュートリアルゲームに、メインメニュー、ゲームクリア機能を追加して、ゲームっぽくしてみましたがいかがだったでしょうか？

`bevy`のメインメニュー、ゲームクリアの実装方法がどこにもなかったので今回記事にしてみました。

この記事が何かの役に立てたら嬉しいです。

