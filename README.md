 # Windowsのターミナル内で画像を表示する(Sixel Graphics)

## Sixel Graphics とは
特別なエスケープシーケンスをキャラクターベースのターミナルに送信することで、画像を表示する技術です。

詳細は[Sixel グラフィックス - VT100.net: VT330/VT340 プログラマーリファレンスマニュアル](https://github.com/fumiyas/translation-ja/blob/master/vt3xx-sixel.md)をご確認ください。

通常のターミナルエミュレーターは[VT100](https://ja.wikipedia.org/wiki/VT100)互換ですが、Sixelを表示するためにはVT200以降の機能をサポートしている必要があります。

LinuxやMacでは様々なターミナルがSixelをサポートしていますが、Windowsではかなり限られるようです。

このあたりを調べて試した限りこの辺はうまくいきませんでしたが、もっと簡単な方法を見つけたのでメモを残します
* [MSYS2](https://www.msys2.org/) 表示できず
* [VSCodeがSixelをサポート](https://zenn.dev/hankei6km/articles/display-images-on-vscode-terminal) 表示できず



## 色々調べた結果、最も早く確実な方法

[git for Windows](https://gitforwindows.org/)環境に含まれる`mintty`がSixelをサポートしているため、インストール、画像表示手順を記載します

1. [git for Windows](https://gitforwindows.org/)のインストール

2. `mintty`の起動と、(Sixcelのエスケープシーケンスに変換済みデータから)画像を表示できるか確認する

3. 画像ファイルからSixelのエスケープシーケンスに変換して、`mintty`で表示する

  libsixel というパッケージに入っている img2sixel コマンドを使うのが王道のようですが、Windowsでは、ソースからのコンパイルが必要になるようですので、node環境のライブラリを利用して変換します


### 1. [git for Windows](https://gitforwindows.org/)のインストール

[git for Windows](https://gitforwindows.org/)の「Download」からインストーラーをダウンロードしてインストールします。

### 2. `mintty`の起動と、(Sixcelのエスケープシーケンスに変換済みデータから)画像を表示できるか確認する

gitをインストールすると、`git bash`が利用できるようになります。

![alt text](./image.png)

`mintty`を起動するため`git bash`を起動します

起動した`git bash`に下記を入力して`mintty`を起動します
```
 mintty
```

起動すると環境を選択するダイアログが表示されます（どれを選んでも画像は表示できます）

![alt text](./image-1.png)

サンプル画像データ(エスケープシーケンスを含むSixel変換後データ)をダウンロードして、カレントディレクトリに保存します
[mountain.sixel]()

catでファイルを開くと画像が表示されます


![alt text](./image-2.png)



### 3. 画像ファイルからSixelのエスケープシーケンスに変換して、`mintty`で表示する

Windows上から簡単に利用できる、node環境のライブラリ[sixel](https://www.npmjs.com/package/sixel)を利用して変換します

* node版sixcel(とcanvas)をインストールする

```
mkdir node-sixel
cd node-sixel
npm init y
npm -i sixel canvas
```

4. [img2sixel.js]() を作成する
https://github.com/jerch/node-sixel/blob/master/img2sixel.js
のコードを一部修正(require('./lib/index') ⇒ require('sixel/lib/index'))

```js:img2sixel.js
/**
 * Example script as cmdline converter.
 * Call: `node img2sixel.js <image files>`
 */

// set to 16 for xterm in VT340 mode
const MAX_PALETTE = 256;

// 0 - default action (background color)
// 1 - keep previous content
// 2 - set background color
const BACKGROUND_SELECT = 0;

const { loadImage, createCanvas } = require('canvas');
const {
  introducer,
  FINALIZER,
  sixelEncode,
  image2sixel,
} = require('sixel/lib/index');

async function processImage(filename, palLimit) {
  // load image
  let img;
  try {
    img = await loadImage(filename);
  } catch (e) {
    console.error(`cannot load image "${filename}"`);
    return;
  }
  const canvas = createCanvas(img.width, img.height);
  const ctx = canvas.getContext('2d');
  ctx.drawImage(img, 0, 0);

  // use image2sixel with internal quantizer
  const data = ctx.getImageData(0, 0, img.width, img.height).data;
  console.log(`${filename}:`);
  console.log(
    image2sixel(data, img.width, img.height, palLimit, BACKGROUND_SELECT)
  );

  // alternatively use custom quantizer library
  // const RgbQuant = require('rgbquant');
  // const q = new RgbQuant({colors: palLimit, dithKern: 'FloydSteinberg', dithSerp: true});
  // q.sample(canvas);
  // const palette = q.palette(true);
  // const quantizedData = q.reduce(canvas);
  // console.log(`${filename}:`);
  // console.log([
  //   introducer(BACKGROUND_SELECT),
  //   sixelEncode(quantizedData, img.width, img.height, palette),
  //   FINALIZER
  // ].join(''));
}

async function main() {
  let palLimit = MAX_PALETTE;
  for (const arg of process.argv) {
    if (arg.startsWith('-p')) {
      palLimit = parseInt(arg.slice(2));
      process.argv.splice(process.argv.indexOf(arg), 1);
      break;
    }
  }
  for (const filename of process.argv.slice(2)) {
    await processImage(filename, palLimit);
  }
}

main();

```

4. 'mintty'を起動する

5. 動作確認
minttyから下記コマンドを実行する(git-bashから`mintty`と叩くと起動する）

```
node img2sixel.js mountain.png
```

![alt text](./image-3.png)

## その他のやり方

* imagemagick をインストールして、minttyから起動
Windowsにはconvertというコマンドが入っているので、フルパスで指定すること
```
convert rose: sixel:
```

`libsixel`
mingw環境でコンパイルすれば動くらしい
https://github.com/saitoha/libsixel?tab=readme-ov-file#cross-compiling-with-mingw

`MSYS2` は表示できなかった（理由不明）
https://www.asobou.co.jp/blog/web/msys2



https://qiita.com/yumetodo/items/4aa03d1eb3d887bca1a8
