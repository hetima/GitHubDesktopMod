# GitHubDesktopMod

GitHub Desktopのウィンドウ下部に最近使ったリポジトリを表示するタブを取り付け、1クリックで選択できるようにした改造です。おまけでdiffのフォント表示がちょっと大きくなっています。monospaceフォントでHackGen35を先頭で指定しています。

`3.5.xx` などのフォルダに入っているファイルを置き換えれば、バージョンが合うならそのまま使えます。以下の準備や実行を行う必要はありません。ファイルをコピーするだけです。

```
C:\Users\YOURNAME\AppData\Local\GitHubDesktop\app-3.x.xx\resources\app
```

の中にあるファイルと入れ替えます。

このリポジトリに含まれていないバージョンでも使いたい場合はAIに作ってもらう必要があります。`PLAN.md`が指示書です。実際に改造した手順は以下の通りです。

## 準備

[GitHub Desktopのソース](https://github.com/desktop/desktop)が必要です。

クローンしてきます

```
git clone https://github.com/desktop/desktop
```

目的のバージョンに合わせます

```
git checkout -b release-3.6.2
```

あるいはこんな感じでリリースタグ指定してクローン

```
git clone --single-branch --depth 1 -b release-3.6.2 --recurse-submodules https://github.com/desktop/desktop
```

ビルドには yarn とかいろいろ必要ですが、足りないパッケージはAIに頼めばなんとかしてくれるでしょう。npmくらいは最初から入れといたほうがいいかもしれません。

## 実行

AIに「PLAN.mdを実行して」とか言えばすべてやってくれると思います。例：

```text
PLAN.mdの内容を実行してください。PLAN.mdの手順に沿って実装してください。
```

```text
PLAN.mdを実行してください。
ただし、PLAN.mdに曖昧な点や危険な変更があれば、実装前に確認してください。
変更は必要最小限にし、既存スタイルに合わせてください。
```

## 補足

AIは正しく実装してくれる保証はありません。モデルによる能力差もあります。うちの場合単純に `PLAN.md` を使ったのではなくいろいろチャットして作成した後にまとめたものがこのプランなので、うまく行かないかもしれません。gpt-5.5で作成しました。

<img src="https://i.imgur.com/obdZpiw.jpeg" />
