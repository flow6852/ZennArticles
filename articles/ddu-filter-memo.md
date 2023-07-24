---
title: "ddu.vimのconverter作成備忘録"
emoji: "🌟"
type: "idea"
topics:
  - "vim"
  - "neovim"
  - "denops"
published: true
published_at: "2023-06-21 06:54"
---

## はじめに

本記事はVim/Neovimのpluginであるdenopsで書かれたddu.vimの拡張plugin，ddu-filter-converter_kindを作成した際の備忘録です.
ddu.vimの拡張pluginの作成ハードルの低さをお伝えできればと思います．

:::message 
本記事は以下の記事の続編です。
https://zenn.dev/flow6852/articles/b5c61d6ec66afb
:::

:::message alert
筆者はtypescriptのHello Worldを書いたかどうか覚えていないくらいの実力です.
そのため,以下に書くような点はtypescript依存かdeno依存かdenops依存かddu.vim依存かよくわかっていない場合があります.
:::

https://github.com/neovim/neovim

https://github.com/vim-denops/denops.vim

https://github.com/Shougo/ddu.vim


## 動機

動機は以下の3つです

1. `ddu-source-url`のkindの候補として`ddu-kind-file`と`ddu-kind-url`があるが、ユーザがkindを変更するのが大変
1. kindだけを試作したい人はsourceを一緒につくるか既存のsourceのkindをかきかえるかしかなさそうだけどユーザーがやることじゃなさそう
1. 前回の記事のネタがなかった

## 結果

https://github.com/flow6852/ddu-filter-converter_kind

## ddu.vimと拡張方法

作者様が書かれた記事がかなりわかりやすいので本格的に作成したい場合はこちらを参照してください.

https://zenn.dev/shougo/articles/ddu-vim-beta

https://zenn.dev/shougo/articles/ddu-vim-make-plugins

## conveter開発時に考えるとよいこと

### 生成後のitemで変更したいものを変える

ddu.vim起動中にitemを選んで`echom ddu#get_item()`をした結果で変更したい要素を変更するだけです.
`ddu-filter-converter_kind`では`kind`を変更し、`action`中の要素でコピーしたいものをコピーしています.

## まとめ

ddu.vimのインタフェースは洗礼されていてきれいに分割されているのでddu.vimのfilterは生成済みのitemを加工することだけ考えればいいいいというすごいpluginなので使いましょう．
