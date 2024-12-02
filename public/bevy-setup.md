---
title: ゲームエンジンBevyのセットアップ方法
tags:
  - Rust
  - Bevy
private: false
updated_at: '2024-12-03T03:30:07+09:00'
id: dae4b690a7d3b55368ac
organization_url_name: null
slide: false
ignorePublish: false
---
この記事ではゲームエンジン`Bevy`のセットアップ方法について書かれています。

実装したコードは以下のURLから見ることもできます。

https://github.com/ittokunvim/bevy-setup

## バージョン

`Bevy`のバージョンは`0.14.2`を使用しています。

このバージョン以外で実装を行うと動作しない可能性があります。

```toml
bevy = "0.14.2"
```

## セットアップ

[リポジトリ](https://github.com/ittokunvim/bevy-setup)をクローンしてから、`cargo run`を実行することで遊ぶことができます。

<img
  src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/922938/f4d01798-9e29-ffaf-bfaa-793e79a0c815.gif"
  alt="bevy-setup"
  width="480"
/>

## 実装内容

このプロジェクトは主にゲームステートの遷移についての記述がされています。

ゲームステートは`Mainmenu, Ingame, Pause, Gameover, Gameclear`の5つで、これらのステートを行き来するコードが書かれています。

各ステートの遷移方法は以下の通り。

### Mainmenu

ゲームを起動した際に初めに遷移するステート。

画面をクリックすることで`Ingame`ステートに遷移します。

### Ingame

ゲームを遊ぶためのステート。

画面左下のポーズボタンをクリックすると`Pause`ステートに遷移します。

キー`A`を押すことで`Gameover`ステートに遷移します。

キー`D`を押すことで`Gameclear`ステートに遷移します。

また10秒間経過で`Gameover`ステートに遷移します。

### Pause

ゲームを一時停止した時に遷移するステート。

画面左下のポーズボタンをクリックすると`Ingame`ステートに遷移します。

### Gameover

ゲームにクリアできなかった時に遷移するステート。

キー`R`を押すことで`Ingame`ステートに遷移します。

キー`B`を押すことで`Mainmenu`ステートに遷移します。

### Gameclear

ゲームをクリアした時に遷移するステート。

キー`R`を押すことで`Ingame`ステートに遷移します。

キー`B`を押すことで`Mainmenu`ステートに遷移します。

## ファイル構造

ファイル構造は以下の通り。

```
bevy-setup
├── Cargo.lock
├── Cargo.toml
├── README.md
├── assets
│   ├── fonts
│   │   └── misaki_gothic.ttf
│   ├── images
│   │   ├── mainmenu.png
│   │   └── pausebutton.png
│   └── sounds
│       └── bgm.ogg
└── src
    ├── gameclear.rs
    ├── gameover.rs
    ├── ingame
    │   ├── key.rs
    │   ├── mod.rs
    │   ├── pausebutton.rs
    │   ├── scoreboard.rs
    │   ├── text.rs
    │   └── timer.rs
    ├── main.rs
    └── mainmenu.rs
```

各ファイルの記述内容は以下のとおり。

| ファイル名              | 記述内容                                       |
| ----------------------- | ---------------------------------------------- |
| `main.rs`               | プロジェクトの環境変数やセットアップコードなど |
| `mainmenu.rs`           | メインメニューのセットアップ、処理など         |
| `gameover.rs`           | ゲームオーバーのセットアップ、処理など         |
| `gameclear.rs`          | ゲームクリアのセットアップ、処理など           |
| `ingame/mod.rs`         | ゲーム内の各パーツをまとめているファイル       |
| `ingame/key.rs`         | ゲーム内のキーボード処理                       |
| `ingame/pausebutton.rs` | ゲーム内のポーズボタンのセットアップ、処理     |
| `ingame/scoreboard.rs`  | ゲーム内のスコアボードのセットアップ、処理     |
| `ingame/text.rs`        | ゲーム内のテキストのセットアップ、処理         |
| `ingame/timer.rs`       | ゲーム内のタイマーなセットアップ、処理         |

## 各ファイルの記述と処理内容

### main.rs

ここには環境変数、定数、ステート、全ステート共通のセットアップなどが書かれています。

```rust:main.rs
use bevy::prelude::*;

mod mainmenu;
mod ingame;
mod gameover;
mod gameclear;

const GAMETITLE: &str = "Bevyセットアップ";
const WINDOW_SIZE: Vec2 = Vec2::new(640.0, 480.0);
const BACKGROUND_COLOR: Color = Color::srgb(0.9, 0.9, 0.9);
const CURSOR_RANGE: f32 = 10.0;
const GAMETIME_LIMIT: f32 = 10.0;
const PATH_FONT: &str = "fonts/misaki_gothic.ttf";
const PATH_IMAGE_MAINMENU: &str = "images/mainmenu.png";
const PATH_IMAGE_PAUSEBUTTON: &str = "images/pausebutton.png";
const PATH_SOUND_BGM: &str = "sounds/bgm.ogg";

#[derive(States, Default, Debug, Clone, PartialEq, Eq, Hash)]
enum AppState {
    #[default]
    Mainmenu,
    Ingame,
    Pause,
    Gameover,
    Gameclear,
}

#[derive(Resource, Deref, DerefMut, Debug)]
struct Config {
    setup_ingame: bool,
}

#[derive(Resource, Deref, DerefMut, Debug)]
struct Score(pub usize);

#[derive(Resource)]
struct GameTimer(Timer);

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
        .insert_resource(Config { setup_ingame: true })
        .insert_resource(Score(0))
        .insert_resource(GameTimer(
            Timer::from_seconds(GAMETIME_LIMIT, TimerMode::Once)
        ))
        .add_systems(Startup, setup)
        .add_plugins(mainmenu::MainmenuPlugin)
        .add_plugins(ingame::IngamePlugin)
        .add_plugins(gameover::GameoverPlugin)
        .add_plugins(gameclear::GameclearPlugin)
        .run();
}

fn setup(
    mut commands: Commands,
    asset_server: Res<AssetServer>,
) {
    println!("main: setup");
    // camera
    commands.spawn(Camera2dBundle::default());
    // bgm
    let bgm_sound = asset_server.load(PATH_SOUND_BGM);

    commands.spawn(
        AudioBundle {
            source: bgm_sound,
            settings: PlaybackSettings::LOOP.with_spatial(true),
        }
    )
    .insert(Name::new("bgm"));
}
```

### mainmenu.rs

ここにはメインメニューのゲームタイトル、クリックスタート、背景画像のセットアップや、
ステート遷移させるコードなどが書かれています。

```rust:mainmenu.rs
use bevy::{
    prelude::*,
    sprite::{MaterialMesh2dBundle, Mesh2dHandle},
};

use crate::{
    GAMETITLE,
    WINDOW_SIZE,
    PATH_FONT,
    PATH_IMAGE_MAINMENU,
    AppState,
};

const GAMETITLE_FONT_SIZE: f32 = 32.0;
const GAMETITLE_COLOR: Color = Color::srgb(0.1, 0.1, 0.1);
const CLICKSTART_TEXT: &str = "クリックしてスタート";
const CLICKSTART_FONT_SIZE: f32 = 20.0;
const CLICKSTART_COLOR: Color = Color::srgb(0.2, 0.2, 0.2);
const TEXT_PADDING: f32 = 40.0;
const BOARD_SIZE: Vec2 = Vec2::new(280.0, 210.0);
const BOARD_COLOR: Color = Color::srgb(0.9, 0.9, 0.9);

#[derive(Component)]
struct Mainmenu;

fn setup(
    mut commands: Commands,
    asset_server: Res<AssetServer>,
    mut meshes: ResMut<Assets<Mesh>>,
    mut materials: ResMut<Assets<ColorMaterial>>,
) {
    println!("mainmenu: setup");
    // game title
    let top = Val::Px(WINDOW_SIZE.y / 2.0 - GAMETITLE_FONT_SIZE / 2.0 - TEXT_PADDING);

    commands.spawn((
        TextBundle::from_section(
            GAMETITLE,
            TextStyle {
                font: asset_server.load(PATH_FONT),
                font_size: GAMETITLE_FONT_SIZE,
                color: GAMETITLE_COLOR,
            }
        )
        .with_style(Style {
            position_type: PositionType::Relative,
            justify_self: JustifySelf::Center,
            top,
            ..Default::default()
        }),
        Mainmenu,
    ))
    .insert(Name::new("gametitle"));
    // click start
    let top = Val::Px(WINDOW_SIZE.y / 2.0 - CLICKSTART_FONT_SIZE / 2.0 + TEXT_PADDING);

    commands.spawn((
        TextBundle::from_section(
            CLICKSTART_TEXT, 
            TextStyle {
                font: asset_server.load(PATH_FONT),
                font_size: CLICKSTART_FONT_SIZE,
                color: CLICKSTART_COLOR,
            }
        )
        .with_style(Style {
            position_type: PositionType::Relative,
            justify_self: JustifySelf::Center,
            top,
            ..Default::default()
        }),
        Mainmenu,
    ))
    .insert(Name::new("clickstart"));
    // board
    commands.spawn((
        MaterialMesh2dBundle {
            mesh: Mesh2dHandle(meshes.add(Rectangle::new(BOARD_SIZE.x, BOARD_SIZE.y))),
            material: materials.add(BOARD_COLOR),
            ..Default::default()
        },
        Mainmenu,
    ))
    .insert(Name::new("board"));
    // image
    commands.spawn((
        SpriteBundle {
            texture: asset_server.load(PATH_IMAGE_MAINMENU),
            transform: Transform::from_xyz(0.0, 0.0, -10.0),
            ..default()
        },
        Mainmenu,
    ))
    .insert(Name::new("image"));
}

fn update(
    mouse_event: Res<ButtonInput<MouseButton>>,
    mainmenu_query: Query<Entity, With<Mainmenu>>,
    mut commands: Commands,
    mut next_state: ResMut<NextState<AppState>>,
) {
    if mouse_event.just_pressed(MouseButton::Left) {
        println!("mainmenu: clicked");
        println!("mainmenu: despawned");
        for mainmenu_entity in mainmenu_query.iter() {
            commands.entity(mainmenu_entity).despawn();
        }
        println!("mainmenu: moved state to Ingame from Mainmenu");
        next_state.set(AppState::Ingame);
    }
}

pub struct MainmenuPlugin;

impl Plugin for MainmenuPlugin {
    fn build(&self, app: &mut App) {
        app
            .add_systems(OnEnter(AppState::Mainmenu), setup)
            .add_systems(Update, update.run_if(in_state(AppState::Mainmenu)));
    }
}
```

### gameover.rs

ここにはゲームオーバーのテキスト、スコア、リトライ、タイトルに戻るセットアップや、
ステートの遷移するコードが書かれています。

```rust:gameover.rs
use bevy::prelude::*;

use crate::{
    WINDOW_SIZE,
    PATH_FONT,
    AppState,
    Config,
    Score,
};

const GAMEOVER_TEXT: &str = "ゲームオーバー";
const GAMEOVER_FONT_SIZE: f32 = 32.0;
const SCORE_TEXT: &str = "スコア: ";
const RETRY_TEXT: &str = "リトライ: Key[R]";
const BACKTOTITLE_TEXT: &str = "タイトルに戻る: Key[B]";
const TEXT_COLOR: Color = Color::srgb(0.1, 0.1, 0.1);
const TEXT_FONT_SIZE: f32 = 20.0;
const TEXT_PADDING: f32 = 40.0;

#[derive(Component)]
struct Gameover;

fn setup(
    mut commands: Commands,
    asset_server: Res<AssetServer>,
    score: Res<Score>,
) {
    println!("gameover: setup");
    // gameover
    let top = Val::Px(WINDOW_SIZE.y / 2.0 - GAMEOVER_FONT_SIZE / 2.0 - TEXT_PADDING * 1.5);

    commands.spawn((
        TextBundle::from_section(
            GAMEOVER_TEXT, 
            TextStyle {
                font: asset_server.load(PATH_FONT),
                font_size: GAMEOVER_FONT_SIZE,
                color: TEXT_COLOR,
            }
        )
        .with_style(Style {
                position_type: PositionType::Relative,
                justify_self: JustifySelf::Center,
                top,
                ..Default::default()
            }),
        Gameover,
    ))
    .insert(Name::new("gameover"));
    // score
    let top = Val::Px(WINDOW_SIZE.y / 2.0 - TEXT_FONT_SIZE / 2.0 - TEXT_PADDING * 0.5);

    commands.spawn((
        TextBundle::from_section(
            format!("{}{}", SCORE_TEXT, **score), 
            TextStyle {
                font: asset_server.load(PATH_FONT),
                font_size: TEXT_FONT_SIZE,
                color: TEXT_COLOR,
            }
        )
        .with_style(Style {
            position_type: PositionType::Relative,
            justify_self: JustifySelf::Center,
            top,
            ..Default::default()
        }),
        Gameover,
    ))
    .insert(Name::new("score"));
    // retry
    let top = Val::Px(WINDOW_SIZE.y / 2.0 - TEXT_FONT_SIZE / 2.0 + TEXT_PADDING * 0.5);

    commands.spawn((
        TextBundle::from_section(
            RETRY_TEXT, 
            TextStyle {
                font: asset_server.load(PATH_FONT),
                font_size: TEXT_FONT_SIZE,
                color: TEXT_COLOR,
            }
        )
        .with_style(Style {
            position_type: PositionType::Relative,
            justify_self: JustifySelf::Center,
            top,
            ..Default::default()
        }),
        Gameover,
    ))
    .insert(Name::new("retry"));
    // back to title
    let top = Val::Px(WINDOW_SIZE.y / 2.0 - TEXT_FONT_SIZE / 2.0 + TEXT_PADDING * 1.5);

    commands.spawn((
        TextBundle::from_section(
            BACKTOTITLE_TEXT, 
            TextStyle {
                font: asset_server.load(PATH_FONT),
                font_size: TEXT_FONT_SIZE,
                color: TEXT_COLOR,
            }
        )
        .with_style(Style {
            position_type: PositionType::Relative,
            justify_self: JustifySelf::Center,
            top,
            ..Default::default()
        }),
        Gameover,
    ))
    .insert(Name::new("backtotitle"));
}

fn update(
    keyboard_input: Res<ButtonInput<KeyCode>>,
    mut config: ResMut<Config>,
    mut next_state: ResMut<NextState<AppState>>,
) {
    let mut closure = |key: &KeyCode, app_state: AppState| {
        println!("gameover: {:?} just pressed", key);
        println!("gameover: config setup ingame is true");
        config.setup_ingame = true;
        println!("gameover: moved state to {:?} from Gameover", app_state);
        next_state.set(app_state);
    };

    for key in keyboard_input.get_just_pressed() {
        match key {
            KeyCode::KeyR => closure(key, AppState::Ingame),
            KeyCode::KeyB => closure(key, AppState::Mainmenu),
            _ => {},
        }
    }
}

fn despawn_gameover(
    mut commands: Commands,
    query: Query<Entity, With<Gameover>>,
) {
    println!("gameover: despawned");
    for entity in query.iter() {
        commands.entity(entity).despawn();
    }
}

pub struct GameoverPlugin;

impl Plugin for GameoverPlugin {
    fn build(&self, app: &mut App) {
        app
            .add_systems(OnEnter(AppState::Gameover), setup)
            .add_systems(Update, update.run_if(in_state(AppState::Gameover)))
            .add_systems(OnExit(AppState::Gameover), despawn_gameover);
    }
}
```

### gameclear.rs

ここにはゲームオーバーの処理とほぼ同じコードが書かれています。

```rust:gameclear.rs
use bevy::prelude::*;

use crate::{
    WINDOW_SIZE,
    PATH_FONT,
    AppState,
    Config,
    Score,
    GameTimer,
};

const GAMECLEAR_TEXT: &str = "ゲームクリア";
const GAMECLEAR_FONT_SIZE: f32 = 32.0;
const SCORE_TEXT: &str = "スコア: ";
const TIMER_TEXT: &str = "タイム: ";
const RETRY_TEXT: &str = "リトライ: Key[R]";
const BACKTOTITLE_TEXT: &str = "タイトルに戻る: Key[B]";
const TEXT_COLOR: Color = Color::srgb(0.1, 0.1, 0.1);
const TEXT_FONT_SIZE: f32 = 20.0;
const TEXT_PADDING: f32 = 40.0;

#[derive(Component)]
struct Gameclear;

fn setup(
    mut commands: Commands,
    asset_server: Res<AssetServer>,
    score: Res<Score>,
    timer: Res<GameTimer>,
) {
    println!("gameclear: setup");
    // gameover
    let top = WINDOW_SIZE.y / 2.0 - GAMECLEAR_FONT_SIZE / 2.0 - TEXT_PADDING * 2.0;

    commands.spawn((
        TextBundle::from_section(
            GAMECLEAR_TEXT, 
            TextStyle {
                font: asset_server.load(PATH_FONT),
                font_size: GAMECLEAR_FONT_SIZE,
                color: TEXT_COLOR,
            }
        )
        .with_style(Style {
            position_type: PositionType::Relative,
            justify_self: JustifySelf::Center,
            top: Val::Px(top),
            ..Default::default()
        }),
        Gameclear,
    ))
    .insert(Name::new("gameclear"));
    // score
    let top = WINDOW_SIZE.y / 2.0 - TEXT_FONT_SIZE / 2.0 - TEXT_PADDING;

    commands.spawn((
        TextBundle::from_section(
            format!("{}{}", SCORE_TEXT, **score), 
            TextStyle {
                font: asset_server.load(PATH_FONT),
                font_size: TEXT_FONT_SIZE,
                color: TEXT_COLOR,
            }
        )
        .with_style(Style {
            position_type: PositionType::Relative,
            justify_self: JustifySelf::Center,
            top: Val::Px(top),
            ..Default::default()
        }),
        Gameclear,
    ))
    .insert(Name::new("score"));
    // timer
    let top = WINDOW_SIZE.y / 2.0 - TEXT_FONT_SIZE / 2.0;

    commands.spawn((
        TextBundle::from_section(
            format!("{}{}", TIMER_TEXT, timer.0.remaining_secs().round().to_string()),
            TextStyle {
                font: asset_server.load(PATH_FONT),
                font_size: TEXT_FONT_SIZE,
                color: TEXT_COLOR,
            }
        )
        .with_style(Style {
            position_type: PositionType::Relative,
            justify_self: JustifySelf::Center,
            top: Val::Px(top),
            ..Default::default()
        }),
        Gameclear,
    ))
    .insert(Name::new("timer"));
    // retry
    let top = WINDOW_SIZE.y / 2.0 - TEXT_FONT_SIZE / 2.0 + TEXT_PADDING;

    commands.spawn((
        TextBundle::from_section(
            RETRY_TEXT, 
            TextStyle {
                font: asset_server.load(PATH_FONT),
                font_size: TEXT_FONT_SIZE,
                color: TEXT_COLOR,
            }
        )
        .with_style(Style {
            position_type: PositionType::Relative,
            justify_self: JustifySelf::Center,
            top: Val::Px(top),
            ..Default::default()
        }),
        Gameclear,
    ))
    .insert(Name::new("retry"));
    // back to title
    let top = WINDOW_SIZE.y / 2.0 - TEXT_FONT_SIZE / 2.0 + TEXT_PADDING * 2.0;

    commands.spawn((
        TextBundle::from_section(
            BACKTOTITLE_TEXT, 
            TextStyle {
                font: asset_server.load(PATH_FONT),
                font_size: TEXT_FONT_SIZE,
                color: TEXT_COLOR,
            }
        )
        .with_style(Style {
            position_type: PositionType::Relative,
            justify_self: JustifySelf::Center,
            top: Val::Px(top),
            ..Default::default()
        }),
        Gameclear,
    ))
    .insert(Name::new("backtotitle"));
}

fn update(
    keyboard_input: Res<ButtonInput<KeyCode>>,
    mut config: ResMut<Config>,
    mut next_state: ResMut<NextState<AppState>>,
) {
    let mut closure = |key: &KeyCode, app_state: AppState| {
        println!("gameclear: {:?} just pressed", key);
        println!("gameclear: config setup ingame is true");
        config.setup_ingame = true;
        println!("gameclear: moved state to {:?} from Gameclear", app_state);
        next_state.set(app_state);
    };

    for key in keyboard_input.get_just_pressed() {
        match key {
            KeyCode::KeyR => closure(key, AppState::Ingame),
            KeyCode::KeyB => closure(key, AppState::Mainmenu),
            _ => {},
        }
    }
}

fn despawn_gameclear(
    mut commands: Commands,
    query: Query<Entity, With<Gameclear>>,
) {
    println!("gameclear: despawned");
    for entity in query.iter() {
        commands.entity(entity).despawn();
    }
}

pub struct GameclearPlugin;

impl Plugin for GameclearPlugin {
    fn build(&self, app: &mut App) {
        app
            .add_systems(OnEnter(AppState::Gameclear), setup)
            .add_systems(Update, update.run_if(in_state(AppState::Gameclear)))
            .add_systems(OnExit(AppState::Gameclear), despawn_gameclear);
    }
}
```

### ingame/mod.rs

ここにはゲームの各パーツをまとめるコードが書かれています。

```rust:ingame/mod.rs
use bevy::prelude::*;

mod key;
mod pausebutton;
mod scoreboard;
mod text;
mod timer;

pub struct IngamePlugin;

impl Plugin for IngamePlugin {
    fn build(&self, app: &mut App) {
        app
            .add_plugins(key::KeyPlugin)
            .add_plugins(pausebutton::PausebuttonPlugin)
            .add_plugins(scoreboard::ScoreBoardPlugin)
            .add_plugins(text::TextPlugin)
            .add_plugins(timer::TimerPlugin);
    }
}
```

### ingame/key.rs

ここにはキーボード操作でステートを遷移先を決めるようなコードが書かれています。

```rust:ingame/key.rs
use bevy::prelude::*;

use crate::AppState;

fn update(
    keyboard_input: Res<ButtonInput<KeyCode>>,
    mut next_state: ResMut<NextState<AppState>>,
) {
    let mut closure = |key: &KeyCode, app_state: AppState| {
        println!("key: {:?} just pressed", key);
        println!("key: moved state to {:?} from Ingame", app_state);
        next_state.set(app_state);
    };

    for key in keyboard_input.get_just_pressed() {
        match key {
            KeyCode::KeyA => closure(key, AppState::Gameover),
            KeyCode::KeyD => closure(key, AppState::Gameclear),
            _ => {},
        }
    }
}

pub struct KeyPlugin;

impl Plugin for KeyPlugin {
    fn build(&self, app: &mut App) {
        app
            .add_systems(Update, update.run_if(in_state(AppState::Ingame)));
    }
}
```

### ingame/pausebutton.rs

ここにはポーズボタンのセットアップや、
クリックした時のトグル、
ステートの遷移、
遷移した時のコンポーネントの削除するコードなどが書かれています。

```rust:ingame/pausebutton.rs
use bevy::{
    prelude::*,
    window::PrimaryWindow,
};

use crate::{
    WINDOW_SIZE,
    CURSOR_RANGE,
    PATH_IMAGE_PAUSEBUTTON,
    AppState,
    Config,
};

const IMAGE_SIZE: u32 = 64;
const SIZE: f32 = 32.0;

#[derive(Component)]
pub struct PauseButton {
    first: usize,
    last: usize,
}

fn setup(
    mut commands: Commands,
    asset_server: Res<AssetServer>,
    mut texture_atlas_layouts: ResMut<Assets<TextureAtlasLayout>>,
    config: Res<Config>,
) {
    if !config.setup_ingame { return }

    println!("pausebutton: setup");
    let layout = TextureAtlasLayout::from_grid(UVec2::splat(IMAGE_SIZE), 2, 1, None, None);
    let texture_atlas_layout = texture_atlas_layouts.add(layout);
    let animation_indices = PauseButton { first: 0, last: 1, };
    let pos = Vec3::new(
        WINDOW_SIZE.x / 2.0 - SIZE,
        -WINDOW_SIZE.y / 2.0 + SIZE,
        10.0
    );

    commands.spawn((
        SpriteBundle {
            sprite: Sprite {
                custom_size: Some(Vec2::splat(SIZE)),
                ..Default::default()
            },
            texture: asset_server.load(PATH_IMAGE_PAUSEBUTTON),
            transform: Transform::from_translation(pos),
            ..Default::default()
        },
        TextureAtlas {
            layout: texture_atlas_layout,
            index: animation_indices.first,
        },
        animation_indices,
    ))
    .insert(Name::new("pausebutton"));
}

fn update(
    mouse_event: Res<ButtonInput<MouseButton>>,
    window_query: Query<&Window, With<PrimaryWindow>>,
    mut query: Query<(&Transform, &PauseButton, &mut TextureAtlas), With<PauseButton>>,
    mut config: ResMut<Config>,
    mut next_state: ResMut<NextState<AppState>>,
) {
    if !mouse_event.just_pressed(MouseButton::Left) { return; }

    let window = window_query.single();
    let mut cursor_pos = window.cursor_position().unwrap();
    let Ok((transform, prop, mut atlas)) = query.get_single_mut() else { return; };
    let pausebutton_pos = transform.translation.truncate();
    // get cursor position
    cursor_pos = Vec2::new(
        cursor_pos.x - WINDOW_SIZE.x / 2.0,
        -cursor_pos.y + WINDOW_SIZE.y / 2.0
    );

    let distance = cursor_pos.distance(pausebutton_pos);

    if distance < SIZE - CURSOR_RANGE {
        println!("pausebutton: clicked");
        if atlas.index == prop.first {
            println!("pausebutton: toggled");
            atlas.index = prop.last;
            println!("pausebutton: moved state to Pause from Ingame");
            next_state.set(AppState::Pause);
        } else {
            println!("pausebutton: change config.setup_ingame to false");
            config.setup_ingame = false;
            println!("pausebutton: toggled");
            atlas.index = prop.first;
            println!("pausebutton: moved state to Ingame from Pause");
            next_state.set(AppState::Ingame);
        }
    }
}

fn despawn_pausebutton(
    mut commands: Commands,
    query: Query<Entity, With<PauseButton>>,
) {
    let entity = query.single();
    println!("pausebutton: despawned");
    commands.entity(entity).despawn();
}

pub struct PausebuttonPlugin;

impl Plugin for PausebuttonPlugin {
    fn build(&self, app: &mut App) {
        app
            .add_systems(OnEnter(AppState::Ingame), setup)
            .add_systems(Update, update.run_if(in_state(AppState::Ingame)))
            .add_systems(Update, update.run_if(in_state(AppState::Pause)))
            .add_systems(OnEnter(AppState::Gameover), despawn_pausebutton)
            .add_systems(OnEnter(AppState::Gameclear), despawn_pausebutton);
    }
}
```

### ingame/scoreboard.rs

ここにはスコア、タイマーが書かれたスコアボードのセットアップ、
値の更新、
ステート遷移した時のコンポーネントの削除するコードなどが書かれています。

```rust:scoreboard.rs
use bevy::prelude::*;

use crate::{
    PATH_FONT,
    AppState,
    Config,
    Score,
    GameTimer,
};

const SCORE_TEXT: &str = "スコア: ";
const TIMER_TEXT: &str = " | タイム: ";
const TEXT_FONT_SIZE: f32 = 20.0;
const TEXT_COLOR: Color = Color::srgb(0.1, 0.1, 0.1);
const TEXT_PADDING: Val = Val::Px(5.0);

#[derive(Component)]
struct ScoreboardUi;

fn setup(
    mut commands: Commands,
    asset_server: Res<AssetServer>,
    config: Res<Config>,
) {
    if !config.setup_ingame { return }

    commands.spawn((
        TextBundle::from_sections([
            TextSection::new(
                SCORE_TEXT,
                TextStyle {
                    font: asset_server.load(PATH_FONT),
                    font_size: TEXT_FONT_SIZE,
                    color: TEXT_COLOR,
                },
            ),
            TextSection::from_style(TextStyle {
                font: asset_server.load(PATH_FONT),
                font_size: TEXT_FONT_SIZE,
                color: TEXT_COLOR,
            }),
            TextSection::new(
                TIMER_TEXT,
                TextStyle {
                    font: asset_server.load(PATH_FONT),
                    font_size: TEXT_FONT_SIZE,
                    color: TEXT_COLOR,
                },
            ),
            TextSection::from_style(TextStyle {
                font: asset_server.load(PATH_FONT),
                font_size: TEXT_FONT_SIZE,
                color: TEXT_COLOR,
            }),
        ])
        .with_style(Style {
            position_type: PositionType::Absolute,
            top: TEXT_PADDING,
            left: TEXT_PADDING,
            ..Default::default()
        }),
        ScoreboardUi,
    ));
}

fn update(
    mut query: Query<&mut Text, With<ScoreboardUi>>,
    score: Res<Score>,
    timer: Res<GameTimer>,
) {
    let mut text = query.single_mut();
    // write score and timer
    text.sections[1].value = score.to_string();
    text.sections[3].value = timer.0.remaining_secs().round().to_string();
}

fn despawn_scoreboard(
    mut commands: Commands,
    query: Query<Entity, With<ScoreboardUi>>,
) {
    let entity = query.single();
    println!("scoreboard: despawned");
    commands.entity(entity).despawn();
}

fn score_points(mut score: ResMut<Score>) {
    if **score >= 500 { **score = 0; }

    **score += 1;
}

fn reset_score(mut score: ResMut<Score>) {
    **score = 0;
}

pub struct ScoreBoardPlugin;

impl Plugin for ScoreBoardPlugin {
    fn build(&self, app: &mut App) {
        app
            .add_systems(OnEnter(AppState::Ingame), setup)
            .add_systems(Update, update.run_if(in_state(AppState::Ingame)))
            .add_systems(Update, score_points.run_if(in_state(AppState::Ingame)))
            .add_systems(OnEnter(AppState::Gameover), despawn_scoreboard)
            .add_systems(OnEnter(AppState::Gameclear), despawn_scoreboard)
            .add_systems(OnExit(AppState::Gameover), reset_score)
            .add_systems(OnExit(AppState::Gameclear), reset_score);
    }
}
```

### ingame/text.rs

ここにはゲーム内に表示するテキストのセットアップ、
ステート遷移した時のコンポーネントを削除するコードなどが書かれています。

```rust:text.rs
use bevy::prelude::*;

use crate::{
    WINDOW_SIZE,
    PATH_FONT,
    AppState,
    Config,
};

const INGAME_TEXT: &str = "ゲーム中";
const INGAME_FONT_SIZE: f32 = 32.0;
const GAMEOVER_TEXT: &str = "ゲームオーバー: Key[A]";
const GAMECLEAR_TEXT: &str = "ゲームクリア: Key[D]";
const TEXT_COLOR: Color = Color::srgb(0.1, 0.1, 0.1);
const TEXT_FONT_SIZE: f32 = 20.0;
const TEXT_PADDING: f32 = 40.0;

#[derive(Component)]
struct IngameText;

fn setup(
    mut commands: Commands,
    asset_server: Res<AssetServer>,
    config: Res<Config>,
) {
    if !config.setup_ingame { return }

    println!("text: setup");
    // ingame
    commands.spawn((
        TextBundle::from_section(
            INGAME_TEXT,
            TextStyle {
                font: asset_server.load(PATH_FONT),
                font_size: INGAME_FONT_SIZE,
                color: TEXT_COLOR,
            }
        )
        .with_style(Style {
            position_type: PositionType::Relative,
            justify_self: JustifySelf::Center,
            top: Val::Px(WINDOW_SIZE.y / 2.0 - INGAME_FONT_SIZE / 2.0 - TEXT_PADDING),
            ..Default::default()
        }),
        IngameText,
    ))
    .insert(Name::new("ingame"));
    // game over
    commands.spawn((
        TextBundle::from_section(
            GAMEOVER_TEXT,
            TextStyle {
                font: asset_server.load(PATH_FONT),
                font_size: TEXT_FONT_SIZE,
                color: TEXT_COLOR,
            }
        )
        .with_style(Style {
            position_type: PositionType::Relative,
            justify_self: JustifySelf::Center,
            top: Val::Px(WINDOW_SIZE.y / 2.0 - TEXT_FONT_SIZE / 2.0),
            ..Default::default()
        }),
        IngameText,
    ))
    .insert(Name::new("gameover"));
    // game clear
    commands.spawn((
        TextBundle::from_section(
            GAMECLEAR_TEXT,
            TextStyle {
                font: asset_server.load(PATH_FONT),
                font_size: TEXT_FONT_SIZE,
                color: TEXT_COLOR,
            }
        )
        .with_style(Style {
            position_type: PositionType::Relative,
            justify_self: JustifySelf::Center,
            top: Val::Px(WINDOW_SIZE.y / 2.0 - INGAME_FONT_SIZE / 2.0 + TEXT_PADDING),
            ..Default::default()
        }),
        IngameText,
    ))
    .insert(Name::new("gameclear"));
}

fn despawn_text(
    mut commands: Commands,
    query: Query<Entity, With<IngameText>>,
) {
    println!("text: despawned");
    for entity in query.iter() {
        commands.entity(entity).despawn();
    }
}

pub struct TextPlugin;

impl Plugin for TextPlugin {
    fn build(&self, app: &mut App) {
        app
            .add_systems(OnEnter(AppState::Ingame), setup)
            .add_systems(OnEnter(AppState::Gameover), despawn_text)
            .add_systems(OnEnter(AppState::Gameclear), despawn_text);
    }
}
```

### ingame/timer.rs

ここにはタイマーが0になった時のステート遷移や、
タイマーをリセットするコードなどが書かれています。

```rust:timer.rs
use bevy::prelude::*;

use crate::{
    AppState,
    GameTimer,
};

fn update(
    mut timer: ResMut<GameTimer>,
    time: Res<Time>,
    mut next_state: ResMut<NextState<AppState>>,
) {
    if timer.0.tick(time.delta()).just_finished() {
        println!("timer: moved state to Gameover from Ingame");
        next_state.set(AppState::Gameover);
    }
}

fn reset_timer(mut timer: ResMut<GameTimer>) {
    println!("timer: reset");
    timer.0.reset();
}

pub struct TimerPlugin;

impl Plugin for TimerPlugin {
    fn build(&self, app: &mut App) {
        app
            .add_systems(Update, update.run_if(in_state(AppState::Ingame)))
            .add_systems(OnExit(AppState::Gameover), reset_timer)
            .add_systems(OnExit(AppState::Gameclear), reset_timer);
    }
}
```

## 使用アセット

以下はBevyでゲーム開発をする時に重宝しているツールやサイトです。

どれもこれも素晴らしいのでゲーム開発におすすめです!!!

### リスト

Bevy(ゲームエンジン)

https://bevyengine.org

Sunny Land(タイトル画像)

https://ansimuz.itch.io/sunny-land-pixel-game-art

美咲フォント

https://littlelimit.net/misaki.htm

ICOOON MONO(ポーズボタン画像)

https://icooon-mono.com/

効果音ラボ(BGM)

https://soundeffect-lab.info

Pixlr(画像編集)

https://pixlr.com

## まとめ

この記事はこれで以上になります。
もしBevyでゲームを開発するときにこの記事が何かの参考になれば筆者も嬉しく思います。

ここまで読んでくれてありがとう!!!
