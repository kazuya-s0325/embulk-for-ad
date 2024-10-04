# embulk-for-ad
collection of embulk configuration files for data migration

# development
## 1. 環境構築
### 1.1 Java JDKインストール
**Java 8** のJDKをダウンロードし、インストールする<br>
Java 8以外はサポートされていないので注意！<br>
https://www.oracle.com/jp/java/technologies/javase/javase8-archive-downloads.html<br>
<br>
※ダウンロードには会員登録が必要<br>

### 1.2 embulkのダウンロード
v0.9.25をダウンロードする<br>
なぜか他のバージョンだとうまく動かなかった<br>
https://github.com/embulk/embulk/releases/tag/v0.9.25
<br>
<br>
1. "embulk-0.9.25.jar" をクリックしてダウンロード
2. ダウンロードしたファイルを "C:\embulk" を作成し、その配下に配置する
3. ダウンロードした "embulk-0.9.25.jar" を "embulk.bat" にリネームする

### 1.3 JRubyのダウンロード
"jruby-complete-9.1.15.0.jar" を "C:\embulk" にダウンロードする<br>
https://www.jruby.org/files/downloads/9.1.15.0/index.html
~~~
C:\embulk
├─  embulk.bat
└─  jruby-complete-9.1.15.0.jar
~~~
### 1.4 プロパティファイルを作成して配置
1. "C:\embulk" フォルダ内に ".embulk" という名前でフォルダを作成する
2. "C:\embulk\.embulk" 配下に "embulk.properties" という名前でファイルを作成する
   - エクスプローラーからの場合は"右クリック" -> "新規作成" -> "テキスト文書"から作成してください
3. 作成した "embulk.properties" をテキストエディタなどで開く
   - メモ帳、サクラエディタ、秀丸などなんでもOKです
4. 以下の内容を貼り付けて保存
~~~
jruby=file:///C:/embulk/.embulk/jruby-complete-9.1.15.0.jar
~~~
ファイル配置
~~~
C:\embulk
  ├─ embulk.bat
  ├─ jruby-complete-9.1.15.0.jar
  └─.embulk
      └─embulk.properties
~~~
### 1.5 その他必要ライブラリのインストール
下準備として、以下の手順を実行する
1. "C:\embulk" フォルダ内に "Gemfile" という名前でファイルを作成する（拡張子は無しであることに注意）
2. "Gemfile" をテキストエディタで開き、以下内容を貼り付けて保存する
~~~
source 'https://rubygems.org'

gem "jwt", "= 2.3.0"
gem "public_suffix", "= 4.0.7"
gem "multipart-post", "= 2.1.1"
gem "jruby-openssl", "= 0.10.7"

# embulk extensions
# input
gem "embulk-input-sqlserver", "= 0.13.2"

# output
gem "embulk-output-mysql", "= 0.10.5"

# filter
gem "embulk-filter-eval", "= 0.1.0"
gem "embulk-filter-column", "= 0.9.0"
gem 'embulk-filter-concat', '= 0.1.0'
gem 'embulk-filter-typecast', '= 0.3.0'
gem 'embulk-filter-gsub', '= 0.2.0'
~~~
上記が完了したら、"C:\embulk" フォルダをコマンドプロンプトで開く<br>
※Power Shellではないことに注意

コマンドプロンプトで以下コマンドを実行し、必要なGem（Rubyでのライブラリの呼び方）を一括インストール
~~~
embulk bundle
~~~

## 2. 変換時の方針
### 2.1 日付の扱い方
現在の基幹システムに保存されている各日付項目は日本時間となっているが、embulkは出力の際はUTCでの出力となる<br>
よって、embulk側でデフォルトタイムゾーンを日本にしておきJST -> UTCへ変換を行った状態で出力する<br>
UTC -> JSTへの戻しはMySQL側で実施する

### 2.2 日付型 -> String型への変換について
embulk側でString型へ変換<br>
(embulk-filter-typecastを使用する)

### 2.3 他の列の値を参照して値を設定する場合について
カラムAは、カラムBの値がxxの場合に1を設定する...<br>
といったタイプはembulk側での対応が困難<br>
-> この場合は、クエリのcase whenを使用して変換を行う

### 2.4 住所、備考などといった項目について
例えば、受注ヘッダーの依頼主住所といった内容は<br>
住所1～4 -> 住所1～3へ減っている<br>
この際は、予めクエリ側で住所1～4を結合しておき、embulk側で文字列を切り出して分解する形で変換を行う<br>
※もちろん、クエリ側で対応してしまってもOK

### 2.x その他注意点
- embulk自体はshift_jis対応は出来ているが、設定用ファイル(ymlファイル)はUTF-8でないとならない
  - 現在の基幹システムのカラム名が日本語となっているのが厄介
    - 例えば、ymlファイルにクエリを埋め込むといったことを行うと、文字化けしてうまく動かないことが考えられる(要検証)

## 3. embulkの設定ファイル
### 3.1 【事前知識】ymlファイル
embulkの設定ファイルにはymlファイルと呼ばれるものが使われている<br>

#### 3.1.1 ymlファイルとは
構造化されたデータを表現する、ファイルの書き方のルールの一つ<br>
主に設定ファイルなどでよく使用され、あらゆるプログラム言語に対応している<br>
また人間がみても見やすい上に、コンピュータでの解析も容易となっている<br>
ファイル拡張子は".yml"で表される<br>
(iniファイルのようなもの、と考えると分かりやすい)

#### 3.1.2 表記方法
以下のように表記される
例はCSVファイルから読み込む場合
~~~
in:
  type: file
  path_prefix: project\juchu\受注H.csv
  parser:
    default_timezone: "Japan"
    charset: SHIFT_JIS
    newline: CRLF
    type: csv
    delimiter: ","
    quote: '"'
    escape: '"'
    null_string: ""
    trim_if_not_quoted: false
    skip_header_lines: 1
    allow_extra_columns: false
    allow_optional_columns: false
    columns:
      - { name: col1, type: long }
      - { name: col2, type: long }
      - { name: col3, type: long }

filters:
  - type: column
    add_columns:
      - { name: col4, type: string, default: "0" }
      - { name: col5, type: string, default: "" }

out:
  type: file
  path_prefix: C:\embulk\project\juchu\変換テスト_受注H_変換後
  file_ext: csv
  formatter:
    type: csv
    delimiter: ","
    newline: CRLF
    newline_in_field: LF
    charset: SHIFT_JIS
    quote_policy: MINIMAL
    quote: '"'
    null_string: ""
~~~
「キーの名前: 値」という形で表される<br>
値となるものは以下の通り
- 文字列
- 数値
- ブール
- 配列
- ハッシュ(連想配列)
インデントを挿入すると、キーのネスト構造を表現することが可能(つまり、インデントに意味を持っている)<br>
キーの前にハイフン + 半角スペース挿入で、配列の要素を表現する

### 3.2 設定ymlファイルについて
設定ymlファイルは大きく分けると以下の3つのキーから成り立つ
- in
  - 元データをDBから取得orファイルから取得といった内容を記載
    - データのインプットに関する情報
- filter
  - データの絞り込みやデータ変換などの内容を記載する
    - インプットしたデータの加工方法
- out
  - データをどのように吐き出すかの内容を記載

### 3.3 実行方法
#### 3.3.1 DryRun
書き出し時のイメージを表示する<br>
本実行の前にこれでチェックすることを推奨
~~~
> embulk preview config.yml
~~~

#### 3.3.2 Run
本実行コマンド
~~~
> embulk run config.yml
~~~

## 4. 移行時に関して
### 4.1 インプット
embulkでは元データを直接DBへクエリを投げて取得することも可能<br>
一旦Management studioからcsvで落とすか、直接DBへ取得しに行くかは判断必要<br>
ただし、直接つなぐ際は事故防止の為読取専用状態にして繋ぎに行くようにしたい

### 4.2 アウトプット
こちらも、csvで吐き出し以外にも直接MySQLのDBに書き込みに行くことも可能<br>
以下の理由から、できれば直接書き込みに行った方が良いと思われる
- csvの書き出しに時間がかかることが予想される
- nullと""(空白)の区別がつけづらい
- マルチスレッドでの書き出しとなるので、吐き出す際にファイルが分割される(embulkの仕様)

## Appendix
https://qiita.com/hiroysato/items/a71669d3e5be2049c238
https://qiita.com/hiroysato/items/861e3689eef430f5e723
https://qiita.com/hiroysato/items/da45e52fb79c39547f69
