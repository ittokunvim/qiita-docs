---
title: deviseを使ったtwitter認証でコケた話
tags:
  - Rails
private: false
updated_at: '2021-07-17T14:53:54+09:00'
id: cf0989e1ae4e45bac5fe
organization_url_name: null
slide: false
ignorePublish: false
---
## 環境
- ruby 3.0.1p64 (2021-04-05 revision 0fb782ee38) [arm64-darwin20]
- Rails 6.1.4

## 内容
[[*Rails*] deviseの使い方（rails6版）](https://qiita.com/cigalecigales/items/16ce0a9a7e79b9c3974e)の記事を参考にdeviseを実装していたところTwitterで認証するところでコケた。

```console:terminal
D, [2021-07-17T13:20:36.167758 #35278] DEBUG -- omniauth: (twitter) Request phase initiated.
W, [2021-07-17T13:20:36.168254 #35278]  WARN -- omniauth: Attack prevented by OmniAuth::AuthenticityTokenProtection
E, [2021-07-17T13:20:36.168409 #35278] ERROR -- omniauth: (twitter) Authentication failure! authenticity_error: OmniAuth::AuthenticityError, Forbidden
```

ページの方でもエラーが発生（何書いてあるか忘れた）

## 解決
gemにomniauth-rails_csrf_protectionを追加する。

```ruby:gemfile
gem 'omniauth-rails_csrf_protection'
```

よくよく調べると`omniauth:2.0`から必要になるGemだそうです
