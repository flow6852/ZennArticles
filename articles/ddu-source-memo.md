---
title: "ddu.vimのsource作成の備忘録"
emoji: "🪄"
type: "idea"
topics:
  - "vim"
  - "neovim"
  - "denops"
published: true
published_at: "2023-02-15 18:35"
---

## はじめに

本記事はVim/Neovimのpluginであるdenopsで書かれたddu.vimの拡張plugin，ddu-source-qfを作成した際の備忘録です.ddu.vimの拡張pluginの作成ハードルの低さをお伝えできればと思います．

:::message alert
筆者はtypescriptのHello Worldを書いたかどうか覚えていないくらいの実力です.そのため,以下に書くような点はtypescript依存かdeno依存かdenops依存かddu.vim依存かよくわかっていない場合があります.
:::

https://github.com/neovim/neovim

https://github.com/vim-denops/denops.vim

https://github.com/Shougo/ddu.vim

https://github.com/flow6852/ddu-source-qf

https://github.com/Shougo/ddu-kind-file


## 動機

私は普段Neovimを使っています.
その中でLaTeXで文書を作成するときにVimTeXとTeXLabを同時に使っています.

https://github.com/lervag/vimtex

https://github.com/latex-lsp/texlab

ビルドや編集時に出力されるエラーやWarningはそれぞれ別々のQuickFixに出力されます(場合によります).しかしQuickFixを一つ一つ切り替える操作が必要で，(筆者のRAMやROMが足りないので)操作時に様々な情報を失ってしまいます.そこで,これら２つを同時に表示できないかを検討していました.

## 結果

以下の画像のような感じに表示できます.過去のQuickFixを複数取り出して１つのfloating windowに表示されています.![](https://storage.googleapis.com/zenn-user-upload/8c5ea46dbe3d-20230214.png)

また,vimgrepの結果を複数表示することが可能です.

![](https://storage.googleapis.com/zenn-user-upload/365bb4e3bed3-20230214.png)

これらは以下のように設定を加えると`;q`や`;v`で起動できます(保存時にDiagnosticsからqflistに変換しています)

```
call ddu#custom#patch_local('qf', {
    \   'ui': 'ff',
    \   'sourceOptions': {
    \     '_': {
    \       'matchers': ['matcher_substring'],
    \     },
    \   },
    \   'uiParams': {
    \     'ff': {
    \       'split': 'floating',
    \       'startFilter': v:false,
    \       'highlights': {'selected': 'Statement'},
    \       'winCol': &columns/8,
    \       'winWidth': 3*&columns/4,
    \       'winRow': &lines/8,
    \       'winHeight': 3*&lines/8,
    \       'autoAction': {'name':'preview'},
    \       'previewFloating': v:true,
    \       'previewCol': &columns/8,
    \       'previewWidth': 3*&columns/4,
    \       'previewRow': 7*&lines/8-1,
    \       'previewHeight': 3*&lines/8,
    \       }
    \   },
    \   'kindOptions': {
    \     'custom-list': {
    \       'defaultAction': 'open',
    \     },
    \   }
    \ })
    
augroup mergeqf
    autocmd!
    autocmd BufWritePost * lua vim.diagnostic.setqflist({open=false})
augroup END

let format='%T|%p|%y|%t'

nmap <silent> ;q <Cmd>call ddu#start({
    \   'name': 'qf',
    \   'sources': [{
    \       'name': 'qf',
    \       'params': {'what': {'title': 'Diagnostics'}, 'format': format}
    \               },
    \       {'name': 'qf',
    \        'params': {'what': {'title': 'VimTeX'}, 'format': format,
    \                   'isSubst': v:true,
    \                  }
    \       }]})<CR>

nmap <silent> ;v <Cmd> call ddu#start({
            \ 'name': 'qf', 'sources': [
            \ {'name': 'qf',
            \ 'params': {'what': {'title': ':vimgrep'},
            \            'isSubst': v:true,
            \            'format': format,
            \            'dup': v:true}},
            \ {'name': 'qf',
            \ 'params': {'what': {'title': ':lvimgrep'},
            \            'isSubst': v:true,
            \            'format': format,
            \            'nr': 0,
            \            'dup': v:true}}]})<CR>
```

## ddu.vimと拡張方法

作者様が書かれた記事がかなりわかりやすいので本格的に作成したい場合はこちらを参照してください.

https://zenn.dev/shougo/articles/ddu-vim-beta

https://zenn.dev/shougo/articles/ddu-vim-make-plugins


## source開発時に考えるべきこと

開発者はsourceの対象を決めることだけです．そしてコーディング時には対象から必要な情報をItemにするだけです．実際に本拡張pluginの本質はユーザが指定した`{what}`に対応するqflistを`getqflist()`を使ってqflistを拾ってきてそれぞれItemにするしかしていません．

https://deno.land/x/ddu_vim@v2.0.0/types.ts?s=Item

:::message alert
vimからデータを拾ってくる際に，一度で大量のデータを拾ってしまうとdenopsサーバへ転送するときにオーバヘッドがかかってしまって最悪denopsのサーバが一旦落ちて再起動してしまうみたいです．(再起動してしまう理由は筆者の勉強不足でよくわかっていません)なので大量のデータを送る場合はvimscript側でデータ数を制限したほうがよいです.
:::

## source開発時に考えるべき”でない”こと

### 一度に複数のsourceを拾うこと

ddu.vimの関数に本体を起動する`ddu#start()`やそのグローバルな設定を与える`ddu#custom#patch_global()`があります．その引数にどのsourceを与えるかを決める`sources`というオプションがあります．この`sources`に複数のsourceを入れることで実質mergeしてくれます．なのでシンプルにただ欲しい物だけをひとつひとつItemにしてつっこむだけです．

:::message 
複数のsourceを同時に指定できる点はddu.vimの強みの１つです．^[[https://twitter.com/ShougoMatsu/status/1623826837951619074?s=20](https://twitter.com/ShougoMatsu/status/1623826837951619074?s=20)]
:::

### 拾ってくるときにフィルタリングをしっかりやる必要はない

ddu.vimはfilterも分離されており，`ddu#start()`後にユーザが欲しい情報をフィルタリングできます．
なのでsourceがやるべきことはなんでもいいから重複してもいいからとにかく拾ってくることだけです（重複してよいかどうかはものによるかもしれません）．

## ddu-kind-fileに依存する場合の注意点

本拡張pluginはddu-kind-fileに依存しています．ユーザが選択したアイテムに対してどのような操作を行うかを与えるものです．以下は作成時にddu-kind-fileに依存する部分でつまずいた点です．作者様に教えていただいて解決しました.

### pathにはVimのカレントディレクトリを使った絶対パスを指定する

denoにも絶対パスを取得する（検索する？）関数が用意されています.

https://deno.land/std@0.177.0/path/mod.ts?s=resolve

ですが,vimにはchdirコマンドでカレントディレクトリを移動したりできるので,検索されたパスと編集中であったり取得したいパスが異なる場合があります.なのでDenopsでフルパスを作らず，vimscriptの`getcwd()`と`bufnr()`を結合したものが推奨されます.

### ~~bufferに取り込まれているがhiddenなファイルはactionのbufNrを指定しない~~

~~タイトルのとおりです.対処の方法は最初に`getbufinfo()`でバッファの情報を取得してhiddenかどうかを判定すればいいです.開発時はこの理由はよくわかっていませんでした.今もよくわかっていませんが,ddu-kind-fileはbufNrが指定されているかどうかの判定を先に行っていて,hiddenのフラグが立っているときにはpreviewが表示されないみたいです.~~ 

1. 修正されて，hiddenなbufferもpreviewできるように修正されました．(多分このcommitです)
2. listedなbufferかどうかの判定が必要です．(ddu-ui-ffにlistedなbufferかどうかを判定して正常にpreviewできるように制御してくれています^[214行目で存在判定を行って232行目以降で具体的に対応しているそうです．bdeleteでユーザがバッファから削除した場合を想定されています]．)

https://github.com/Shougo/ddu-ui-ff

## まとめ

ddu.vimのインタフェースは洗礼されていてきれいに分割されているのでddu.vimのsourceはとにかく拾ってきてItemに入れるだけで完成するすごいpluginなので使いましょう．

## 今後の開発方針

1. textやwordに入れるときのformatを設定できるようにしていますが，種類が少ないのでformatを拡張していきたいです(現在フルパスを指定できますが，例えば相対パスやファイル名のみが欲しい場合があるかもしれません)．
2. 現在ddu-source-qfはddu-kind-fileに依存していますが，対象のidと一致するqflistをcopenしたい場合があるのでもしかしたら不要かもしれません(が，ddu-kind-fileにbufNrを渡す渡さないをユーザに任せるのは結構大変な気がします)．
3. (既出かもしれませんが)ActionDataの中身ををfilterするddu-filter-action_dataやQuickFixまわりの操作をするddu-kind-qfを作れないかなと思っています．

## 最後に

本拡張pluginの作成にあたり，ddu.vimの作者様には指導していただき，質問に返答頂いたvim-jpの皆様にたくさんの助言をいただきました．心より感謝致します．