コマンドクックブック

独自のカスタム WP-CLI コマンドを作成することは、見た目よりも簡単です。`wp scaffold package`（リポジトリ）を使用して、コマンド自体以外のすべてを動的に生成できます。

## 概要

WP-CLI の目標は、WordPress 管理画面の完全な代替手段を提供することです。WordPress 管理画面で実行したいアクションには、同等の WP-CLI コマンドが存在するべきです。コマンドは WP-CLI 機能の原子単位です。`wp plugin install`（ドキュメント）はそのようなコマンドの 1 つであり、`wp plugin activate`（ドキュメント）も同様です。コマンドは、複雑なタスクを実行するためのシンプルで正確なインターフェースを提供できるため、WordPress ユーザーにとって有用です。

しかし、WordPress 管理画面は無限の複雑さを持つスイスアーミーナイフのようなものです。このプロジェクトだけでは、すべてのユースケースに対応することはできません。そのため、WP-CLI は一般的な内部コマンドのセットを含むと同時に、サードパーティが独自のコマンドを記述して登録できる豊富な内部 API も提供しています。

WP-CLI コマンドは、スタンドアロンパッケージとして配布することも、WordPress プラグインやテーマにバンドルすることもできます。前者の場合、`wp scaffold package`（リポジトリ）を使用して、コマンド自体以外のすべてを動的に生成できます。

パッケージは WP-CLI にとって、プラグインが WordPress にとっての存在と同じです。WP-CLI パッケージを作成する際のアプローチには明確な違いがあります。WP-CLI は/wp-admin の代替手段として成長し続けていますが、WordPress API の使用方法を検討する前に、まず WP-CLI 内部 API で動作するようにパッケージを記述する必要があることに注意することが重要です。

## コマンドの種類

### バンドルコマンド

- 通常、標準インストールの WordPress で提供される機能をカバーします。ただし、このルールには例外があり、特に`wp search-replace`（ドキュメント）が挙げられます。
- プラグイン、テーマなどの他のコンポーネントに依存しません。
- WP-CLI チームによってメンテナンスされています。

### サードパーティコマンド

- プラグインやテーマで定義できます。
- `wp scaffold package`（リポジトリ）を使用して、スタンドアロンプロジェクトとして簡単にスキャフォールドできます。
- パッケージインデックスでプラグインやテーマとは独立して配布できます。

### すべてのコマンド

- ドキュメント標準に従います。

## コマンドの構造

WP-CLI は、呼び出し可能なクラス、関数、またはクロージャをコマンドとして登録することをサポートしています。`WP_CLI::add_command()`（ドキュメント）は、内部コマンドとサードパーティコマンドの両方の登録に使用されます。

コマンドのシノプシスは、コマンドが受け入れる位置引数と連想引数を定義します。`wp plugin install`のシノプシスを見てみましょう：

```bash
$ wp plugin install
usage: wp plugin install <plugin|zip|url>... [--version=<version>] [--force] [--activate] [--activate-network]
```

この例では、`<plugin|zip|url>...`が受け入れられる位置引数です。実際、`wp plugin install`は同じ位置引数（インストールするプラグインのスラッグ、ZIP、または URL）を複数回受け入れます。`[--version=<version>]`は受け入れられる連想引数の 1 つです。インストールするプラグインのバージョンを指定するために使用されます。また、引数定義の周りの角括弧に注意してください。角括弧は引数がオプションであることを意味します。

WP-CLI には、すべてのコマンドで機能する一連のグローバル引数もあります。たとえば、`--debug`を含めると、コマンド実行時にすべての PHP エラーが表示され、WP-CLI ブートストラッププロセスに追加の詳細情報が追加されます。

## 必須の登録引数

コマンドを登録する際、`WP_CLI::add_command()`には 2 つの引数が必要です：

- `$name`は、WP-CLI の名前空間内のコマンド名です（例：`plugin install`または`post list`）。
- `$callable`は、呼び出し可能なクラス、関数、またはクロージャとしてのコマンドの実装です。

次の例では、`wp foo`の各インスタンスは機能的に同等です：

```php
// 1. コマンドは関数です
function foo_command( $args ) {
    WP_CLI::success( $args[0] );
}
WP_CLI::add_command( 'foo', 'foo_command' );

// 2. コマンドはクロージャです
$foo_command = function( $args ) {
    WP_CLI::success( $args[0] );
}
WP_CLI::add_command( 'foo', $foo_command );

// 3. コマンドはクラスのメソッドです
class Foo_Command {
    public function __invoke( $args ) {
        WP_CLI::success( $args[0] );
    }
}
WP_CLI::add_command( 'foo', 'Foo_Command' );

// 4. コマンドはコンストラクタ引数を持つクラスのメソッドです
class Foo_Command {
    protected $bar;
    public function __construct( $bar ) {
        $this->bar = $bar;
    }
    public function __invoke( $args ) {
        WP_CLI::success( $this->bar . ':' . $args[0] );
    }
}
$instance = new Foo_Command( 'Some text' );
WP_CLI::add_command( 'foo', $instance );
```

重要なことに、クラスは関数やクロージャとは少し異なる動作をします：

- クラスのパブリックメソッドは、コマンドのサブコマンドとして登録されます。たとえば、上記の例では、クラス Foo のメソッド`bar()`は`wp foo bar`として登録されます。しかし...
- `__invoke()`はマジックメソッドとして扱われます。クラスが`__invoke()`を実装している場合、コマンド名はそのメソッドに登録され、そのクラスの他のメソッドはコマンドとして登録されません。

**注意：** 歴史的に、WP-CLI は拡張するための基本クラス`WP_CLI_Command`を提供していましたが、このクラスを拡張する必要はなく、コマンドの動作も変わりません。

すべてのコマンドは、独自のトップレベル名前空間（例：`wp foo`）に登録することも、既存の名前空間のサブコマンドとして登録することもできます（例：`wp core foo`）。後者の場合、コマンド定義の一部として既存の名前空間を含めるだけです。

```php
class Foo_Command {
    public function __invoke( $args ) {
        WP_CLI::success( $args[0] );
    }
}
WP_CLI::add_command( 'core foo', 'Foo_Command' );
```

## クイックで簡単な実行

一回限りのタスク用の短いスクリプトを作成し、`WP_CLI::add_command()`で正式に登録する必要がない場合、`wp eval-file`が役立ちます（ドキュメント）。

`simple-command.php`ファイルがあるとします：

```php
<?php
WP_CLI::success( "The script has run!" );
```

「コマンド」は`wp eval-file simple-command.php`で実行できます。コマンドが WordPress に依存しない場合、または WordPress が利用できない場合、`--skip-wordpress`フラグを使用して WordPress の読み込みを回避できます。

## オプションの登録引数

WP-CLI は、コマンドのオプション引数を登録する 2 つの方法をサポートしています：呼び出し可能な PHPDoc を通じて、または`WP_CLI::add_command()`の 3 番目の`$args`パラメータとして渡す方法です。

### PHPDoc による注釈

典型的な WP-CLI クラスは次のようになります：

```php
<?php
/**
 * サンプルコマンドを実装します。
 */
class Example_Command {

    /**
     * 挨拶を出力します。
     *
     * ## OPTIONS
     *
     * <name>
     * : 挨拶する人の名前。
     *
     * [--type=<type>]
     * : 成功またはエラーで挨拶するかどうか。
     * ---
     * default: success
     * options:
     *   - success
     *   - error
     * ---
     *
     * ## EXAMPLES
     *
     *     wp example hello Newman
     *
     * @when after_wp_load
     */
    function hello( $args, $assoc_args ) {
        list( $name ) = $args;

        // タイプでメッセージを出力
        $type = $assoc_args['type'];
        WP_CLI::$type( "Hello, $name!" );
    }
}

WP_CLI::add_command( 'example', 'Example_Command' );
```

このコマンドの PHPDoc は 3 つの方法で解釈されます：

### Shortdesc

shortdesc は PHPDoc の最初の行です：

    /**
     * Prints a greeting.

### Longdesc

longdesc は PHPDoc の中間部分です：

     * ## OPTIONS
     *
     * <name>
     * : The name of the person to greet.
     *
     * [--type=<type>]
     * : Whether or not to greet the person with success or error.
     * ---
     * default: success
     * options:
     *   - success
     *   - error
     * ---
     *
     * ## EXAMPLES
     *
     *     wp example hello Newman

longdesc で定義されたオプションは、コマンドのシノプシスとして解釈されます：

- `<name>`は必須の位置引数です。`<name>...`に変更すると、コマンドは 1 つ以上の位置引数を受け入れることができます。`[<name>]`に変更すると、位置引数がオプションになり、最後に`[<name>...]`に変更すると、コマンドは複数のオプション位置引数を受け入れることができます。
- `[--type=<type>]`は、デフォルトが'success'で、'success'または'error'を受け入れるオプションの連想引数です。`[--error]`に変更すると、引数はオプションのブール値フラグとして動作します。
- `[--field[=<value>]]`により、オプション引数を値ありまたは値なしで使用できます。この例としては、`--skip-plugins[=<plugins>]`のようなグローバルパラメータを使用する場合があり、すべてのプラグインの読み込みをスキップするか、カンマ区切りのプラグインリストをスキップできます。

**注意：** 任意/無制限の数のオプション連想引数を受け入れるには、注釈`[--<field>=<value>]`を使用します。例：

     * [--<field>=<value>]
     * : Allow unlimited number of associative parameters.

コマンドのシノプシスは、実装に渡す前に引数を検証するために使用されます。

longdesc は、ヘルプコマンドを呼び出すときにも表示されます（例：`wp help example hello`）。その構文は Markdown Extra で、WP-CLI で処理される方法についての追加の注意事項は次のとおりです：

- longdesc は一般的に自由形式のテキストとして扱われます。OPTIONS と EXAMPLES のセクション名は強制されず、一般的で推奨されるだけです。
- セクション名（`## NAME`）は色付けされ、インデントなしで印刷されます。
- その他はすべて 2 文字インデントされ、オプションの説明はさらに 2 文字インデントされます。
- ワードラッピングは少し複雑です。各行で可能な限り多くのスペースを利用し、次の行に 1 つまたは 2 つの単語が残るようなワードラッピングのアーティファクトを避けたい場合は、次のルールに従ってください：
  - コロンとスペースの後に 75 文字でオプションの説明をハードラップします。
  - その他はすべて 90 文字でハードラップします。

コマンドドキュメントのフォーマット方法の詳細については、WP-CLI のドキュメント標準を参照してください。

### Docblock タグ

これは最後のセクションで、longdesc の直後に始まります：

     * @when after_wp_load
     */

定義されたタグのリストは次のとおりです：

#### @subcommand

メソッド名をサブコマンドの名前にできない場合があります。たとえば、`list`というメソッド名にすることはできません。`list`は PHP の予約キーワードだからです。

そのような場合、`@subcommand`タグが役立ちます：

```php
/**
 * @subcommand list
 */
function _list( $args, $assoc_args ) {
    ...
}

/**
 * @subcommand do-chores
 */
function do_chores( $args, $assoc_args ) {
    ...
}
```

#### @alias

`@alias`タグを使用すると、サブコマンドを呼び出す別の方法を追加できます。例：

```php
/**
 * @alias hi
 */
function hello( $args, $assoc_args ) {
    ...
}
```

```bash
$ wp example hi Joe
Success: Hello, Joe!
```

#### @when

これは、WP-CLI にコマンドをいつ実行するかを伝える特別なタグです。登録されているすべての WP-CLI フックをサポートします。

ほとんどの WP-CLI コマンドは、WordPress が読み込まれた後に実行されます。コマンドのデフォルトの動作は次のとおりです：

```
@when after_wp_load
```

WordPress が読み込まれる前に WP-CLI コマンドを実行するには、次を使用します：

```
@when before_wp_load
```

ほとんどの WP-CLI フックは WordPress が読み込まれる前に発火することに注意してください。コマンドがプラグインやテーマから読み込まれる場合、`@when`は実質的に無視されます。

プラグインやテーマから読み込まれるコマンドで使用する場合、効果はありません。その時点では、WordPress 自体がすでに読み込まれているためです。

### WP_CLI::add_command()の 3 番目の$args パラメータ

PHPDoc でサポートされている各設定オプションは、代わりにコマンド登録の 3 番目の引数として渡すことができます：

```php
$hello_command = function( $args, $assoc_args ) {
    list( $name ) = $args;
    $type = $assoc_args['type'];
    WP_CLI::$type( "Hello, $name!" );
    if ( isset( $assoc_args['honk'] ) ) {
        WP_CLI::log( 'Honk!' );
    }
};
WP_CLI::add_command( 'example hello', $hello_command, array(
    'shortdesc' => 'Prints a greeting.',
    'synopsis' => array(
        array(
            'type'        => 'positional',
            'name'        => 'name',
            'description' => 'The name of the person to greet.',
            'optional'    => false,
            'repeating'   => false,
        ),
        array(
            'type'        => 'assoc',
            'name'        => 'type',
            'description' => 'Whether or not to greet the person with success or error.',
            'optional'    => true,
            'default'     => 'success',
            'options'     => array( 'success', 'error' ),
        ),
        array(
            'type'     => 'flag',
            'name'     => 'honk',
            'optional' => true,
        ),
    ),
    'when' => 'after_wp_load',
    'longdesc' =>   '## EXAMPLES' . "\n\n" . 'wp example hello Newman',
) );
```

`longdesc`属性は、シノプシスから生成されたオプションの説明に追加されるため、この引数は使用例を追加するのに最適です。シノプシスがない場合、`longdesc`属性はそのまま説明として使用されます。

## コマンドの内部

コマンドを登録する方法がわかったので、自由にコマンドを作成できます。コールバック内で、コマンドは何でも実行できます。

### 引数の受け入れ

ランタイム引数を処理するには、呼び出し可能な関数に 2 つのパラメータを追加する必要があります：`$args`と`$assoc_args`です。

```php
function hello( $args, $assoc_args ) {
    /* ここにコードを記述 */
}
```

`$args`変数はすべての位置引数を格納します：

```bash
$ wp example hello Joe Doe
```

```php
WP_CLI::line( $args[0] ); // Joe
WP_CLI::line( $args[1] ); // Doe
```

`$assoc_args`変数は、`--key=value`、`--flag`、または`--no-flag`のように定義されたすべての引数を格納します：

```bash
$ wp example hello --name='Joe Doe' --verbose --no-option
```

```php
WP_CLI::line( $assoc_args['name'] ); // Joe Doe
WP_CLI::line( $assoc_args['verbose'] ); // true
WP_CLI::line( $assoc_args['option'] ); // false
```

また、引数のタイプを組み合わせることもできます：

```bash
$ wp example hello --name=Joe foo --verbose bar
```

```php
WP_CLI::line( $assoc_args['name'] ); // Joe
WP_CLI::line( $assoc_args['verbose'] ); // true
WP_CLI::line( $args[0] ); // foo
WP_CLI::line( $args[1] ); // bar
```

### WP-CLI 内部 API の効果的な再利用

例として、マルチサイトネットワークで使用されていないすべてのテーマを見つけるタスクを担当したとします（#2523）。WordPress 管理画面でこのタスクを手動で実行する必要がある場合、おそらく数時間、場合によっては数日かかるでしょう。しかし、WP-CLI コマンドの記述に慣れている場合、15 分以内にタスクを完了できます。

このようなコマンドの例は次のとおりです：

```php
/**
 * マルチサイトネットワークで使用されていないテーマを見つけます。
 *
 * ネットワーク上のすべてのサイトを反復処理して、どのサイトでも有効になっていない
 * テーマを見つけます。
 */
$find_unused_themes_command = function() {
    $response = WP_CLI::launch_self( 'site list', array(), array( 'format' => 'json' ), false, true );
    $sites = json_decode( $response->stdout );
    $unused = array();
    $used = array();
    foreach( $sites as $site ) {
        WP_CLI::log( "Checking {$site->url} for unused themes..." );
        $response = WP_CLI::launch_self( 'theme list', array(), array( 'url' => $site->url, 'format' => 'json' ), false, true );
        $themes = json_decode( $response->stdout );
        foreach( $themes as $theme ) {
            if ( 'no' == $theme->enabled && 'inactive' == $theme->status && ! in_array( $theme->name, $used ) ) {
                $unused[ $theme->name ] = $theme;
            } else {
                if ( isset( $unused[ $theme->name ] ) ) {
                    unset( $unused[ $theme->name ] );
                }
                $used[] = $theme->name;
            }
        }
    }
    WP_CLI\Utils\format_items( 'table', $unused, array( 'name', 'version' ) );
};
WP_CLI::add_command( 'find-unused-themes', $find_unused_themes_command, array(
    'before_invoke' => function(){
        if ( ! is_multisite() ) {
            WP_CLI::error( 'This is not a multisite installation.' );
        }
    },
) );
```

このコマンドが目標を達成するために使用する内部 API を見てみましょう：

- `WP_CLI::add_command()`（ドキュメント）は、`find-unused-themes`コマンドを`$find_unused_themes_command`クロージャに登録するために使用されます。`before_invoke`引数により、コマンドがマルチサイトインストールで実行されていることを確認し、そうでない場合はエラーを出すことができます。
- `WP_CLI::error()`（ドキュメント）は、適切にフォーマットされたエラーメッセージを表示して終了します。
- `WP_CLI::launch_self()`（ドキュメント）は、最初にすべてのサイトのリストを取得するプロセスを生成し、その後、特定のサイトのテーマのリストを取得するために使用されます。
- `WP_CLI::log()`（ドキュメント）は、エンドユーザーに情報出力を表示します。
- `WP_CLI\Utils\format_items()`（ドキュメント）は、コマンドが検出を完了した後、使用されていないテーマのリストを表示します。

## ヘルプのレンダリング

コマンドの PHPDoc（または登録された定義）は、ヘルプコマンドを使用してレンダリングされます。出力は次の順序で表示されます：

1. 短い説明
2. シノプシス
3. 長い説明（OPTIONS、EXAMPLES など）
4. グローバルパラメータ

## テストの記述

WP-CLI は Behat ベースのテストフレームワークを使用しており、あなたも使用すべきです。Behat は WP-CLI コマンドに最適な選択です。理由は次のとおりです：

- 新しいテストを簡単に記述できるため、実際に記述されます。
- テストは、ユーザーがコマンドとインターフェースするのと同じ方法でコマンドとインターフェースします。

Behat テストは、プロジェクトの`features/`ディレクトリに配置されます。`features/cli-info.feature`からの例は次のとおりです：

```gherkin
Feature: Review CLI information

  Scenario: Get the path to the packages directory
    Given an empty directory

    When I run `wp cli info --format=json`
    Then STDOUT should be JSON containing:
      """
      {"wp_cli_packages_dir_path":"/tmp/wp-cli-home/.wp-cli/packages/"}
      """

    When I run `WP_CLI_PACKAGES_DIR=/tmp/packages wp cli info --format=json`
    Then STDOUT should be JSON containing:
      """
      {"wp_cli_packages_dir_path":"/tmp/packages/"}
      """
```

機能テストは通常、次のパターンに従います：

- Given（背景）
- When（ユーザーが特定のアクションを実行）
- Then（最終結果は X（および Y と Z）であるべき）

納得しましたか？`wp-cli/scaffold-package-command`にアクセスして始めましょう。

## 配布

誇りに思うコマンドを作成したので、世界と共有する時が来ました。これを行う一般的な方法は 2 つあります。

### プラグインやテーマに含める

WP-CLI コマンドを共有する 1 つの方法は、プラグインやテーマにパッケージ化することです。多くの人は、`WP_CLI`定数の存在に基づいて、コマンドを条件付きで読み込み（および登録）しています。

```php
if ( defined( 'WP_CLI' ) && WP_CLI ) {
    require_once dirname( __FILE__ ) . '/inc/class-plugin-cli-command.php';
}
```

### スタンドアロンコマンドとして配布

スタンドアロンの WP-CLI コマンドは、任意の git リポジトリ、ZIP ファイル、またはフォルダからインストールできます。唯一の技術的要件は、autoload 宣言を含む有効な`composer.json`ファイルを含めることです。プロジェクトを明示的に WP-CLI パッケージとして区別するために、`"type": "wp-cli-package"`を含めることを推奨します。

サーバーコマンドからの完全な`composer.json`の例は次のとおりです：

```json
{
  "name": "wp-cli/server-command",
  "description": "Start a development server for WordPress",
  "type": "wp-cli-package",
  "homepage": "https://github.com/wp-cli/server-command",
  "license": "MIT",
  "authors": [
    {
      "name": "Package Maintainer",
      "email": "packagemaintainer@homepage.com",
      "homepage": "https://www.homepage.com"
    }
  ],
  "require": {
    "php": ">=5.3.29"
  },
  "autoload": {
    "files": ["command.php"]
  }
}
```

`command.php`を読み込む autoload 宣言に注意してください。

プロジェクトリポジトリに有効な`composer.json`ファイルを追加すると、WP-CLI ユーザーは、選択した保存場所からパッケージマネージャー経由でそれを取得できます。保存場所と、パッケージマネージャー経由でインストールするための対応する構文の例をいくつか示します：

### Git リポジトリ

git リポジトリにあるパッケージをインストールするには、パッケージインストールコマンドに git リポジトリへの HTTPS リンクまたは SSH リンクを提供できます。

```bash
# HTTPSリンクを使用してパッケージをインストール
$ wp package install https://github.com/wp-cli/server-command.git

# SSHリンクを使用してパッケージをインストール
$ wp package install git@github.com:wp-cli/server-command.git
```

### ZIP ファイル

ZIP ファイルからパッケージをインストールするには、`wp package install`コマンドにそのファイルへのパスを提供します。

```bash
# ZIPファイルを使用してパッケージをインストール
$ wp package install ~/Downloads/server-command-main.zip
```
