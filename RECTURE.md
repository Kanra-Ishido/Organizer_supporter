# ドメイン駆動設計手法講座

参考資料：
https://zenn.dev/yamachan0625/books/ddd-hands-on/viewer/chapter1_intro

## 「ドメイン」とは
ソフトウェアが解決しようとしている特定の問題領域を指します。

## 戦術的設計
![alt text](https://storage.googleapis.com/zenn-user-upload/0852995fe722-20240108.png)

|レイヤー|説明|
|-|-|
|ドメイン層|これはアプリケーションのコアであり、ビジネスルールやビジネスロジックを表現します。主にエンティティ、値オブジェクト、ドメインサービス、およびリポジトリのインターフェイスが含まれます。|
|アプリケーション層|アプリケーション層はドメイン層のクライアントです。ユースケースを組み立てるためにドメイン層を使用します。主にアプリケーションサービスが含まれます。|
|インフラストラクチャ層|この層はアプリケーションに必要な外部リソース (データベース、ファイルシステム、外部サービス) への通信を担当します。主にリポジトリの実装が含まれます。|
|プレゼンテーション層|ユーザーのリクエストを受け取り、適切な応答を返します。主に Web UI、REST API、CLI などが含まれます。|

## 依存性逆転の原則 (DIP）
オニオンアーキテクチャにおける依存関係の方向性は、「依存性逆転の原則 (Dependency Inversion Principle, DIP) 」というソフトウェア設計の原則を反映しています。この原則は以下の二つのポイントで構成されています。

* **上位モジュールは下位モジュールに依存してはならない。**

    ビジネスロジックを含む上位モジュール (ドメイン層) は、データアクセスや外部 API のような下位モジュール (インフラストラクチャ層やプレゼンテーション層) に依存してはいけません、。そして、どちらのモジュールも抽象に依存すべきです。

    例えば、ドメイン層のオブジェクトの中にアプリケーション層のオブジェクトが入っていると、どちらかを変更した際にもう一方に影響が出てしまいます。これは依存性逆転の原則に反しています。

    ```python
    # これは依存性逆転の原則に反している
    def sayHello(name: str) -> None:
        logger(f'Hello {name}!')

    sayHello('World')
    ```

    ```python
    # これは依存性逆転の原則に従っている
    def sayHello(name: str, logger: function) -> None:
        logger(f'Hello {name}!')

    sayHello('World', print)
    ```

* **抽象は詳細に依存してはならない。詳細が抽象に依存すべきである。**

    抽象 (インターフェイスや抽象クラス) は詳細 (具体的な実装) に依存せず、詳細は抽象に依存すべきです。


## 階層構造
```
├── 外部サービスと接続オブジェクト
├── テストコード
└── APIの関数内処理・UI内の処理など
    └── ユースケース
        ├── サービス
        └── レポジトリ
            └── エンティティ
                └── 値オブジェクト
```

## それぞれのサンプルコード
### 値オブジェクト
```python
class ID:
    # [!]値の型を定義してやること
    _value: int

    def __init__(self, value: int) -> None:
        self._value = varidate(value)

    # [!]受け取った値をチェックしてから保持する
    def varidate(value: int) -> int:
        if not isinstance(value, int):
            raise ValueError('値がintではありません')
        if value < 0:
            raise ValueError('値が0未満です')
        
        # ...

        return value

    # [!]クラスが保持してる変数を取り出したい場合は、変数を直接叩かず、メソッドを介して取り出す
    def value(self) -> int: ## こういう役割のメソッドのことをgetterという
        return self._value
```

```python
id = ID(1)

print(id.value()) # 1

# [!]値オブジェクトは絶対に保持してる値を上書きしてはいけない
id._value = 2

error_id = ID(-1) # ValueError: 値が0未満です
```


### エンティティ
データベースの属性に近いもの
|id|name|age|
|-|-|-|
|1|Alice|20|
|2|Bob|30|
|3|Charlie|40|
|...|...|...|

↑こういう感じのDBを想像している場合、
必要な値オブジェクトを全部定義してあげた上で、エンティティを定義する

```python
from .value_object import ID, Name, Age

# [!]値オブジェクトを使ってエンティティを定義する
class UserEntity:
    _id: ID
    name: Name
    age: Age

    def __init__(self, id: ID, name: Name, age: Age):
        self._id = id
        self.name = name
        self.age = age

    # [!]エンティティが持ってる値オブジェクトを取り出すメソッドを介して取り出す
    # [!]値オブジェクトの保持する値を直接取り出すことはしない
    def id(self) -> ID:
        return self._id

    def name(self) -> Name:
        return self.name

    def age(self) -> Age:
        return self.age


    # [!]リスト、タプル、JSON、辞書で返す場合でも同様
    def toDict(self) -> dict:
        return {
            'id': self._id,
            'name': self.name,
            'age': self.age
        }
```

```python
user = UserEntity(ID(1), Name('Alice'), Age(20))

print(user.id()) # ID(1)
print(user.name()) # Name('Alice')
print(user.age()) # Age(20)

# [!]エンティティを区別する値以外は上書きできる
user.name = Name('Bob')
user._id = ID(2) # これはダメ

```

### リポジトリ
データベースの操作に近いもの
* CRUD操作
    * `Create`：新しいデータを作成する
    * `Read`：データを取得する
    * `Update`：データを更新する
    * `Delete`：データを削除する

```python
class UserRepository:
    # [!]コンストラクタにデータベースのクライアントを受け取る
    def __init__(self, db_client):
        self.client = db_client

    # [!]CRUD操作を定義する
    # [!]テーブルレベルの操作はやっちゃダメ（テーブル名の変更、テーブルのリセット、列追加）→ サービス層でやる
    def create(self, user: UserEntity) -> None:
        try:
            self.client.execute(
                'INSERT INTO users (id, name, age) VALUES (?, ?, ?)',
                (
                    user.id().value(), 
                    user.name().value(), 
                    user.age().value()
                )
            )
        except Exception as e:
            raise Exception('ユーザーの作成に失敗しちゃったｱｾｱｾ👉👈💦')
        else:
            self.client.commit()


    def selectAll(self) -> list:
        try:
            self.client.execute('SELECT * FROM users')
            users = self.client.fetchall()
        except Exception as e:
            raise Exception('ユーザーの取得に失敗しちゃったｱｾｱｾ👉👈💦')
        else:
            return users


```


```python
# DBのクライアントのイメージ
db_client = sqlite3.connect('example.db').cursor()
db_client.execute(
    'CREATE TABLE users (id int, name text, age int)'
)

# [!]クライアントを渡してリポジトリを作成する
repository = UserRepository(db_client)
repository.create(
    UserEntity(ID(1), Name('Alice'), Age(20)
)
```


### サービス
* 値オブジェクト、エンティティ、リポジトリで禁止していたことをやる場
* 値オブジェクト
    * 重複チェック ... データベースの中身をチェックする
    * ハッシュ化 ... パスワードをハッシュ化する
* レポジトリ
    * テーブルレベルの操作 ... テーブル名の変更、テーブルのリセット、列追加


```python

# 例えば名前の重複チェックを行うサービス
class CheckDuplicateNameService:
    # [!] 基本、クラスメソッドはexecute()にする、引数はなし
    def __init__(self, db_client, name_obj: Name):
        self.client = db_client
        self.name = name_obj

    def execute(self):
        try:
            self.client.execute(
                'SELECT * FROM users WHERE name = ?',
                (self.name.value(),)
            )
            user = self.client.fetchone()

            if user:
                raise ValueError('名前が重複しています')
            
        except Exception as e:
            raise Exception('名前の重複チェックに失敗しちゃったｱｾｱｾ👉👈💦')
        else:
            return self.name
```