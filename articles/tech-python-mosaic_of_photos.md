---
title: "pythonで始めるフォトモザイクアート"
emoji: "🧑‍💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python"]
published: true
---

# はじめに
Hey guys！
高専キャリア2024冬の全国大会で、起業家ピッチとLTで発表したアヤノです。

この記事は、LTで話した「高専キャリアのフォトモザイクアートを作る！」の記事版です。

![LTのタイトルスライド](/images/articles/tech-python-mosaic_of_photos/title.png)

# フォトモザイクアートとは？
フォトモザイクアートは、**数百から数千枚もの小さな写真や画像をタイルとして扱い、それらを無数に組み合わせることで一つの大きな画像や絵を作り上げるアート手法**です。このアート手法は、企業の広告やイベントの記念品、思い出や記念日を振り返る手段として使われています。

:::details フォトモザイクアートの名前の由来
フォトモザイクアートは、「写真（Photo）」と「モザイク（Mosaic）」の組み合わせから生まれました。

モザイクとは、もともと小さなタイルや石を組み合わせて模様や絵を作る古代の装飾技法の一つで、**バラバラの小さな要素が集まって全体として一つの大きな作品を形成する**ことを意味します。

そのため、モザイク技法を写真の世界に応用したフォトモザイクアートは、「フォト（写真）＋モザイク」という言葉で表現され、それがそのまま名前として定着しました。
:::


## フォトモザイクアートの例
言葉だけでは分かりずらいので、『私に天使が舞い降りた！』ののフォトモザイクアートを例として紹介します。

これがフォトモザイクアートです。写真を拡大すると、『私に天使が舞い降りた！』に登場する星野みやこの写真から、星野みやこのストカーの松本香子の写真が作られていることが分かります。このように大量の画像から一つの写真を作るのがフォトモザイクアートです。
![ex](/images/articles/tech-python-mosaic_of_photos/D1dPs5NUYAAJFUX.jpg)
*「私に天使が舞い降りた！」公式😇Blu-ray＆DVD発売中❣（@watatentv）より引用*

このフォトモザイクアートの作成経緯は今回の記事とは主題との関係が薄いため、そこも知りたい方は、KAI-YOUさんが執筆した[アニメ「わたてん」公式Twitterが超有能　作者のつぶやきを光の速さで具現化](https://kai-you.net/article/62793)を見てください。

# フォトモザイクアート生成
それでは、フォトモザイクアート生成をやっていきましょう。
フォトモザイクアート生成に必要なモノとディレクトリ構成は以下のようになります。
```bash
# OS: Windowss11
# Language: Python 3.12.7
./mosaic_of_photos
│  EDSR_x2.pb        # 学習済み超解像モデル（https://github.com/Saafke/EDSR_Tensorflow/blob/master/models/EDSR_x2.pb）
│  mosaic_of_photos.py           # メインスクリプト
│  target.jpg        # モザイク生成対象の画像
│
└─TileImage          # タイル画像を格納するディレクトリ
```

## 使用する画像
LTと同様に高専キャリアの公式Twitter（現X）から写真を使おうと思いましたが、個人情報や権利関係が大変なので、『ご注文はうさぎですか?』でフォトモザイクアートを作成します。

タイル画像データに関しては、Kaggleのデータベースとして公開されている「GochiUsa_Faces」を使用します。
https://www.kaggle.com/datasets/rignak/gochiusa-faces

また、生成する画像（以下、ターゲット画像）は、以下の画像で行います。
![origin](/images/articles/tech-python-mosaic_of_photos/R.jpg)

## プログラム
:::message
このプログラムは、以下のモジュールをインストールする必要があります。
```bash
pip install opencv-python opencv-contrib-python numpy
```

また、以下のリンクから、EDSR_x2.pbを取得してください。
https://github.com/Saafke/EDSR_Tensorflow/blob/master/models/EDSR_x2.pb
:::

プログラムの流れです。
1. 画像の読み込みとリサイズ
2. ターゲット画像の超解像（オプション）
   - apply_super_resolution()の引数scaleで、ターゲット画像の解像度を整数倍にする
3. ターゲット画像のタイルに分割
   - ターゲット画像のタイルに分割の際に、タイル画像の大きさに合わせてターゲット画像の大きさを調整しているので、誤差が出ないように処理をしている
   - 107, 108行目で、タイルの大きさを指定している
4. BGR 平均値に基づいてタイルを選択
5. モザイク画像の保存
   - 「mosaic_of_photos.jpg」で保存される

```python
import os
import cv2
import numpy as np
from typing import List, Tuple

# 定数定義
IMAGES_FOLDER = './TileImage'                   # タイル画像が格納されているフォルダ
TARGET_IMAGE = 'target.jpg'                     # 変換対象の画像
OUTPUT_IMAGE = 'mosaic_of_photos.jpg'           # 生成される画像
SUPER_RES_MODEL = 'EDSR_x2.pb'                  # 超解像モデルのパス
VALID_EXTENSIONS = ('.png', '.jpg', '.jpeg')    # 有効な画像拡張子

def load_images(folder: str, extensions: Tuple[str], tile_width: int, tile_height: int) -> Tuple[List[np.ndarray], List[np.ndarray]]:
    """指定フォルダ内の画像を読み込み、セルサイズにリサイズしBGR平均値を計算する"""
    image_list = []
    bgr_list = []
    
    files = [f for f in os.listdir(folder) if f.endswith(extensions)]
    for filename in files:
        filepath = os.path.join(folder, filename)
        image = cv2.imread(filepath)
        if image is None:
            print(f"Warning: Could not read file {filename}")
            continue
        resized_image = cv2.resize(image, (tile_width, tile_height))
        image_list.append(resized_image)
        bgr_list.append(np.mean(resized_image, axis=(0, 1)))
    
    return image_list, bgr_list

def calculate_closest_image(part_bgr: np.ndarray, image_list: List[np.ndarray], bgr_list: List[np.ndarray]) -> np.ndarray:
    """各セルのBGR平均値に最も近い画像を選択"""
    min_distance = float('inf')
    closest_image = None
    
    for img, bgr in zip(image_list, bgr_list):
        distance = np.sum((part_bgr - bgr) ** 2)
        if distance < min_distance:
            min_distance = distance
            closest_image = img
    
    return closest_image

def apply_super_resolution(image: np.ndarray, model_path: str, scale: int = 2) -> np.ndarray:
    """画像に超解像処理を適用"""
    sr = cv2.dnn_superres.DnnSuperResImpl_create()
    sr.readModel(model_path)
    sr.setModel("edsr", scale)
    return sr.upsample(image)

def create_replacement_image(
    target_image_path: str,
    output_image_path: str,
    image_list: List[np.ndarray],
    bgr_list: List[np.ndarray],
    tile_width: int,
    tile_height: int,
    super_resolution: bool = False,
    model_path: str = SUPER_RES_MODEL
):
    """ターゲット画像を変換し、結果を保存する"""
    target_image = cv2.imread(target_image_path)
    if target_image is None:
        raise FileNotFoundError(f"Target image {target_image_path} could not be read.")
    
    if super_resolution:
        target_image = apply_super_resolution(target_image, model_path)

    height, width, _ = target_image.shape
    resized_width = width // tile_width * tile_width
    resized_height = height // tile_height * tile_height
    resized_image = cv2.resize(target_image, (resized_width, resized_height))
    replacement_image = np.zeros((resized_height, resized_width, 3), dtype=np.uint8)

    for y in range(0, resized_height, tile_height):
        for x in range(0, resized_width, tile_width):
            part = resized_image[y:y+tile_height, x:x+tile_width]
            part_bgr = np.mean(part, axis=(0, 1))
            closest_image = calculate_closest_image(part_bgr, image_list, bgr_list)
            replacement_image[y:y+tile_height, x:x+tile_width] = closest_image
    
    cv2.imwrite(output_image_path, replacement_image)

def main(tile_width: int = 5, tile_height: int = 5, super_resolution: bool = False):
    """メイン処理"""
    try:
        image_list, bgr_list = load_images(IMAGES_FOLDER, VALID_EXTENSIONS, tile_width, tile_height)
        if not image_list:
            raise ValueError("No valid images found in the specified folder.")
        
        create_replacement_image(
            TARGET_IMAGE,
            OUTPUT_IMAGE,
            image_list,
            bgr_list,
            tile_width,
            tile_height,
            super_resolution,
            SUPER_RES_MODEL
        )
        print(f"Replacement image saved to {OUTPUT_IMAGE}.")
    except Exception as e:
        print(f"Error: {e}")

if __name__ == "__main__":
    """タイルサイズと超解像の設定（必要に応じて変更可能）"""
    TILE_WIDTH = 20             # タイルの幅
    TILE_HEIGHT = 20            # タイルの高さ
    USE_SUPER_RESOLUTION = True # 超解像を使用するか
    
    main(TILE_WIDTH, TILE_HEIGHT, USE_SUPER_RESOLUTION)

```

## 結果
タイルの大きさを100, 50, 20と変化させて実行しました。タイルの大きさを小さくしていくと、ターゲット画像の再現度が高まり、様々な画像が使われているモザイクフォトアートの生成がされる傾向が確認できました。

しかし、同じ画像が何度も使われている点や、超解像を使用すると実行時間がかなりかかる等の改良点がありました。今後はその辺りを改良していきたいです。

![100](/images/articles/tech-python-mosaic_of_photos/mosaic_of_photos_1_100.jpg) *タイルサイズが100×100のフォトモザイクアート*
![50](/images/articles/tech-python-mosaic_of_photos/mosaic_of_photos_2_50.jpg) *タイルサイズが50×50のフォトモザイクアート*
![20](/images/articles/tech-python-mosaic_of_photos/mosaic_of_photos_3_20.jpg) *タイルサイズが20×20のフォトモザイクアート*

# まとめ
フォトモザイクアートを生成するプログラムを書きました。
LTでは説明できなかった点がかなりあったので、これで供養出来ました。

# おまけ：高専キャリア2024冬の全国大会の感想
![高専キャリア2024冬の全国大会](/images/articles/tech-python-mosaic_of_photos/event.webp)

## 1日目
「高専卒スタートアップが集合！スタートアップ meetup！」では、高専を卒業した先輩方の企業に関する経験や起業して大変だった点等、企業に関する貴重な話が聞けました。周りに企業経験者が少なかったため、企業に関する考えや視野が広がりました。

「高専キャリア 起業部ピッチ」で発表して、発表の内容が薄かった点や、何が重要なのかが上手く伝わっていなかったのが残念でした。ですが、周りの発表からいい刺激をもらったり、日記のメリットを伝える事ができたので、その点ではプラスです。

## 2日目
「今更聞けないデザイン思考」では、1日目の夜から楽しみにしていたちげさんが行うデザイン思考の講座を受けました。ちげさんは、話が上手く、スライドも見やすかったので、デザイン思考に関する内容がスラスラと頭に入ってきて脱帽しました。あんな発表ができるようになりたいと感じたり、デザイン思考をできるようにしたい等、学びが多い時間でした。また、後日資料公開もあったので、振り仮の意味も込めて再度勉強しなおします。

https://x.com/Chige12_/status/1872941250531536925

「高専キャリア2024 大忘年会＆LT会」も、昨日に引き続き発表しました。資料を発表当日の会場で作ったため、クオリティが低い内容だったので、残念でした。話そうと思っていた内容がノートパソコンでは動かないモノばかりだったので、リモートデスクトップや、リモート環境が欲しくなりました。

## 総括
初めて参加した高専キャリアのイベントでしたが、2日間楽しく参加できたので良かったです。普段関われない学生や大人の方々と話せるいい機会だったので、めちゃくちゃ刺激をもらいました。特に、未踏に挑戦したいだったり、個人ブログ（1月中に作る）の重要性を知れたのは、「自分の帰路を変えたのでは!?」と思える程の刺激でした。

# 参考資料
- [画像でモザイク画を作る(フォトモザイク)](https://tat-pytone.hatenablog.com/entry/2022/04/10/073825)
  - BGR平均値を求める処理を参考にした
- [EDSR : 画像の超解像処理を行う機械学習モデル](https://medium.com/axinc/edsr-%E7%94%BB%E5%83%8F%E3%81%AE%E8%B6%85%E8%A7%A3%E5%83%8F%E5%87%A6%E7%90%86%E3%82%92%E8%A1%8C%E3%81%86%E6%A9%9F%E6%A2%B0%E5%AD%A6%E7%BF%92%E3%83%A2%E3%83%87%E3%83%AB-2842b1d244d)
  - 超解像の書き方を勉強した
- [画像圧縮ツール あっしゅくま](https://imguma.com/)
  - Zennに結果を載せる時に使った