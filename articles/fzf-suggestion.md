---
title: "fzf-suggestion: シェルでコマンドパレットを使う"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [bash, zsh, fzf]
published: false
---

### 概要

[fzf-suggestion](https://github.com/k-ota106/dotfiles/blob/main/bin/fzf-suggestion)は、登録した文字列の絞り込みと、  
得られた文字列を使ったコマンド実行をサポートするフレームワークです。

Alveo操作の入力補助を例として、このツールの紹介を行います。

`fzf-suggestion`は以下のことが出来ます。

* 選択行をファイルとして閲覧 ([bat](https://github.com/sharkdp/bat) or less)
* 任意コマンドを実行（選択行を`{}`として参照できる）
* 選択行の後に任意文字列を追加で打ち込んでコマンド実行
* 登録文字列の分散ファイル管理 (`./`、Gitルート、`$HOME/`を参照する) 
* 登録文字列の動的生成
    * 設定ファイルがスクリプトやプログラムならそれを実行して、
      標準出力を登録文字列として扱う

以下のような状況でおススメ出来ます。

* 初めての作業でどんなコマンドを使ったら良いのか分からない。
* 良く使うコマンドをメモ帳に書いて、毎回コピペしている。
* ディレクトリ毎に使うコマンドがだいたい決まっている。
* 汎用的な定型文を簡単に呼び出したい。

具体的な動作例を先に見たい方は、ページ下部にあるスクリーンショットを見て下さい。

---

`fzf-suggestion`の絞り込み機能は、[fzf][]を使って実装しています。  
`fzf`単独でもとても便利なツールですが、長くなるのでここでの説明は省略します。  
`fzf`について知りたい人は[github][fzf]を参照して下さい。  

これをインストールするだけでも、CLI生活がかなり快適になります。  
コマンド履歴を引き出すのに時間が掛かったり、1ファイルを探すのに  
タブ補完を何回も押している人には特におススメ出来ます。

[fzf]: https://github.com/junegunn/fzf

### fzf-suggestionの起動

端末で`fzf-suggestion`を実行すれば起動できます。  

コマンドが起動したら、`Ctrl-h`を押して操作方法を確認出来ます。

ショートカット登録しておくと瞬時に呼び出せて超便利です。  
以下にbash/zshで`Ctrl-o`を押したら`fzf-suggestion`を起動する例を示します。  

* bash (`~/.bashrc` or `~/.fzf.bash` に記載する)

    ```bash
    # open fzf-suggestion command
    __fzf_suggestion__() {
        # Save the tty settings and restore them on exit.
        SAVED_TERM_SETTINGS="$(stty -g)"
        trap "stty \"${SAVED_TERM_SETTINGS}\"" EXIT
        
        # Force the tty (back) into canonical line-reading mode.
        stty cooked echo
    
        fzf-suggestion
        stty $SAVED_TERM_SETTINGS
    }
    
    bind -m emacs-standard -x '"\C-o": __fzf_suggestion__'
    bind -m vi-command -x '"\C-o": __fzf_suggestion__'
    bind -m vi-insert -x '"\C-o": __fzf_suggestion__'
    ```

* zsh (`~/.zshrc` or `~/.fzf.zsh` に記載する)

    ```zsh
    # open fzf-suggestion command
    bindkey -s '^O' '^ufzf-suggestion^M'
    ```

### .suggestionファイルのフォーマット

* ファイル内の各行が選択候補に入ります。
* `#`で始まる行は無視します。
* 空行は無視します。
* 行中の`#`以降は表示されますが、コマンドとしての実行対象になりません。
* 実行可能ファイルの場合、その標準出力が上記のルールに従います。

コメントが書けるので、自分用の作業メモをこのルールに合わせれば、  
.suggestionファイルとしても活用出来ます。

### .suggestionファイルの配置

* 以下の順番で`.suggestion`を検索し、見つかったものを全て読み込みます。
    1. `./.suggestion`
    2. `<.gitのあるパス>/.suggestion`
    3. `$HOME/.suggestion`
* 良く使うコマンドをカレントディレクトリに配置し、とりあえず候補に入れておきたいものを
  それ以外のパスに配置するのが良さそうです。
* 例えば、lintやsim結果など、良く参照するファイル名を絶対パスで`$HOME/.suggestion`に書いておけば、  
  どのディレククトリからでも簡単に開くことが出来ます。

### .suggestionファイルのサンプル (Alveo操作)

XilinxのAlveoカードを操作する場合の例を示します。

この例では、`.suggestion`ファイルに実行権限を付けて、
その標準出力を選択候補としています。

デバイスの設定など、可変要素がある場合にこの方法を使うと便利です。

`#`でコメントを付けることが出来ます。  
コメント部分はコマンド候補には入りませんが、`fzf`の絞り込み対象に  
なるので、カテゴリなどのタグを埋め込むのもおススメです。

```bash
#!/bin/bash

xbutil=/opt/xilinx/xrt/bin/xbutil

# $ xbutil examine
# Devices present
#   [0000:01:00.1] : xilinx_u55c_gen3x16_xdma_base_2 user(inst=129)
#    ~~~~~~~~~~~~
bdf=0000:01:00.1

cat <<EOF
$xbutil reset -d $bdf                           # Reset FPGA
$xbutil examine -d $bdf -r all                  # Report FPGA status
$xbutil examine -d $bdf -r firewall             # firewall should be Level=0
$xbutil examine -d $bdf -r thermal              # FPGA tempature (50度超えたら注意)
$xbutil examine -d $bdf -r dynamic-regions      # FPGA内のカーネル情報
$xbutil program -d $bdf -u vadd.xclbin          # Download to FPGA
EOF
```

**デモ**

![fzf-suggestion.gif](/images/fzf-suggestion.gif)

