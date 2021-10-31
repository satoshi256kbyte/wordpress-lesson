Wordpress勉強会①

#　本テキストの対象者

Wordpress自体はある程度触ったことがあり、
何らかのカスタマイズをしようとしている開発者。

# Wordpressのローカル開発環境

Wordpressのローカル開発環境を作る場合、
5〜6年前はVagrantベースのVCCWが一番楽でした。
現在はDocker compose を使うのが最も楽だと思います。
http://vccw.cc/
https://docs.docker.jp/compose/wordpress.html

ただし、現実にはWordpressでDockerの使用を採用させるのは結構難しいので、
マニュアルセットアップに慣れるためにあえて手動で環境を作るのもアリです。

# Wordpressの処理の流れ

https://www.findxfine.com/programming/wp/995562146.html
基本的にWordpressは一本道で、
最後にページのタイプに応じたテンプレートファイルを呼ぶという仕組みになっています。

通常のブログ記事ならば、
wordpress/wp-content/themes/twentytwentyone/template-parts/content/content-single.php

固定ページならば、
wordpress/wp-content/themes/twentytwentyone/template-parts/content/content-page.php

twentytwentyoneというのは現行Wordpress(2021/10/31時点：5.8.1)のデフォルトテーマ名です。
Wordpressのテーマは管理画面の「外観」から変更できます。

# はじめてのカスタマイズ、SQLログを出してみる

WordpressにはSQLログ出力機能はありません。（驚くでしょう？）
デバッグログ・・・というかエラーログは出すことができるので、それを利用してログ出力を行います。

## ログ出力をONにする

wp-config.phpに次の記述を追加します。

```
define( 'WP_DEBUG', true );
define( 'WP_DEBUG_LOG', true );
```

これでエラーログを出せるようになります。
ただし、これだけだとエラーログが画面に出てしまうので、

```
define( 'WP_DEBUG_DISPLAY', false );
```

これを追加します。

設定は、最後の行の前にしてください。
最後の行は設定を反映させるためのコードなので、これより下に書くと効きません。
また、WP_DEBUGは上の方でFalseを指定されることがあるのでその場合は消しておいてください。
こんな感じですね。

```
define( 'WP_DEBUG', true );
define( 'WP_DEBUG_LOG', true );
define( 'WP_DEBUG_DISPLAY', false );

/** Sets up WordPress vars and included files. */
require_once ABSPATH . 'wp-settings.php';
```

これでwp-content/debug.logにログが出力されるようになります。
といってもこの時点ではログを出す人がいないので何も出ませんが。

## SQLログを出すようカスタマイズ

wordpress/wp-content/themes/twentytwentyone/functions.php
の一番下に、下記コードを貼り付けてください。

```
function sql_dump( $query )
{
    error_log( $query );
    return $query;
}
add_filter( 'query', 'sql_dump' );
```

この後、Wordpresの画面を操作すると
wp-contet/debug.logにSQLログが出るようになります。

## カスタマイズの仕組み

```add_filter( 'query', 'sql_dump' );```
が肝です。

wp-dp.phpに
```$query = apply_filters( 'query', $query );```
という記述があります。
この記述の意味は、
*add_filterの第一引数で、'query'が指定されているものがあれば、
add_filterの第二引数で指定された関数を呼ぶ。
この時、指定された関数にadd_filterの第二引数を渡しその戻り値を受け取る。*
です。

このように、add_filterを使えばWordpresの処理に独自ロジックを差し込むこどができます。
どこに差し込めるかは、Wordpreess本体または各種プラグインのソースをapply_filtersで検索することで見つけられます。

# add_filterとadd_action

add_filterの他に、add_actionを使うことでも独自ロジックを差し込むこどができます。
それぞれ、apply_filters・do_actionと対になります。

add_filterとadd_actionの違いは戻り値があるかないかです。
呼び出し元を見ると違いがわかります。

wp-login.phpより

```
/**
	* Enqueue scripts and styles for the login page.
	*
	* @since 3.1.0
	*/
do_action( 'login_enqueue_scripts' );

/**
	* Fires in the login page header after scripts are enqueued.
	*
	* @since 2.1.0
	*/
do_action( 'login_head' );

$login_header_url = __( 'https://wordpress.org/' );

/**
	* Filters link URL of the header logo above login form.
	*
	* @since 2.1.0
	*
	* @param string $login_header_url Login header logo URL.
	*/
$login_header_url = apply_filters( 'login_headerurl', $login_header_url );
```

単に処理を差し込むならdo_action >>> add_actionを使い、
データやHTMLなど出力結果を加工するならばapply_filters >>> add_filterを使うとよいでしょう。

## 補足

先程のSQLログのカスタマイズはこれでもOKです。

```
add_filter( 'query', function ( $query ){
    error_log( $query );
    return $query;
});
```