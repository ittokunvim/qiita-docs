---
title: Bevyでタイミングゲームを作る
tags:
  - game
  - Rust
  - Bevy
private: false
updated_at: '2025-03-14T00:03:08+09:00'
id: a598d512a2ec817eb125
organization_url_name: null
slide: false
ignorePublish: false
---

この記事では、Rust製のBevyを使ったタイミングゲームの作り方について説明しています。

タイミングゲームとは、細長いスライダーの中に左右に動くバーがあり、
そのバーを真ん中にタイミングを合わせて、高得点を狙うというゲームです。

こんな感じ

![タイミングゲーム](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/922938/abcbd0bc-97f0-4fa0-a740-113c6f16c681.gif)

## 参考リンク

タイミングゲームを作成する際に参考にしたサイトは以下の通り。

https://bevyengine.org/examples/

## ソースコード

実装したタイミングゲームのソースコードは`GitHub`に保存しています。

https://github.com/ittokunvim/doc-bevy-timing-game

## バージョン

タイミングゲームの実装に使用した`Bevy`のバージョンは`0.10.1`です。

もしも上記とは異なるバージョンで開発を行うと動作しない可能性が高いです。

## Cargoを追加

`Bevy`を使用するには`Cargo`が必要です。

以下のコマンドを実行して`Cargo`を追加しましょう。

```sh
# ディレクトリを作成
mkdir bevy-timing-game
# Cargoを追加
cargo init
# 動作を確認
cargo run
```

## アセットをダウンロード

以下のリストは、タイミングゲームで使用するアセットです。

ゲームを動かすには以下のアセットをダウンロードする必要があります。

保存場所は、まず`assets`ディレクトリを作成し、フォントは`assets/fonts`、音源は`assets/sounds`に保存します。

- [FiraSans-Bold.ttf](https://github.com/ittokun/bevy-games/blob/main/assets/fonts/FiraSans-Bold.ttf)
- [FiraMono-Medium.ttf](https://github.com/ittokun/bevy-games/blob/main/assets/fonts/FiraMono-Medium.ttf)
- [timing.ogg](https://github.com/ittokunvim/example-bevy/blob/main/assets/sounds/timing.ogg)

## Bevyを追加

`Cargo`に`Bevy`を追加して、動かしてみましょう。

まず`Cargo.toml`に以下の記述を行います。

```toml
[dependencies]
bevy = "0.10.1"
```

次に`src/main.rs`に以下のコードを記述します。

ここでは、先ほどダウンロードしたアセットのパス、画面サイズ、タイトル名、カメラ、終了キーなどが設定されています。

終了キーは`Esc`キーで、押すとプログラムが終了します。

```rust
use bevy::prelude::*;

const GAMETITLE: &str = "タイミングゲーム";
const WINDOW_SIZE: Vec2 = Vec2::new(640.0, 480.0);
const BACKGROUND_COLOR: Color = Color::rgb(0.9, 0.9, 0.9);

const PATH_FONT_BOLD: &str = "fonts/FiraSans-Bold.ttf";
const PATH_FONT_MEDIUM: &str = "fonts/FiraMono-Medium.ttf";
const PATH_SOUND_TIMING: &str = "sounds/timing.ogg";

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
        .insert_resource(ClearColor(BACKGROUND_COLOR))
        .insert_resource(FixedTime::new_from_secs(1.0 / 60.0))
        .add_startup_system(setup_camera)
        .add_system(bevy::window::close_on_esc)
        .run();
}

fn setup_camera(mut commands: Commands) {
    let camera = Camera2dBundle::default();
    commands.spawn(camera);
}
```

ここまでできたら以下のコマンドを実行してみましょう。
ウィンドウが表示されたら成功です。

```sh
cargo run
```

## セットアップを実装

次にタイミングゲームのセットアップ、初期画面の設定をしていきます。

以下のコードを`src/main.rs`に追加してみましょう。

ここではスライダー、キューというコンポーネントを作成し、描画しています。

スライダーはタイミングを判定する役割、キューはタイミングを決定する役割を持ちます。

```rust
// ...

const CUE_SIZE: Vec2 = Vec2::new(6.0, 48.0);
const CUE_POSITION: Vec3 = Vec3::new(0.0, 0.0, 99.0);
const CUE_COLOR: Color = Color::rgb(0.4, 0.4, 0.4);

const SLIDER_SIZE: Vec2 = Vec2::new(480.0, 48.0);
const SLIDER_POSITION: Vec3 = Vec3::new(0.0, 0.0, 0.0);
const SLIDER_COLOR: Color = Color::rgb(0.8, 0.8, 0.8);

#[derive(Component)]
struct Cue;

#[derive(Component)]
struct Slider;

fn main() {
    App::new()
        // ...
        .add_startup_system(setup)
        // ...
}

fn setup(mut commands: Commands) {
    // Cue
    commands.spawn((
        SpriteBundle {
            sprite: Sprite {
                color: CUE_COLOR,
                custom_size: Some(CUE_SIZE),
                ..default()
            },
            transform: Transform::from_translation(CUE_POSITION),
            ..default()
        },
        Cue,
    ));
    // Slider
    commands.spawn((
        SpriteBundle {
            sprite: Sprite {
                color: SLIDER_COLOR,
                custom_size: Some(SLIDER_SIZE),
                ..default()
            },
            transform: Transform::from_translation(SLIDER_POSITION),
            ..default()
        }, 
        Slider,
    ));
}
```

セットアップが終わったら以下のコマンドを実行して動作を確認してみましょう。

```sh
cargo run
```

### キューを動かす

次にキューを動かす処理を書いていきましょう。

以下のコードを追加することでキューが右に動かすことができます。

内容は`Velocity`コンポーネントを作成し、それをキューに追加します。
これによりキューは、速度（`Velocity`）を持つことができ、キューを動かすことが可能となります。

しかしコンポーネントを持たせるだけでは意味がないので、`apply_velocity`というシステムを追加して、
`Velocity`コンポーネントにどのように動くことができるのかを定義しています。

```rust
// ...
const CUE_SPEED: f32 = 200.0;
const CUE_DIRECTION: Vec2 = Vec2::new(1.0, 0.0);

// ...

#[derive(Component, Deref, DerefMut)]
struct Velocity(Vec2);

fn main() {
    App::new()
        // ...
        .add_system(apply_velocity)
    // ...
}

fn setup() {
    // ...

    // Cue
    commands.spawn((
        SpriteBundle {
            // ...
        },
        Cue,
        Velocity(CUE_DIRECTION.normalize() * CUE_SPEED),
    ));
 }

fn apply_velocity(mut query: Query<(&mut Transform, &Velocity)>, time_step: Res<FixedTime>) {
    for (mut transform, velocity) in &mut query {
        transform.translation.x += velocity.x * time_step.period.as_secs_f32();
    }
}
```

では`cargo run`を実行して動作を確認してみましょう。
キューが右に移動すれば成功です。

```bash
cargo run --example timing
```

## 衝突判定を追加する

次にキューに衝突判定を追加します。

キューは中央から右に移動し、スライダーの右端にたどり着いたら左に移動し、左端に着いたら右に移動し...を繰り返す処理を実装していきます。

`baunceback_cue`の内容は、まずキュー、スライダーの位置と、キューの移動速度の値を取得しています。

そして、キューがスライダーの左右の端に達したら、
移動速度の値を反転させて今まで移動していた方向と逆の方向に移動させています。

```rust
// ...

fn main() {
    App::new()
        // ...
        .add_system(baunceback_cue)
    // ...
}

// ...

fn baunceback_cue(
    mut cue_query: Query<(&Transform, &mut Velocity), With<Cue>>,
    slider_query: Query<&Transform, With<Slider>>,
) {
    let (cue_transform, mut cue_velocity) = cue_query.single_mut();
    let slider_transform = slider_query.single();
    let cue_x = cue_transform.translation.x;
    let slider_x = slider_transform.translation.x;

    if cue_x - CUE_SIZE.x / 2.0 < slider_x - SLIDER_SIZE.x / 2.0
    || cue_x + CUE_SIZE.x / 2.0 > slider_x + SLIDER_SIZE.x / 2.0 {
        cue_velocity.x = -cue_velocity.x;
    }
}
```

では`cargo run`を実行して動作を確認してみましょう。
キューが左右に移動すれば成功です。

## タイミング処理を追加する

次はスライダー中に`ok, good, perfect`の3つのエリアを追加し、
そのエリア内で左右に動くキューでタイミングを決める処理を書いていきます。

内容は、まずスライダーの上にタイミングのエリアがわかるように色付けしたそれぞれのレンジを描画します。

そして`decide_timing`を定義し、`TIMING_KEY`が押されたタイミングにキューの位置を取得し、
エリアに対応したテキストを出力します。

```rust
// ...

const TIMING_KEY: KeyCode = KeyCode::Space;

// ...

const PERFECT_RANGE: f32 = 20.0;
const PERFECT_COLOR: Color = Color::rgb(0.8, 0.2, 0.2);
const GOOD_RANGE: f32 = 80.0;
const GOOD_COLOR: Color = Color::rgb(0.2, 0.8, 0.2);
const OK_RANGE: f32 = 160.0;
const OK_COLOR: Color = Color::rgb(0.2, 0.2, 0.8);

// ...

fn main() {
    App::new()
        // ...
	.add_system(decide_timing)
	// ...
}

fn setup(mut commands: Commands) {
    // ...
    // Range
    let closure = |range :f32, color: Color, z: f32| {
        SpriteBundle {
            sprite: Sprite {
                color,
                custom_size: Some(Vec2::new(range, SLIDER_SIZE.y)),
                ..Default::default()
            },
            transform: Transform::from_xyz(0.0, 0.0, z),
            ..Default::default()
        }
    };
    commands.spawn(closure(PERFECT_RANGE, PERFECT_COLOR, 3.0));
    commands.spawn(closure(GOOD_RANGE, GOOD_COLOR, 2.0));
    commands.spawn(closure(OK_RANGE, OK_COLOR, 1.0));
}

fn decide_timing(
    keybord_input: Res<Input<KeyCode>>,
    query: Query<&Transform, With<Cue>>,
) {
    if keybord_input.just_pressed(TIMING_KEY) {
        let transform = query.single();
        let x = transform.translation.x;

        if x < PERFECT_RANGE / 2.0 && x > -PERFECT_RANGE / 2.0 { println!("Perfect!!!"); }
        else if x < GOOD_RANGE / 2.0 && x > -GOOD_RANGE / 2.0 { println!("Good!!"); }
        else if x < OK_RANGE / 2.0 && x > -OK_RANGE / 2.0 { println!("Ok!"); }
        else { println!("Bad..."); }
    }
}
```

では`cargo run`を実行して動作を確認してみましょう。
キューの位置に応じたテキストが表示されたら成功です。

## スコアボードを追加する

次はスコアボードを追加します。

スコアボードを実装するために、`Bevy`が提供している`Resource`を使用します。
これはゲーム内で値を表示したり変更したりするのに便利です。

内容はスコアボードを画面左上に配置し、`update_scoreboard`でスコアボードの値が変更されたらすぐに値を描画し直す処理を追加しています。

```rust
// ...

const PATH_FONT_BOLD: &str = "fonts/FiraSans-Bold.ttf";
const PATH_FONT_MEDIUM: &str = "fonts/FiraMono-Medium.ttf";

// ...

const TEXT_COLOR: Color = Color::rgb(0.1, 0.1, 0.1);

// ...

const SCOREBOARD_FONT_SIZE: f32 = 30.0;
const SCOREBOARD_TEXT: &str = "Score: ";
const SCOREBOARD_PADDING: Val = Val::Px(5.0);

// ...

#[derive(Resource)]
struct Scoreboard(usize);

fn main() {
    App::new()
        // ...
        .insert_resource(Scoreboard(0))
        // ...
        .add_system(update_scoreboard)
}

// ...

fn setup(
    mut commands: Commands,
    asset_server: Res<AssetServer>
) {
    // ...

    // Scoreboard
    commands.spawn(
        TextBundle::from_sections([
            TextSection::new(
                SCOREBOARD_TEXT,
                TextStyle {
                    font: asset_server.load(PATH_FONT_BOLD),
                    font_size: SCOREBOARD_FONT_SIZE,
                    color: TEXT_COLOR,
                },
            ),
            TextSection::from_style(TextStyle {
                font: asset_server.load(PATH_FONT_MEDIUM),
                font_size: SCOREBOARD_FONT_SIZE,
                color: TEXT_COLOR,
            }),
        ])
        .with_style(Style {
            position_type: PositionType::Absolute,
            position: UiRect {
                top: SCOREBOARD_PADDING,
                left: SCOREBOARD_PADDING,
                ..Default::default()
            },
            ..Default::default()
        }),
    );
}

fn update_scoreboard(
    scoreboard: Res<Scoreboard>,
    mut query: Query<&mut Text>,
) {
    let mut text = query.single_mut();
    text.sections[1].value = scoreboard.0.to_string();
}
```

では`cargo run`を実行して動作を確認してみましょう。
画面左上にスコアが表示されていれば成功です。

## ポイントを追加

次はタイミングを決めた時にエリアに応じたポイントを定義し、
スコアに加算減算していく処理を追加していきます。

ここでは先ほど作成した`decide_timing`の`println`の箇所に変更を加えています。
さらにスコアがマイナスにならないような処理も追加しています。

```rust
// ...

const POINT_PERFECT: usize = 100;
const POINT_GOOD: usize = 50;
const POINT_OK: usize = 10;
const POINT_BAD: usize = 100;

// ...

fn decide_timing(
    mut scoreboard: ResMut<Scoreboard>,
    keybord_input: Res<Input<KeyCode>>,
    query: Query<&Transform, With<Cue>>,
) {
    if keybord_input.just_pressed(TIMING_KEY) {
        let transform = query.single();
        let x = transform.translation.x;

        if x < PERFECT_RANGE / 2.0 && x > -PERFECT_RANGE / 2.0 { scoreboard.0 += POINT_PERFECT; }
        else if x < GOOD_RANGE / 2.0 && x > -GOOD_RANGE / 2.0  { scoreboard.0 += POINT_GOOD;    }
        else if x < OK_RANGE / 2.0 && x > -OK_RANGE / 2.0      { scoreboard.0 += POINT_OK;      }
        else {
            scoreboard.0 = if scoreboard.0 < POINT_BAD { 0 }
                else { scoreboard.0 - POINT_BAD }
        }
    }
}
```

では`cargo run`を実行して動作を確認してみましょう。
画面左上にスコアが更新されていれば成功です。

## タイミング音を追加

次にタイミングを決めた時に音が鳴るようにします。

ここでは`Bevy`の`Event`機能を使用します。この機能はとても便利で、
条件をイベントとして扱うことができ、別のシステムで処理を記述することができます。
説明が難しい...

```rust
// ...

const PATH_SOUND_TIMING: &str = "sounds/timing.ogg";

// ...

#[derive(Default)]
struct TimingEvent;

#[derive(Resource)]
struct TimingSound(Handle<AudioSource>);

fn main() {
    App::new()
        // ...
        .add_event::<TimingEvent>()
        .add_system(play_timing_sound)
}

// ...

fn setup(mut commands: Commands, asset_server: Res<AssetServer>) {
    // Sound
    let sound = asset_server.load(PATH_SOUND_TIMING);
    commands.insert_resource(TimingSound(sound));

     // ...
}

fn decide_timing(
    mut events: EventWriter<TimingEvent>,
    // ...
) {
    if keyboard_input.just_pressed(TIMING_KEY) {
        // ...

        events.send_default();

        // ...
    }
}

fn play_timing_sound(
    mut events: EventReader<TimingEvent>,
    audio: Res<Audio>,
    sound: Res<TimingSound>,
) {
    if events.is_empty() { return; }
    events.clear();
    audio.play(sound.0.clone());
}
```

では`cargo run`を実行して動作を確認してみましょう。
タイミング決定時に音が出れば成功です。

## まとめ

お疲れ様でした！いかがだったでしょうか？

`Bevy`はまだ発展途上ではありますが、小規模なゲームなどは難なく作ることができてしまいます。

もしこの記事が何かの助けになれば幸いです。ありがとうございました！

