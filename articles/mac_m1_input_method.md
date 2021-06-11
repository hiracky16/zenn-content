---
title: "MBA（M1 チップ搭載） にて Hammerspoon を使って Google 日本語入力に切り替える"
emoji: "🐶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [M1, Mac]
published: false
---

# 背景
会社にて MBA を新しく買ってもらったところ、新品はもうすでに M1 チップ搭載のものしかなかったらしい。
一抹の不安がありつつも了承して希望通り US キーボードで買ってもらったところ Karabiner が動かず日本語入力が満足にできなくなってしまった。
（cmd を入力ソースの切り替えに使っていたため）
下記イシューと同じ症状。

https://github.com/pqrs-org/Karabiner-Elements/issues/2636

仕方なく Karabiner 以外の解決策を探してたところ Hammerspoon にたどり着いて入力ソースの切り替えができるようにできた話。

# Hammerspoon について
- Mac でショートカットの設定やキーバインドの変更ができるツール
- Karabier は GUI ベースでキーバインドを変更できたが Hammerspoon は lua 言語でコードベースで記述していく

コード管理できるは今後マシンが変わったときでも同じ設定が使える点便利だと思った。

# 使い方
1. [ここ](https://github.com/Hammerspoon/hammerspoon/releases/tag/0.9.90) から zip でダウンロードする
2. 解凍して `Applications/` にファイルを移動
2. すると `~/.hammerspoon/` という隠しディレクトリができるのでその配下に `init.lua` を作成
2. `init.lua` にコードを記述していく
2. メニューから Reload Config する

実際に書いたコードは以下を参考にした。
https://mac-ra.com/hammerspoon-command-eikana02/

参考にしたコードでは MBA にデフォルトで入っている入力ソースを使っていたが、
筆者は普段 Google 日本語入力を使っているため、 下記にてそれらに切り替えていく。

## Google 日本語入力への対応

変更したい入力ソースに手動で切り替え、hammerspoon console に入力すると入力ソースが得られる。

```
> hs.keycodes.currentMethod()
Alphanumeric (Google)
```
この `Alphanumeric (Google)` （Google 日本語入力の英数） の文字列をコピーしておく。
同じように日本語入力ソースにも手動で切り替えて入力ソース名の文字列を得る（`Hiragana (Google)` だと思われる）

あとは参考に上げた記事のソースコードの L12 ~ L16 あたりを修正すると cmd での入力ソースの変更が可能になる。
ちなみにキーマップは [ここ](https://www.hammerspoon.org/docs/hs.keycodes.html#map) で定義されている。

```
if c == map['cmd'] then
  hs.keycodes.setMethod('Alphanumeric (Google)')
elseif c == map['rightcmd'] then
  hs.keycodes.setMethod('Hiragana (Google)')
end
```

最後にメニューから Reload Config してあげると完了！
