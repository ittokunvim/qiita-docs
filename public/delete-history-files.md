---
title: カレントディレクトリ下のヒストリーファイルを消去する
tags:
  - ShellScript
  - termi
private: false
updated_at: '2024-12-03T03:22:20+09:00'
id: 8d0d57a7a00357149635
organization_url_name: null
slide: false
ignorePublish: false
---
皆さん！ヒストリーファイルが邪魔だなーと思うことがありませんか（僕はいつも思います）
そんな方に一ついいコマンドを紹介します

`rm \.[a-z]*_history`

「.から始まり_historyで終わるaからzまでのファイル名を消去する」という内容になっています。

```zsh
alias rmhist="rm \.[a-z]*_history"
```
自分はこのコマンドを「.zshrc」にエイリアスとして保存してます。

参考にまでどうぞ！
