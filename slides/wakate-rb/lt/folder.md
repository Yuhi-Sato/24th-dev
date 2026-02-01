階層構造を表現するデータ構造とリファクタリング 〜1年で10倍成長したプロダクトの変化と課題〜

# Abstract
サービス開発では、階層構造を持つデータを扱う場面があると思います。例えばECサイトでは商品のタグを階層化したい場合などがあると思います。SaaSプロダクト「サクミル」ではフォルダや見積書の階層化機能を実装しています。

こうした階層構造をRDB上で表現するには実はいくつかのデータ構造が存在し、成長フェーズ・ワークロード特性・要件によって最適なデータ構造が変わります。本セッションではサクミルにおける2つの階層化機能において、プロダクトの成長に伴い直面した課題とそれを解決する2つのリファクタリングについて紹介したいと思います。

---

# For Review Committee

## Details
階層化とは、いわゆるノードをツリー構造で管理することを指します。
ECサイトにおける商品タグであったり、Google Driveのようなファイルストレージサービスにおけるフォルダを階層化したいことがあると思います。

弊社のプロダクトは現在リリース2年弱で、昨年度に階層化機能を実装しました。
当時はユーザー数も少なく、サービス全体の機能も不十分な状況でした。また、開発リソースも足りておらず、当時ほぼ未経験であった新卒1年目の私が階層化機能を実装する必要がありました（ちなみに社会人初タスクでした）。
これらを踏まえてパフォーマンスよりもバグなく最速で機能をリリースすることを優先し、実装が容易なAdjacency List（隣接リスト）を階層化機能のデータ構造として採用しました。

しかし、この1年でプロダクトの導入社数が10倍以上に成長した結果、階層化機能のパフォーマンス面の課題が生じ、
少ない開発リソースの中でリファクタリングを行う必要が出てきました。

リファクタリング対象となるのはフォルダと見積書の階層化機能なのですが、これら2つの機能はワークロード特性と要件が異なります。
フォルダについてはリードヘビーで、階層の深さ制限がないです。一方で見積書についてはライトヘビーで、階層の深さ制限は3階層までです。
フォルダでは読み取りが高速なClosure Table（閉包テーブル）へ、見積書では書き込みが高速なAdjacency Listのままで再帰CTEを利用することでパフォーマンスを改善します。

本セッションでは、階層構造を持つデータ構造について解説し、弊社プロダクトのリファクタリング経験を通して成長フェーズ・ワークロード特性・要件に応じたリファクタリングの大切さを伝えたいと思います。

### 対象とするターゲット
1. 階層構造を持つデータ構造のDB設計について何もわからないが、興味がある方
2. 階層構造を持つデータ構造のDB設計についてなんとなく知っているが、機能実装の経験がない方やどんなSQLを発行するべきかわからない方
3. 急成長SaaSプロダクトにおける困難とそれに対するリファクタリングについて興味がある方
4. 階層化機能をサービスに実装したい方

### 発表の流れ
#### タイトル・自己紹介(30sec)
軽く自己紹介します。

#### 階層化機能とは？(2min)
弊社のサービスとそのフォルダと見積書の階層化機能について解説します。
これらの機能は下記のような一般的な階層化機能における要件を満たしているので、ターゲットの方々の階層化機能へのイメージを具体化させることができると思います。

- 特定のノードの直下にノードを作成（e.g. フォルダの中にフォルダ・ファイルを作成）
- 特定のノードを削除（e.g. フォルダを削除）
- 特定のノードを別のノードの直下へ移動 （e.g. フォルダを別のフォルダの直下へ移動）
- 特定のノードの祖先ノードを全て取得（e.g. フォルダのパンくずリストを表示）
- 特定のノードの子孫ノードを全て取得（e.g. フォルダの中にあるフォルダ・ファイルを全て取得）

<details>
  <summary>時間配分</summary>

| 内容 | 想定スライド枚数 | 想定時間 |
| ---- | ---- | ---- |
| 弊社サービスについて | 1枚 | 30sec |
| フォルダと見積書の機能について | 2枚 | 1min・30sec |
</details>

#### 当時のプロダクト状況と実装について(3min・30sec)
昨年度のプロダクトの状況とリファクタ前の階層化機能の実装について紹介します。
当時、プロダクトのリリース間もないということもあり、ユーザー数も少なく、サービス全体の機能も不十分な状況でした。
また、開発リソースも足りておらず、新卒1年目の私が階層化機能を実装する必要がありました。
当時はほぼ未経験で、かつ社会人初タスクであったため、Railsわからん・DB設計わからんでとても苦労した記憶があります（笑）。
当然、階層構造を持つデータ構造についての知識もありませんでした。

これらの状況を踏まえて、プロダクトとしてはパフォーマンスよりもバグを出さずに最速で機能をリリースすることを優先し、実装が容易なAdjacency List（隣接リスト）を階層化機能のデータ構造として採用しました。

Adjacency Listのデータ構造でFolderモデルを実装すると下記のようなテーブルとモデルになります。

[テーブル図](https://drive.google.com/file/d/1rS-lgS4WnxTWClUxnTo9wuFuizdXHZYI/view?usp=sharing)

```ruby
  class Folder < ApplicationRecord
    has_many :child_folders, class_name: "Folder", inverse_of: 'parent_folder', dependent: :destroy
    belongs_to :parent_folder, class_name: "Folder", foreign_key: 'parent_id', optional: true

    # 祖先ノードを全て取得
    def ancestors
      return [] if parent_id.blank?

      parent_folder.ancestors + [parent_folder]
    end

    # 子孫ノードを全て取得
    def descendants
      child_folders + child_folders.map(&:descendants).flatten
    end

    validate :parent_folder_cannot_be_self, if: -> { parent_id_changed? && parent_folder.present? }
    validate :parent_folder_cannot_be_descendant, if: -> { parent_id_changed? && parent_folder.present? }

    # 自分自身を親フォルダに指定できないようにする
    def parent_folder_cannot_be_self
      if parent_folder == self
        errors.add(:parent_folder, 'に自分自身を指定できません')
      end
    end

    # 子孫フォルダを親フォルダに指定できないようにする
    def parent_folder_cannot_be_descendant
      if descendants.include?(parent_folder)
        errors.add(:parent_folder, 'に子孫フォルダを指定できません（循環が発生します）')
      end
    end
  end
```

<details>
  <summary>時間配分</summary>

| 内容 | 想定スライド枚数 | 想定時間 |
| ---- | ---- | ---- |
| 当時のプロダクト状況と意思決定 | 1枚 | 30sec |
| Adjacency Listについて | 1枚 | 1min |
| Adjacency Listの実装 | 3枚 | 2min |
</details>

#### プロダクト成長と直面した課題(2min)
階層化機能を実装して一年が経つと、プロダクトの導入社数が10倍以上に成長を遂げました。
この成長に伴い、フォルダ階層化機能のパフォーマンス面に課題が生じてきました。

Adjacency Listのメリットとしてノードの作成・移動が高速であることが挙げられます。発行されるSQLは1回で済みます。
一方で、デメリットとしてはノード削除と祖先・子孫ノードの取得が低速であることです。例えば、現在の実装だと`descendants`メソッドを実行すると、サブツリーのノード数分のSQLが発行されます。さらに、ツリーの循環を防ぐためにバリデーションを加えている都合上、書き込みの際にも`descendants`メソッドが実行されます。下記は`ancestors`メソッドと`descendants`メソッドとノード削除を実行した際のログです。特に`descendants`メソッドとノード削除ではN+1問題が発生しています。

```ruby
=== 作成されたツリー構造 ===
root
├── folder_1_1
│   ├── folder_2_1
│   └── folder_2_2
└── folder_1_2
    ├── folder_2_3
    └── folder_2_4

=== クエリの実行 ===
folder_2_4.ancestors（祖先の取得） を実行中...
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" WHERE "folders"."id" = ? LIMIT ?  [["id", 7], ["LIMIT", 1]]
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" WHERE "folders"."id" = ? LIMIT ?  [["id", 3], ["LIMIT", 1]]
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" WHERE "folders"."id" = ? LIMIT ?  [["id", 1], ["LIMIT", 1]]
祖先フォルダ数: 2

root.descendants（子孫の取得） を実行中...
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" WHERE "folders"."id" = ? LIMIT ?  [["id", 1], ["LIMIT", 1]]
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" WHERE "folders"."parent_id" = ?  [["parent_id", 1]]
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" WHERE "folders"."parent_id" = ?  [["parent_id", 2]]
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" WHERE "folders"."parent_id" = ?  [["parent_id", 4]]
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" WHERE "folders"."parent_id" = ?  [["parent_id", 5]]
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" WHERE "folders"."parent_id" = ?  [["parent_id", 3]]
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" WHERE "folders"."parent_id" = ?  [["parent_id", 6]]
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" WHERE "folders"."parent_id" = ?  [["parent_id", 7]]
子孫フォルダ数: 6

root.destroy（ノードの削除） を実行中...
  TRANSACTION (0.0ms)  BEGIN immediate TRANSACTION
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" WHERE "folders"."parent_id" = ?  [["parent_id", 1]]
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" WHERE "folders"."parent_id" = ?  [["parent_id", 2]]
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" WHERE "folders"."parent_id" = ?  [["parent_id", 4]]
  Folder Destroy (0.0ms)  DELETE FROM "folders" WHERE "folders"."id" = ?  [["id", 4]]
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" WHERE "folders"."parent_id" = ?  [["parent_id", 5]]
  Folder Destroy (0.1ms)  DELETE FROM "folders" WHERE "folders"."id" = ?  [["id", 5]]
  Folder Destroy (0.0ms)  DELETE FROM "folders" WHERE "folders"."id" = ?  [["id", 2]]
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" WHERE "folders"."parent_id" = ?  [["parent_id", 3]]
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" WHERE "folders"."parent_id" = ?  [["parent_id", 6]]
  Folder Destroy (0.0ms)  DELETE FROM "folders" WHERE "folders"."id" = ?  [["id", 6]]
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" WHERE "folders"."parent_id" = ?  [["parent_id", 7]]
  Folder Destroy (0.0ms)  DELETE FROM "folders" WHERE "folders"."id" = ?  [["id", 7]]
  Folder Destroy (0.0ms)  DELETE FROM "folders" WHERE "folders"."id" = ?  [["id", 3]]
  Folder Destroy (0.0ms)  DELETE FROM "folders" WHERE "folders"."id" = ?  [["id", 1]]
  TRANSACTION (0.0ms)  COMMIT TRANSACTION
```

プロダクト成長に伴ってツリーのノード数が増えた結果、クエリ発行数が増えたのはもちろんのこと、ネットワーク往復のオーバーヘッドも大きくなり、パフォーマンス面の課題が生じてきました。

<details>
  <summary>時間配分</summary>

| 内容 | 想定スライド枚数 | 想定時間 |
| ---- | ---- | ---- |
| プロダクトの成長について | 1枚 | 30sec |
| パフォーマンスの課題 | 2枚 | 1min・30sec |
</details>

#### フォルダ階層化機能のリファクタリング(2-3min)
フォルダ階層化機能には読み込みが多いというワークロード特性があります。また、階層の深さ制限がないという仕様があります。
そこで、読み込みに強いClosure Table（閉包テーブル）とAdjacency Listのハイブリット型のデータ構造へリファクタリングを行います。

閉包テーブルとはツリーのノードにおける全てのパスの始点と終点を記録したテーブルです。[参考テーブル図](https://drive.google.com/file/d/1YfdMWsxlf-INeLmox8G10FZTOF5XNJqY/view?usp=sharing)。メリットとして祖先・子孫ノードの取得が高速でSQLが1回で済むことが挙げられます。下記のコードは`ancestors`メソッドと`descendants`メソッドを実行した際のログです。

```ruby
folder_2_4.ancestors（祖先の取得） を実行中...
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" WHERE "folders"."id" = ? LIMIT ?  [["id", 7], ["LIMIT", 1]]
  Folder Load (0.2ms)  SELECT "folders".* FROM "folders" INNER JOIN "folder_hierarchies" ON "folders"."id" = "folder_hierarchies"."ancestor_id" WHERE "folder_hierarchies"."descendant_id" = ? AND ("folders"."id" != ?) ORDER BY "folder_hierarchies".generations ASC  [["descendant_id", 7], [nil, 7]]
祖先フォルダ数: 2

root.descendants（子孫の取得） を実行中...
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" WHERE "folders"."id" = ? LIMIT ?  [["id", 1], ["LIMIT", 1]]
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" INNER JOIN "folder_hierarchies" ON "folders"."id" = "folder_hierarchies"."descendant_id" WHERE "folder_hierarchies"."ancestor_id" = ? AND ("folders"."id" != ?) ORDER BY "folder_hierarchies".generations ASC  [["ancestor_id", 1], [nil, 1]]
子孫フォルダ数: 6
```

一方でデメリットとして、ノードの作成・移動・削除がやや低速になります。作成の場合はフォルダの挿入に加えて、閉包テーブルの挿入も必要なので3回のINSERT文が必要になります。ノード移動の場合は、親ノードの更新と子孫の閉包テーブルの更新が必要になります。

```ruby
folder_2_4.children.create（ノードの作成） を実行中...
  TRANSACTION (0.0ms)  BEGIN immediate TRANSACTION
  Folder Create (0.1ms)  INSERT INTO "folders" ("name", "parent_id") VALUES (?, ?) RETURNING "id"  [["name", "folder_2_5"], ["parent_id", 7]]
  FolderHierarchy Create (0.0ms)  INSERT INTO "folder_hierarchies" ("ancestor_id", "descendant_id", "generations") VALUES (?, ?, ?)  [["ancestor_id", 8], ["descendant_id", 8], ["generations", 0]]
   (0.0ms)  INSERT INTO "folder_hierarchies" (ancestor_id, descendant_id, generations) SELECT x.ancestor_id, 8, x.generations + 1 FROM "folder_hierarchies" x WHERE x.descendant_id = 7
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" WHERE "folders"."parent_id" = ? ORDER BY "folders"."id" ASC LIMIT ?  [["parent_id", 8], ["LIMIT", 1000]]
  TRANSACTION (0.0ms)  COMMIT TRANSACTION

folder_1_1.add_child(folder_1_2)（ノードの移動） を実行中...
  TRANSACTION (0.0ms)  BEGIN immediate TRANSACTION
  Folder Exists? (0.1ms)  SELECT 1 AS one FROM "folders" INNER JOIN "folder_hierarchies" ON "folders"."id" = "folder_hierarchies"."ancestor_id" WHERE "folder_hierarchies"."descendant_id" = ? AND "folders"."id" = ? LIMIT ?  [["descendant_id", 2], ["id", 3], ["LIMIT", 1]]
  Folder Update (0.0ms)  UPDATE "folders" SET "parent_id" = ? WHERE "folders"."id" = ?  [["parent_id", 2], ["id", 3]]
   (0.1ms)  DELETE FROM "folder_hierarchies" WHERE descendant_id IN ( SELECT DISTINCT descendant_id FROM (SELECT descendant_id FROM "folder_hierarchies" WHERE ancestor_id = 3 OR descendant_id = 3 ) AS x )
  FolderHierarchy Create (0.0ms)  INSERT INTO "folder_hierarchies" ("ancestor_id", "descendant_id", "generations") VALUES (?, ?, ?)  [["ancestor_id", 3], ["descendant_id", 3], ["generations", 0]]
   (0.0ms)  INSERT INTO "folder_hierarchies" (ancestor_id, descendant_id, generations) SELECT x.ancestor_id, 3, x.generations + 1 FROM "folder_hierarchies" x WHERE x.descendant_id = 2
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" WHERE "folders"."parent_id" = ? ORDER BY "folders"."id" ASC LIMIT ?  [["parent_id", 3], ["LIMIT", 1000]]
   (0.0ms)  DELETE FROM "folder_hierarchies" WHERE descendant_id IN ( SELECT DISTINCT descendant_id FROM (SELECT descendant_id FROM "folder_hierarchies" WHERE ancestor_id = 6 OR descendant_id = 6 ) AS x )
  FolderHierarchy Create (0.0ms)  INSERT INTO "folder_hierarchies" ("ancestor_id", "descendant_id", "generations") VALUES (?, ?, ?)  [["ancestor_id", 6], ["descendant_id", 6], ["generations", 0]]
   (0.0ms)  INSERT INTO "folder_hierarchies" (ancestor_id, descendant_id, generations) SELECT x.ancestor_id, 6, x.generations + 1 FROM "folder_hierarchies" x WHERE x.descendant_id = 3
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" WHERE "folders"."parent_id" = ? ORDER BY "folders"."id" ASC LIMIT ?  [["parent_id", 6], ["LIMIT", 1000]]
   (0.0ms)  DELETE FROM "folder_hierarchies" WHERE descendant_id IN ( SELECT DISTINCT descendant_id FROM (SELECT descendant_id FROM "folder_hierarchies" WHERE ancestor_id = 7 OR descendant_id = 7 ) AS x )
  FolderHierarchy Create (0.0ms)  INSERT INTO "folder_hierarchies" ("ancestor_id", "descendant_id", "generations") VALUES (?, ?, ?)  [["ancestor_id", 7], ["descendant_id", 7], ["generations", 0]]
   (0.1ms)  INSERT INTO "folder_hierarchies" (ancestor_id, descendant_id, generations) SELECT x.ancestor_id, 7, x.generations + 1 FROM "folder_hierarchies" x WHERE x.descendant_id = 3
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" WHERE "folders"."parent_id" = ? ORDER BY "folders"."id" ASC LIMIT ?  [["parent_id", 7], ["LIMIT", 1000]]
   (0.1ms)  DELETE FROM "folder_hierarchies" WHERE descendant_id IN ( SELECT DISTINCT descendant_id FROM (SELECT descendant_id FROM "folder_hierarchies" WHERE ancestor_id = 8 OR descendant_id = 8 ) AS x )
  FolderHierarchy Create (0.0ms)  INSERT INTO "folder_hierarchies" ("ancestor_id", "descendant_id", "generations") VALUES (?, ?, ?)  [["ancestor_id", 8], ["descendant_id", 8], ["generations", 0]]
   (0.0ms)  INSERT INTO "folder_hierarchies" (ancestor_id, descendant_id, generations) SELECT x.ancestor_id, 8, x.generations + 1 FROM "folder_hierarchies" x WHERE x.descendant_id = 7
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" WHERE "folders"."parent_id" = ? ORDER BY "folders"."id" ASC LIMIT ?  [["parent_id", 8], ["LIMIT", 1000]]
  FolderHierarchy Load (0.0ms)  SELECT "folder_hierarchies".* FROM "folder_hierarchies" WHERE "folder_hierarchies"."descendant_id" = ? ORDER BY "folder_hierarchies".generations ASC  [["descendant_id", 3]]
  Folder Load (0.0ms)  SELECT "folders".* FROM "folders" INNER JOIN "folder_hierarchies" ON "folders"."id" = "folder_hierarchies"."ancestor_id" WHERE "folder_hierarchies"."descendant_id" = ? ORDER BY "folder_hierarchies".generations ASC  [["descendant_id", 3]]
  TRANSACTION (0.0ms)  COMMIT TRANSACTION
```

しかし、フォルダ階層化機能においては書き込み処理よりも読み込み処理の方が多いため、読み取りの高速化の方が恩恵があるためClosure Tableが採用されました。Closure TableをActiveRecordで実装する際には[closure_tree](https://github.com/ClosureTree/closure_tree)というgemがおすすめです。既存のAdjacency Listのデータ構造からそのままClosure Tableとのハイブリッド型のデータ構造に簡単に移行することができます。

#### 見積書階層化機能のリファクタリング(2-3min)
見積書階層化機能においては書き込み処理の方が多いワークロード特性があります。また、階層の深さも3までに制限されています。
そのため、Closure Tableのデータ構造よりもAdjacency Listのままでも十分なパフォーマンスが出ます。
さらに高速化するために再帰CTEを活用した実装へリファクタリングを行いました。
実はRailsでも[ActiveRecord::QueryMethods#with_recursive](https://api.rubyonrails.org/classes/ActiveRecord/QueryMethods.html#method-i-with_recursive)に再帰CTEが実装されています。
この再帰クエリを`ancestors`メソッドと`descendants`メソッドの実装にすることで、パフォーマンスを改善することができます。

```ruby
  # 見積階層化機能では工事を階層化する
  class Construction < ActiveRecord::Base
    has_many :child_constructions, class_name: "Construction", inverse_of: 'parent_construction', dependent: :destroy
    belongs_to :parent_construction, class_name: "Construction", foreign_key: 'parent_id', optional: true

    # 祖先ノードを全て取得
    def ancestors
      return [] if parent_id.blank?

      Construction.with_recursive(
        ancestors: [
          Construction.where(id: parent_id),
          Construction.joins('JOIN ancestors ON constructions.id = ancestors.parent_id')
        ]
      ).from('ancestors')
    end

    # 子孫ノードを全て取得
    def descendants
      return [] if child_constructions.blank?

      Construction.with_recursive(
        descendants: [
          Construction.where(id: child_constructions),
          Construction.joins('JOIN descendants ON constructions.parent_id = descendants.id')
        ]
      ).from('descendants')
    end
  end
```

```ruby
construction_2_4.ancestors（祖先の取得）を実行中...
  Construction Load (0.0ms)  SELECT "constructions".* FROM "constructions" WHERE "constructions"."id" = ? LIMIT ?  [["id", 7], ["LIMIT", 1]]
  Construction Count (0.1ms)  WITH RECURSIVE "ancestors" AS ( SELECT "constructions".* FROM "constructions" WHERE "constructions"."id" = ? UNION ALL SELECT "constructions".* FROM "constructions" JOIN ancestors ON constructions.id = ancestors.parent_id ) SELECT COUNT(*) FROM ancestors  [["id", 3]]
祖先工事数: 2

root.descendants（子孫の取得） を実行中...
  Construction Load (0.0ms)  SELECT "constructions".* FROM "constructions" WHERE "constructions"."id" = ? LIMIT ?  [["id", 1], ["LIMIT", 1]]
  Construction Load (0.0ms)  SELECT "constructions".* FROM "constructions" WHERE "constructions"."parent_id" = ?  [["parent_id", 1]]
  Construction Count (0.4ms)  WITH RECURSIVE "descendants" AS ( SELECT "constructions".* FROM "constructions" WHERE "constructions"."id" IN (SELECT "constructions"."id" FROM "constructions" WHERE "constructions"."parent_id" = ?) UNION ALL SELECT "constructions".* FROM "constructions" JOIN descendants ON constructions.parent_id = descendants.id ) SELECT COUNT(*) FROM descendants  [["parent_id", 1]]
子孫工事数: 6
```

このようにワークロード特性と要件に合わせてデータ構造と技術選定を行うことが大事です。

### 参考実装
#### Adjacency List
```ruby
  # frozen_string_literal: true

  require "bundler/inline"

  gemfile(true) do
    source "https://rubygems.org"

    gem "rails"
    # If you want to test against edge Rails replace the previous line with this:
    # gem "rails", github: "rails/rails", branch: "main"

    gem "sqlite3"
  end

  require "active_record/railtie"
  require "minitest/autorun"

  # This connection will do for database-independent bug reports.
  ENV["DATABASE_URL"] = "sqlite3::memory:"

  class TestApp < Rails::Application
    config.load_defaults Rails::VERSION::STRING.to_f
    config.eager_load = false
    config.logger = Logger.new($stdout)
    config.secret_key_base = "secret_key_base"

    config.active_record.encryption.primary_key = "primary_key"
    config.active_record.encryption.deterministic_key = "deterministic_key"
    config.active_record.encryption.key_derivation_salt = "key_derivation_salt"
  end
  Rails.application.initialize!

  ActiveRecord::Schema.define do
    create_table :folders, force: true do |t|
      t.string :name
      t.integer :parent_id
    end
  end

  class Folder < ActiveRecord::Base
    has_many :child_folders, class_name: "Folder", inverse_of: 'parent_folder', dependent: :destroy
    belongs_to :parent_folder, class_name: "Folder", foreign_key: 'parent_id', optional: true

    # 祖先ノードを全て取得
    def ancestors
      return [] if parent_id.blank?

      parent_folder.ancestors + [parent_folder]
    end

    # 子孫ノードを全て取得
    def descendants
      child_folders + child_folders.map(&:descendants).flatten
    end

    validate :parent_folder_cannot_be_self, if: -> { parent_id_changed? && parent_folder.present? }
    validate :parent_folder_cannot_be_descendant, if: -> { parent_id_changed? && parent_folder.present? }

    # 自分自身を親フォルダに指定できないようにする
    def parent_folder_cannot_be_self
      if parent_folder == self
        errors.add(:parent_folder, 'に自分自身を指定できません')
      end
    end

    # 子孫フォルダを親フォルダに指定できないようにする
    def parent_folder_cannot_be_descendant
      if descendants.include?(parent_folder)
        errors.add(:parent_folder, 'に子孫フォルダを指定できません（循環が発生します）')
      end
    end
  end

  puts "=== ツリー構造を作成中 ==="

  root = Folder.create!(name: "root")

  folder_1_1 = Folder.create!(name: "folder_1_1", parent_folder: root)
  folder_1_2 = Folder.create!(name: "folder_1_2", parent_folder: root)

  folder_2_1 = Folder.create!(name: "folder_2_1", parent_folder: folder_1_1)
  folder_2_2 = Folder.create!(name: "folder_2_2", parent_folder: folder_1_1)
  folder_2_3 = Folder.create!(name: "folder_2_3", parent_folder: folder_1_2)
  folder_2_4 = Folder.create!(name: "folder_2_4", parent_folder: folder_1_2)

  puts "\n=== 作成されたツリー構造 ==="
  puts "#{root.name}"
  puts "├── #{folder_1_1.name}"
  puts "│   ├── #{folder_2_1.name}"
  puts "│   └── #{folder_2_2.name}"
  puts "└── #{folder_1_2.name}"
  puts "    ├── #{folder_2_3.name}"
  puts "    └── #{folder_2_4.name}"

  puts "\n=== クエリの実行 ==="
  ActiveRecord::Base.logger.level = 0
  ActiveRecord::Base.logger.formatter = proc do |severity, datetime, progname, msg|
    "#{msg}\n"
  end

  puts "folder_2_4.ancestors（祖先の取得）を実行中..."
  folder_2_4_reloaded = Folder.find(folder_2_4.id)
  ancestors = folder_2_4_reloaded.ancestors
  puts "祖先フォルダ数: #{ancestors.count}"

  puts "\nroot.descendants（子孫の取得） を実行中..."
  root_reloaded = Folder.find(root.id)
  descendants = root_reloaded.descendants
  puts "子孫フォルダ数: #{descendants.count}"

  puts "\nroot.destroy（ノードの削除） を実行中..."
  root.destroy
```

#### Closure Table
```ruby
  # frozen_string_literal: true

  require "bundler/inline"

  gemfile(true) do
    source "https://rubygems.org"

    gem "rails"
    # If you want to test against edge Rails replace the previous line with this:
    # gem "rails", github: "rails/rails", branch: "main"

    gem "sqlite3"
    gem "closure_tree"
  end

  require "active_record/railtie"
  require "minitest/autorun"

  # This connection will do for database-independent bug reports.
  ENV["DATABASE_URL"] = "sqlite3::memory:"

  class TestApp < Rails::Application
    config.load_defaults Rails::VERSION::STRING.to_f
    config.eager_load = false
    config.logger = Logger.new($stdout)
    config.secret_key_base = "secret_key_base"

    config.active_record.encryption.primary_key = "primary_key"
    config.active_record.encryption.deterministic_key = "deterministic_key"
    config.active_record.encryption.key_derivation_salt = "key_derivation_salt"
  end
  Rails.application.initialize!

  ActiveRecord::Schema.define do
    create_table :folders, force: true do |t|
      t.string :name
      t.integer :parent_id
    end

    create_table :folder_hierarchies, id: false, force: true do |t|
      t.integer :ancestor_id, null: false
      t.integer :descendant_id, null: false
      t.integer :generations, null: false
    end

    add_index :folder_hierarchies, [:ancestor_id, :descendant_id, :generations],
              unique: true, name: "folder_anc_desc_idx"
    add_index :folder_hierarchies, [:descendant_id],
              name: "folder_desc_idx"
  end

  class Folder < ActiveRecord::Base
    has_closure_tree dependent: :destroy

    has_many :child_folders, class_name: "Folder", inverse_of: 'parent_folder', dependent: :destroy
    belongs_to :parent_folder, class_name: "Folder", foreign_key: 'parent_id', optional: true
  end

  puts "=== ツリー構造を作成中 ==="

  root = Folder.create!(name: "root")

  folder_1_1 = Folder.create!(name: "folder_1_1", parent: root)
  folder_1_2 = Folder.create!(name: "folder_1_2", parent: root)

  folder_2_1 = Folder.create!(name: "folder_2_1", parent: folder_1_1)
  folder_2_2 = Folder.create!(name: "folder_2_2", parent: folder_1_1)
  folder_2_3 = Folder.create!(name: "folder_2_3", parent: folder_1_2)
  folder_2_4 = Folder.create!(name: "folder_2_4", parent: folder_1_2)

  puts "\n=== 作成されたツリー構造 ==="
  puts "#{root.name}"
  puts "├── #{folder_1_1.name}"
  puts "│   ├── #{folder_2_1.name}"
  puts "│   └── #{folder_2_2.name}"
  puts "└── #{folder_1_2.name}"
  puts "    ├── #{folder_2_3.name}"
  puts "    └── #{folder_2_4.name}"

  puts "\n=== クエリの実行 ==="
  ActiveRecord::Base.logger.level = 0
  ActiveRecord::Base.logger.formatter = proc do |severity, datetime, progname, msg|
    "#{msg}\n"
  end

  puts "folder_2_4.ancestors（祖先の取得） を実行中..."
  folder_2_4_reloaded = Folder.find(folder_2_4.id)
  ancestors = folder_2_4_reloaded.ancestors.to_a
  puts "祖先フォルダ数: #{ancestors.count}"

  puts "\nroot.descendants（子孫の取得） を実行中..."
  root_reloaded = Folder.find(root.id)
  descendants = root_reloaded.descendants.to_a
  puts "子孫フォルダ数: #{descendants.count}"

  puts "\nfolder_2_4.children.create（子ノードの追加） を実行中..."
  folder_2_4.children.create(name: "folder_2_5")

  puts "\nfolder_1_1.add_child(folder_1_2)（ノードの移動） を実行中..."
  folder_1_1.add_child(folder_1_2)

  puts "\nroot.destroy を実行中..."
  root.destroy

```

### Adjacency List + 再帰CTE
```ruby
  # frozen_string_literal: true

  require "bundler/inline"

  gemfile(true) do
    source "https://rubygems.org"

    gem "rails"
    # If you want to test against edge Rails replace the previous line with this:
    # gem "rails", github: "rails/rails", branch: "main"

    gem "sqlite3"
  end

  require "active_record/railtie"
  require "minitest/autorun"

  # This connection will do for database-independent bug reports.
  ENV["DATABASE_URL"] = "sqlite3::memory:"

  class TestApp < Rails::Application
    config.load_defaults Rails::VERSION::STRING.to_f
    config.eager_load = false
    config.logger = Logger.new($stdout)
    config.secret_key_base = "secret_key_base"

    config.active_record.encryption.primary_key = "primary_key"
    config.active_record.encryption.deterministic_key = "deterministic_key"
    config.active_record.encryption.key_derivation_salt = "key_derivation_salt"
  end
  Rails.application.initialize!

  ActiveRecord::Schema.define do
    create_table :constructions, force: true do |t|
      t.string :name
      t.integer :parent_id
    end
  end

  class Construction < ActiveRecord::Base
    has_many :child_constructions, class_name: "Construction", inverse_of: 'parent_construction', dependent: :destroy
    belongs_to :parent_construction, class_name: "Construction", foreign_key: 'parent_id', optional: true

    # 祖先ノードを全て取得
    def ancestors
      return [] if parent_id.blank?

      Construction.with_recursive(
        ancestors: [
          Construction.where(id: parent_id),
          Construction.joins('JOIN ancestors ON constructions.id = ancestors.parent_id')
        ]
      ).from('ancestors')
    end

    # 子孫ノードを全て取得
    def descendants
      return [] if child_constructions.blank?

      Construction.with_recursive(
        descendants: [
          Construction.where(id: child_constructions),
          Construction.joins('JOIN descendants ON constructions.parent_id = descendants.id')
        ]
      ).from('descendants')
    end

    validate :parent_construction_cannot_be_self, if: -> { parent_id_changed? && parent_construction.present? }
    validate :parent_construction_cannot_be_descendant, if: -> { parent_id_changed? && parent_construction.present? }

    # 自分自身を親工事に指定できないようにする
    def parent_construction_cannot_be_self
      if parent_construction == self
        errors.add(:parent_construction, 'に自分自身を指定できません')
      end
    end

    # 子孫工事を親工事に指定できないようにする
    def parent_construction_cannot_be_descendant
      if descendants.include?(parent_construction)
        errors.add(:parent_construction, 'に子孫工事を指定できません（循環が発生します）')
      end
    end
  end

  puts "=== ツリー構造を作成中 ==="

  root = Construction.create!(name: "root")

  construction_1_1 = Construction.create!(name: "construction_1_1", parent_construction: root)
  construction_1_2 = Construction.create!(name: "construction_1_2", parent_construction: root)

  construction_2_1 = Construction.create!(name: "construction_2_1", parent_construction: construction_1_1)
  construction_2_2 = Construction.create!(name: "construction_2_2", parent_construction: construction_1_1)
  construction_2_3 = Construction.create!(name: "construction_2_3", parent_construction: construction_1_2)
  construction_2_4 = Construction.create!(name: "construction_2_4", parent_construction: construction_1_2)

  puts "\n=== 作成されたツリー構造 ==="
  puts "#{root.name}"
  puts "├── #{construction_1_1.name}"
  puts "│   ├── #{construction_2_1.name}"
  puts "│   └── #{construction_2_2.name}"
  puts "└── #{construction_1_2.name}"
  puts "    ├── #{construction_2_3.name}"
  puts "    └── #{construction_2_4.name}"

  puts "\n=== クエリの実行 ==="
  ActiveRecord::Base.logger.level = 0
  ActiveRecord::Base.logger.formatter = proc do |severity, datetime, progname, msg|
    "#{msg}\n"
  end

  puts "construction_2_4.ancestors（祖先の取得）を実行中..."
  construction_2_4_reloaded = Construction.find(construction_2_4.id)
  ancestors = construction_2_4_reloaded.ancestors
  puts "祖先工事数: #{ancestors.count}"

  puts "\nroot.descendants（子孫の取得） を実行中..."
  root_reloaded = Construction.find(root.id)
  descendants = root_reloaded.descendants
  puts "子孫工事数: #{descendants.count}"

  puts "\nroot.destroy（ノードの削除） を実行中..."
  root.destroy

```

## Pitch
弊社の建設系SaaSプロダクト「サクミル」はリリースしてから2年弱ですが、1年で10倍以上の成長を遂げて昨年度末に導入社数が1,000を突破しました。この成長スピードのプロダクトのリファクタリングについてはあまり聞けない話題だと思います。

私はISUCON14にチーム「黒酢唐揚げサン丼」として出場し、入賞しました。そのため、DB設計のリファクタによるパフォーマンスチューニングについては一定の経験があります。[ISUCON14の公式サイト](https://isucon.net/archives/58837992.html)

また`with_recursive`についてRailsへTypo修正の[コントリビュート](https://github.com/rails/rails/pull/55213)をしました（熱意を伝えたい笑）。
