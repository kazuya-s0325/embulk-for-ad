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
   - VSCode、メモ帳、サクラエディタ、秀丸などなんでもOKです
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

## 2. embulkの設定ファイル
### 2.1 【事前知識】ymlファイル
embulkの設定ファイルにはyml（ヤムル）ファイルと呼ばれるものが使われている<br>

#### 2.1.1 ymlファイルとは
構造化されたデータを表現する、ファイルの書き方のルールの一つ<br>
主に設定ファイルなどでよく使用され、あらゆるプログラム言語に対応している<br>
また人間がみても見やすい上に、コンピュータでの解析も容易となっている<br>
ファイル拡張子は".yml"（または".yaml"）で表される<br>
(iniファイルのようなもの、と考えると分かりやすい)

#### 2.1.2 表記方法
以下のように表記される<br>
公式の設定マニュアルは[こちら](https://www.embulk.org/docs/built-in.html)<br>
以下はCSVファイルから読み込む場合の設定例
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
- ハッシュ(連想配列)<br><br>
インデントを挿入すると、キーのネスト構造を表現することが可能(つまり、インデントに意味を持っている)<br>
キーの前にハイフン + 半角スペース挿入で、配列の要素を表現する

#### 2.1.3 SQLServerへの接続
[このリポジトリ](https://github.com/embulk/embulk-input-jdbc/tree/master/embulk-input-sqlserver#example)を参照
<br>
また、Microsoft提供のSQLSever用のJDBC Driverが必要なので[ここからダウンロード](https://learn.microsoft.com/ja-jp/sql/connect/jdbc/download-microsoft-jdbc-driver-for-sql-server?view=sql-server-ver16)しておく<br>
<br>
ダウンロード後は`sqljdbc_12.8`フォルダをCドライブ直下に保存<br>
ディレクトリ階層は以下のような感じ
~~~
C:
└─sqljdbc_12.8
  └─jpn
      ├─auth
      ├─jars
      ├─samples
      └─xa
~~~

### 2.2 設定ymlファイルについて
設定ymlファイルは大きく分けると以下の3つのキーから成り立つ
- in
  - 元データをDBから取得orファイルから取得といった内容を記載
    - データのインプットに関する情報
- filter
  - データの絞り込みやデータ変換などの内容を記載する
    - インプットしたデータの加工方法
- out
  - データをどのように吐き出すかの内容を記載

### 2.3 実行方法
#### 2.3.1 DryRun
書き出し時のイメージを表示する<br>
本実行の前にこれでチェックすることを推奨
~~~
> embulk preview config.yml
~~~

#### 2.3.2 Run
本実行コマンド
~~~
> embulk run config.yml
~~~

## Appendix
https://qiita.com/hiroysato/items/a71669d3e5be2049c238<br>
https://qiita.com/hiroysato/items/861e3689eef430f5e723<br>
https://qiita.com/hiroysato/items/da45e52fb79c39547f69<br>
