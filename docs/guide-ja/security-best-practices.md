セキュリティのベストプラクティス
================================

下記において、一般的なセキュリティの指針を復習し、Yii を使ってアプリケーションを開発するときに脅威を回避する方法を説明します。
これらの原則のほとんどのものは Yii に固有のものではなく、ウェブサイトまたはソフトウェアの開発一般に適用されるものです。
従って、これらの原則の背後にある一般的な考え方について、さらに参照すべき文書へのリンクをも含んでいます。

基本的な指針
------------

どのようなアプリケーションが開発されているかに関わらず、セキュリティに関しては二つの大きな指針が存在します。

1. 入力をフィルタする。
2. 出力をエスケープする。


### 入力をフィルタする

入力をフィルタするとは、入力値は決して安全なものであると見なさず、取得した値が実際に許可さていれる値に含まれるか否かを常にチェックしなければならない、ということを意味します。
例えば、並べ替えが三つのフィールド `title`、`created_at` および `status` によって実行され、フィールドの名前がユーザの入力によって提供されるものであることを知っている場合、取得した値を受信するその場でチェックする方が良い、ということです。
基本的な PHP の形式では、次のようなコードになります。

```php
$sortBy = $_GET['sort'];
if (!in_array($sortBy, ['title', 'created_at', 'status'])) {
	throw new Exception('sort の値が不正です。');
}
```

Yii においては、たいていの場合、同様のチェックを行うために [フォームのバリデーション](input-validation.md) を使うことになるでしょう。

このトピックについて更に読むべき文書:

- <https://www.owasp.org/index.php/Data_Validation>
- <https://www.owasp.org/index.php/Input_Validation_Cheat_Sheet>


### 出力をエスケープする

データを使用するコンテキストに応じて、出力をエスケープしなければなりません。
つまり、HTML のコンテキストでは、`<` や `>` などの特殊な文字をエスケープしなければなりません。
JavaScript や SQL のコンテキストでは、対象となる文字は別のセットになります。
全てを手動でエスケープするのは間違いを生じやすいことですから、Yii は異なるコンテキストに応じたエスケープを実行するためのさまざまなツールを提供しています。

このトピックについて更に読むべき文書:

- <https://www.owasp.org/index.php/Command_Injection>
- <https://www.owasp.org/index.php/Code_Injection>
- <https://www.owasp.org/index.php/Cross-site_Scripting_%28XSS%29>


SQL インジェクションを回避する
------------------------------

SQL インジェクションは、次のように、エスケープされていない文字列を連結してクエリテキストを構築する場合に発生します。

```php
$username = $_GET['username'];
$sql = "SELECT * FROM user WHERE username = '$username'";
```

正しいユーザ名を提供する代りに、攻撃者は `'; DROP TABLE user; --` のような文字列をあなたのアプリケーションに与えることが出来ます。
結果として構築される SQL は次のようになります。

```sql
SELECT * FROM user WHERE username = ''; DROP TABLE user; --'
```

これは有効なクエリで、空のユーザ名を持つユーザを探してから、`user` テーブルを削除します。
おそらく、ウェブサイトは破壊されて、データは失われることになります (定期的なバックアップは設定済みですよね、ね? )。

Yii においては、ほとんどのデータベースクエリは、PDO のプリペアドステートメントを適切に使用する [アクティブレコード](db-active-record.md) を経由して実行されます。
プリペアドステートメントの場合は、上で説明したようなクエリの改竄は不可能です。

それでも、[生のクエリ](db-dao.md) や [クエリビルダ](db-query-builder.md) を必要とする場合はあります。
その場合には、データを渡すための安全な方法を使わなければなりません。
データをカラムの値として使う場合は、プリペアドステートメントを使うことが望まれます。

```php
// query builder
$userIDs = (new Query())
    ->select('id')
    ->from('user')
    ->where('status=:status', [':status' => $status])
    ->all();

// DAO
$userIDs = $connection
    ->createCommand('SELECT id FROM user where status=:status')
    ->bindValues([':status' => $status])
    ->queryColumn();
```

データがカラム名やテーブル名を指定するために使われる場合は、事前定義された一連の値だけを許可するのが最善の方法です。
 
```php
function actionList($orderBy = null)
{
    if (!in_array($orderBy, ['name', 'status'])) {
        throw new BadRequestHttpException('name と status だけを並べ替えに使うことが出来ます。')
    }
    
    // ...
}
```

それが不可能な場合は、テーブル名とカラム名をエスケープしなければなりません。
Yii はそういうエスケープのための特別な文法を持っており、それを使うと、サポートされている全てのデータベースに対して同じ方法でエスケープすることが出来ます。

```php
$sql = "SELECT COUNT([[$column]]) FROM {{table}}";
$rowCount = $connection->createCommand($sql)->queryScalar();
```

この文法の詳細は、[テーブルとカラムの名前を引用符で囲む](db-dao.md#quoting-table-and-column-names) で読むことが出来ます。

このトピックについて更に読むべき文書:

- <https://www.owasp.org/index.php/SQL_Injection>


XSS を回避する
--------------

XSS すなわちクロスサイトスクリプティングは、ブラウザに HTML を出力する際に、出力が適切にエスケープされていないと発生します。
例えば、ユーザ名を入力できるフォームで `Alexander` の代りに `<script>alert('Hello!');</script>` と入力した場合、ユーザ名をエスケープせずに出力している全てのページでは、JavaScript `alert('Hello!');` が実行されて、ブラウザにアラートボックスがポップアップ表示されます。
ウェブサイト次第では、そのようなスクリプトを使って、無害なアラートではなく、あなたの名前を使ってメッセージを送信したり、さらには銀行取引を実行したりすることが可能です。

XSS の回避は、Yii においてはとても簡単です。一般に、二つのケースがあります。

1. データを平文テキストとして出力したい。
2. データを HTML として出力したい。

平文テキストしか必要でない場合は、エスケープは次のようにとても簡単です。


```php
<?= \yii\helpers\Html::encode($username) ?>
```

HTML である場合は、HtmlPurifier から助けを得ることが出来ます。

```php
<?= \yii\helpers\HtmlPurifier::process($description) ?>
```

HtmlPurifier の処理は非常に重いので、キャッシュを追加することを検討してください。

このトピックについて更に読むべき文書:

- <https://www.owasp.org/index.php/Cross-site_Scripting_%28XSS%29>


CSRF を回避する
---------------

CSRF は、クロスサイトリクエストフォージェリ (cross-site request forgery) の略称です。
多くのアプリケーションは、ユーザのブラウザから来るリクエストはユーザ自身によって発せられたものだと仮定しているけれども、その仮定は間違っているかもしれない ... というのが CSRF の考え方です。

例えば、`an.example.com` というウェブサイトが `/logout` という URL を持っており、この URL を単純な GET でアクセスするとユーザをログアウトさせるようになっているとします。
ユーザ自身によってこの URL がリクエストされる限りは何も問題はありませんが、ある日、悪い奴が、ユーザが頻繁に訪れるフォーラムに `<img src="http://an.example.com/logout">` というリンクを含むコンテントを何とかして投稿することに成功します。
ブラウザは画像のリクエストとページのリクエストの間に何ら区別を付けませんので、ユーザがそのような `img` タグを含むページを開くとブラウザはその URL に対して GET リクエストを送信します。
そして、ユーザが `an.example.com` からログアウトされてしまうことになる訳です。

これは基本的な考え方です。ユーザがログアウトされるぐらいは大したことではない、と言うことも出来るでしょう。
しかし、悪い奴は、この考え方を使って、もっとひどいことをすることも出来ます。
例えば、`http://an.example.com/purse/transfer?to=anotherUser&amount=2000` という URL を持つウェブサイトがあると考えて見てください。
この URL に GET リクエストを使ってアクセスすると、権限を持つユーザアカウントから `anotherUser` に $2000 が送金されるのです。
私たちは、ブラウザは画像をロードするのに常に GET リクエストを使う、ということを知っていますから、この URL が POST リクエストだけを受け入れるようにコードを修正することは出来ます。
しかし残念なことに、それで問題が解決する訳ではありません。
攻撃者は `<img>` タグの代りに何らかの JavaScript コードを書いて、その URL に対する POST リクエストの送信を可能にすることが出来ます。

CSRF を回避するためには、常に次のことを守らなければなりません。

1. HTTP の規格、すなわち、GET はアプリケーションの状態を変更すべきではない、という規則に従うこと。
2. Yii の CSRF 保護を有効にしておくこと。

場合によっては、コントローラやアクションの単位で CSRF 検証を無効化する必要があることがあるでしょう。
これは、そのプロパティを設定することによって達成することが出来ます。

```php
namespace app\controllers;

use yii\web\Controller;

class SiteController extends Controller
{
    public $enableCsrfValidation = false;

    public function actionIndex()
    {
        // CSRF 検証はこのアクションおよびその他のアクションに対して適用されない
    }

}
```

特定のアクションに対して CSRF 検証を無効化したいときは、次のようにすることが出来ます。
```php
namespace app\controllers;

use yii\web\Controller;

class SiteController extends Controller
{
    public function beforeAction($action)
    {
        // ... ここで何らかの条件に従って `$this->enableCsrfValidation` を設定する ...
        // 親のメソッドを呼ぶ。プロパティが true であれば、その中で CSRF がチェックされる。
        return parent::beforeAction($action);
    }
}
```

[スタンドアロンアクション](structure-controllers.md#standalone-actions) における CSRF 検証の無効化は `init()` メソッドの中で行わなければなりません。
このコードは `beforeRun()` メソッドに置いてはいけません。そこでは効果がありません。

```php
<?php

namespace app\components;

use yii\base\Action;

class ContactAction extends Action
{
    public function init()
    {
        parent::init();
        $this->controller->enableCsrfValidation = false;
    }

    public function run()
    {
          $model = new ContactForm();
          $request = Yii::$app->request;
          if ($request->referrer === 'yiipowered.com'
              && $model->load($request->post())
              && $model->validate()
          ) {
              $model->sendEmail();
          }
    }
}
```

> Warning: CSRF を無効化すると、あらゆるサイトから POST リクエストをあなたのサイトに送信することが出来るようになります。
  その場合には、IP アドレスや秘密のトークンをチェックするなど、追加の検証を実装することが重要です。

このトピックについて更に読むべき文書:

- <https://www.owasp.org/index.php/CSRF>

ファイルの曝露を回避する
------------------------

デフォルトでは、サーバのウェブルートは、`index.php` がある `web` ディレクトリを指すように意図されています。
共有ホスティング環境の場合、それをすることが出来ずに、全てのコード、構成情報、ログをサーバのウェブルートの下に置かなくてはならないことがあり得ます。

そういう場合には、`web` 以外の全てに対してアクセスを拒否することを忘れないでください。
それも出来ない場合は、アプリケーションを別の場所でホストすることを検討してください。

本番環境ではデバッグ情報とデバッグツールを無効にする
----------------------------------------------------

デバッグモードでは、Yii は極めて多くのエラー情報を出力します。これは確かに開発には役立つものです。
しかし、実際の所、これらの饒舌なエラー情報は、攻撃者にとっても、データベース構造、構成情報の値、コードの断片などを曝露してくれる重宝なものです。
本番でのアプリケーションにおいては、決して `index.php` の `YII_DEBUG` を `true` にして走らせてはいけません。

本番環境では Gii やデバッグツールバーを決して有効にしてはいけません。
これらを有効にすると、データベース構造とコードに関する情報を得ることが出来るだけでなく、コードを Gii によって生成したもので書き換えることすら出来てしまいます。

デバッグツールバーは本当に必要でない限り本番環境では使用を避けるべきです。
これはアプリケーションと構成情報の全ての詳細を曝露することが出来ます。
どうしても必要な場合は、あなたの IP だけに適切にアクセス制限されていることを再度チェックしてください。

このトピックについて更に読むべき文書:

- <https://www.owasp.org/index.php/Exception_Handling>
- <https://www.owasp.org/index.php/Top_10_2007-Information_Leakage>


TLS によるセキュアな接続を使う
------------------------------

Yii が提供する機能には、クッキーや PHP セッションに依存するものがあります。
これらのものは、接続が侵害された場合には、脆弱性となり得ます。
アプリケーションが TLS (しばしば [SSL](https://ja.wikipedia.org/wiki/Transport_Layer_Security) と呼ばれます)
によるセキュアな接続を使用している場合は、この危険性を減少させることが出来ます。

その設定の仕方については、あなたのウェブサーバのドキュメントの指示を参照してください。
H5BP プロジェクトが提供する構成例を参考にすることも出来ます。

- [Nginx](https://github.com/h5bp/server-configs-nginx)
- [Apache](https://github.com/h5bp/server-configs-apache).
- [IIS](https://github.com/h5bp/server-configs-iis).
- [Lighttpd](https://github.com/h5bp/server-configs-lighttpd).

サーバの構成をセキュアにする
----------------------------

この節の目的は、Yii ベースのウェブサイトをホストするサーバの構成を作成するときに、
考慮に入れなければならないリスクに照明を当てることにあります。
ここで触れられる問題点以外にも、セキュリティに関連して考慮すべき構成オプションがあるかもしれません。
この節の説明が完全であるとは考えないで下さい。

### `Host` ヘッダ攻撃を避ける

[[yii\web\UrlManager]] や [[yii\helpers\Url]] のクラスは、リンクを生成するために [[yii\web\Request::getHostInfo()|現在リクエストされているホスト名]] を使うことがあります。
って
ウェブサーバが `Host` ヘッダの値とは無関係に同じサイトとして応答するように構成されている場合は、
この情報は [HTTP リクエストを送信するユーザによって偽装され](https://www.acunetix.com/vulnerabilities/web/host-header-attack) て、信頼できないものになっている可能性があります。
そのような状況においては、ウェブサーバの構成を改修して、指定されたホスト名に対してのみ応答するようにするか、
または、`request` アプリケーションコンポーネントの [[yii\web\Request::setHostInfo()|hostInfo]] プロパティを設定して、ホスト名の値を設定ないしフィルタするか、
どちらかの対策を取るべきです。

サーバの構成についての詳細な情報は、ウェブサーバのドキュメントを参照して下さい。

- Apache 2: <http://httpd.apache.org/docs/trunk/vhosts/examples.html#defaultallports>
- Nginx: <https://www.nginx.com/resources/wiki/start/topics/examples/server_blocks/>

サーバの構成にアクセスする権限がない場合は、このような攻撃に対して防御するために、[[yii\filters\HostControl]] フィルタを設定することが出来ます。

```php
// ウェブアプリケーション構成ファイル
return [
    'as hostControl' => [
        'class' => 'yii\filters\HostControl',
        'allowedHosts' => [
            'example.com',
            '*.example.com',
        ],
        'fallbackHostInfo' => 'https://example.com',
    ],
    // ...
];
```

> Note: 「ホストヘッダ攻撃」に対する保護のためには、常に、フィルタの使用よりもウェブサーバの構成を優先すべきです。
  [[yii\filters\HostControl]] は、サーバの構成が出来ない場合にだけ使うべきものです。
