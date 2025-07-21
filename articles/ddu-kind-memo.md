---
title: "ddu.vimのkind作成備忘録"
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

本記事はVim/Neovimのpluginであるdenopsで書かれたddu.vimの拡張plugin，ddu-kind-vim_typeを作成した際の備忘録です.ddu.vimの拡張pluginの作成ハードルの低さをお伝えできればと思います．

:::message 
本記事は以下の記事の続編です。
https://zenn.dev/flow6852/articles/bcf4557b882e0d
:::

:::message alert
筆者はtypescriptのHello Worldを書いたかどうか覚えていないくらいの実力です.そのため,以下に書くような点はtypescript依存かdeno依存かdenops依存かddu.vim依存かよくわかっていない場合があります.
:::

https://github.com/neovim/neovim

https://github.com/vim-denops/denops.vim

https://github.com/Shougo/ddu.vim


## 動機

他要素も作ってみたくなったので作りました。

## 結果

https://github.com/flow6852/ddu-kind-vim_type

上のkindを使うsourceとしてvimの変数や関数を拾ってくるものも作りました．

https://github.com/flow6852/ddu-source-vim_variable

https://github.com/flow6852/ddu-source-vim_function


## ddu.vimと拡張方法

作者様が書かれた記事がかなりわかりやすいので本格的に作成したい場合はこちらを参照してください.

https://zenn.dev/shougo/articles/ddu-vim-beta

https://zenn.dev/shougo/articles/ddu-vim-make-plugins


## kind開発時に考えるべきこと

### kindは各itemを使って実現したいこと

開発者はsourceの各itemについて実現したいことを書くだけです．実際に本拡張pluginの`setcmdline`アクションはユーザが指定したitemをコマンドラインにコピーしています．previewもcontentsに値を渡しているだけです．

既存のsourceを使ってkindを試験したい場合は`ddu-filter-converter_kind`を用いると楽です．
https://github.com/flow6852/ddu-filter-converter_kind

## kind開発時に考えるべき”でない”こと

### 書くか書かないかで迷うくらいなら書いたほうがいい

`ddu#custom#action`という関数が`ddu.vim`に用意されており、エンドユーザが自由にactionを追加できます。なので過不足を考えるより機能モリモリにしたほうがエンドユーザ的には嬉しいのではないかと思います。

## まとめ

ddu.vimのインタフェースは洗礼されていてきれいに分割されているのでddu.vimのkindはItemをどう使いたいかだけを考えればいいというすごいpluginなので使いましょう．
