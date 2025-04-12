---
title: RailsでCustom Fontを使う(Webpacker)
tags:
  - Rails
private: false
updated_at: '2024-12-03T03:23:16+09:00'
id: 572690601880ca2d4076
organization_url_name: null
slide: false
ignorePublish: false
---
フォントをダウンロードして、app/javascript/src配下に置く

app/javascript/src/style.scssにフォントのパスを記入と使用

```scss
@font-face {
  font-family: "maru-gosic";
  src: url('./Corporate-Logo-Rounded.ttf') format('truetype');
}

html {
  font-family: "maru-gosic";
}
```

これで、OK!!

参考：　https://anthonybroadcrawford.com/2020/02/14/custom-fonts-rails-six.html
