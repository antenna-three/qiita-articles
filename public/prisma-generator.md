---
title: Prismaジェネレーターは簡単に作れる
tags:
  - JavaScript
  - Node.js
  - TypeScript
  - prisma
private: false
updated_at: '2024-05-05T19:58:55+09:00'
id: 38ea8ecaf9faf9738121
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに

[Prisma](https://www.prisma.io/)ではスキーマから[ジェネレーター](https://www.prisma.io/docs/orm/prisma-schema/overview/generators)という機能を使ってコードやドキュメントを自動生成することができます。例えば、公式が提供する`prisma-client-js`をはじめとする各言語のクライアントはジェネレーターを使って生成されています。また、TypeGraphQLのリゾルバを生成する[`typegraphql-prisma`](https://www.npmjs.com/package/typegraphql-prisma)や、ER図を生成する[`prisma-erd-generator`](https://www.npmjs.com/package/prisma-erd-generator)などもあります。個人的なおすすめはダミーデータを作ってくれる[`@quramy/prisma-fabbrica`](https://github.com/Quramy/prisma-fabbrica)です。

Prismaジェネレーター自体は世の中にたくさんあるのですが、出力される形式はほぼ固定されているものが多く、要求に完璧にマッチするものは案外見つからなかったりします。また、古いまま放置されているために動かないものも多いです。

そこで、Prismaジェネレーターを一から作ってみました。公式ドキュメントにはジェネレーターに関する記述はほとんどなく、断片的な記述と既存のジェネレーターのコードが頼りでしたが、意外と簡単に求めているものを出力するジェネレーターが作れたので、作り方や注意点などをまとめました。

# 作り方

## 実行環境

PrismaジェネレーターはPrisma CLIと標準入出力上でJSON PRCを使ってやり取りするので、標準入出力を扱える任意の実行環境で作成できます。

後述する通りPrisma公式から提供されている`@prisma/generator-helper`はNPMパッケージなので、NPMパッケージが扱える環境で作るのが一番楽です。以下はNode.jsで作る前提で進めます。[^1]

[^1]: `node@20.10.0`を使用しました。検証していないですが、DenoやBunでも作れると思います。

## 依存関係

[`@prisma/generator-helper`](https://www.npmjs.com/package/@prisma/generator-helper)がJSON RPC回りのハンドリングと型定義の提供をしてくれます。

また、[Prisma CLI](https://www.npmjs.com/package/prisma)を`devDependencies`として入れておくと、手動ですがその場でE2Eテストできるので便利です。[^2]

[^2]: `@prisma/generator-helper@5.13.0`および`prisma@5.13.0`を使用しました。

```console
$ npm install @prisma/generator-helper
$ npm install -D prisma
```

あとは使いたいライブラリや必要なツールを適宜インストールしてください。

## エントリーポイント

エントリーポイントで`@prisma/generator-helper`の`generatorHandler`を呼び出します。

`onManifest`はPrisma CLIがコンソールに出力する内容やジェネレーターの実行順序を決めるためのマニフェストです。他のジェネレーターに特に依存していない場合は`defaultOutput`, `prettyName`, `version`だけ返しておけば十分です。

`onGenerate`は実際の生成処理です。今回はPrismaスキーマを解析したASTであるDMMF (Data Model meta format) が含まれている`options.dmmf`の内容を、`output`で指定されたJSONファイルにそのまま吐き出します。

```js:index.js
#!/usr/bin/env node
import fs from 'node:fs/promises'
import path from 'node:path'
import helper from '@prisma/generator-helper'

helper.generatorHandler({
	onManifest(config) {
		return {
			defaultOutput: 'dmmf.json',
			prettyName: 'DMMF JSON',
			version: '1.2.3',
		}
	},
	async onGenerate(options) {
		const path = options.generator.output?.value
		assert(path != null, 'output is required.')
		const content = JSON.stringify(options.dmmf, null, '\t')
		await fs.writeFile(path, content)
	},
})
```

## 実行

Prismaスキーマにジェネレーターを追加します。

```js:schema.prisma
generator awesome {
	provider = "node ./index.js"
}
```

`prisma generate`を実行すると、次のような結果になります。

```console
$ npx prisma generate
Prisma schema loaded from schema.prisma

✓ Generated DMMF JSON (1.2.3) to ./dmmf.json in 12ms
```

```json:dmmf.json
{
	"datamodel": {
		"enums": [...],
		"models": [...],
		"types": [...]
	},
	"schema": {
		"inputObjectTypes": {
			"prisma": [...]
		},
		"outputObjectTypes": {
			"prisma": [...],
			"model": [...]
		},
		"enumTypes": {
			"prisma": [...],
			"model": [...]
		},
		"fieldRefTypes": {
			"prisma": [...],
		}
	},
	"mappings": {
		"modelOperations": [...],
		"otherOperations": [...]
	}
}
```

これで最低限の構成は完了です。あとはDMMFから必要な情報を読み取って、ほしい形式に変換すれば目的のジェネレーターが完成します。

## 設定

Prismaスキーマからジェネレーターに設定を渡すことができます。型は常に`string | string[] | undefined`になります。

```js:schema.prisma
generator awesome {
	provider = "node ./index.js"
	output   = "generated"
	string   = "any string"
	array    = ["foo", "bar"]
}
```

`provider`, `output`, `binaryTargets`, `previewFeatures`の4つのキーは予約されており、`generator[key].value`から取得することになっています。その他のキーについては`generator.config`から取得できます。[^3]

[^3]: 実はPrismaスキーマに書かれた予約キーも`generator.config`に入りますが、Prisma CLIによる変換がされていない生の値なので、使わずに捨てた方が無難です。

`output`に相対パスを渡した場合、`schema.prisma`からの相対パスとして扱われ、Prisma CLI側で絶対パスに変換されます。また、`schema.prisma`で`output`を渡さなかった場合、マニフェストの`defaultOutput`が同様に変換されて使われます。

```js:options
{
	generator: {
		name: 'awesome',
		provider: {
			value: 'node index.js',
		},
		output: {
			value: '/absolute/path/to/output',
		},
		config: {
			string: 'any string',
			array: ['foo', 'bar']
		}
	}
}
```

`string | string[] | undefined`以外の型は扱えないため、これより複雑な型を扱いたい場合などは、`generator.config`で設定ファイルのパスを渡す形にするのがよいかと思います。

# 注意点

## DMMFの互換性は保証されない

DMMFはPrismaの内部で使われるASTのため、メジャーバージョン間で互換性がありません。また、マイナーバージョン・パッチバージョン間での互換性も保証されません。[^4]よって、利用側でPrismaのバージョンを上げる際には、ジェネレーターが動作するかどうかを確かめ、必要に応じてジェネレーター側もバージョンを上げなければなりません。

[^4]: [これが理由で](https://github.com/prisma/prisma/discussions/10721)公式ドキュメントにはPrismaジェネレーターのドキュメントが置かれてないらしいです。

最新の型定義についてはここで確認できます。

https://github.com/prisma/prisma/blob/main/packages/generator-helper/src/dmmf.ts

## ジェネレーターから標準エラー出力は使えない

前述の通り、PrismaジェネレーターはPrisma CLIと標準入出力を使ってやり取りしますが、そのうちPrismaジェネレーターからPrisma CLIへのレスポンスの受け渡しは標準エラー出力を使って行われます。また、JSON RPC以外の標準エラー出力はすべて無視されます。よって、エラーログを出力したい場合はその場で`Error`を投げて`@prisma/generator-helper`からPrisma CLIにエラーレスポンスを返してもらうか、標準出力の方にエラーログを出力する必要があります。

また、Prisma CLIはv5.13.0時点でエラーのスタックトレースをログに出してくれないので、スタックトレースが欲しい場合は自前でどこかに吐き出す必要があります。

# 作ったもの

最初はテーブル定義書とかER図とかGraphQLスキーマとかを出力するジェネレーターを1つずつ書いていたんですが、コードをいちいち書くのが面倒でメンテナンスコストも高そうだったので、DMMFを[Handlebars](https://handlebarsjs.com/)テンプレートに丸投げしてくれるジェネレーターを作りました。

こうやってテンプレートを書いたら、

```handlebars:templates/models.md.handlebars
Models:

{{#each datamodel.models}}
- {{name}}
	{{#each fields}}
	- {{name}}
	{{/each}}
{{/each}}

Enums:

{{#each datamodel.enums}}
- {{name}}
	{{#each values}}
	- {{name}}
	{{/each}}
{{/each}}
```

こういうのが出力できます。

```markdown:models.md
Models:

- User
	- id
	- email
	- type
	- name
	- posts
- Post
	- id
	- createdAt
	- updatedAt
	- title
	- content
	- published
	- viewCount
	- author
	- authorId

Enums:

- UserType
	- ADMIN
	- USER
```

ファイルの読み書きとHandlebarsへの受け渡し以外はほとんど何もしていないので、ジェネレーターというよりはジェネレーターヘルパーと言った方が正しいかもしれません。

テンプレートを書かなければいけないので手軽ではないですが、Partialsによるテンプレートの分割・再利用やHelpersによる関数の埋め込みにも対応しているので、汎用性は非常に高いと思います。よろしければぜひ利用してください。

https://www.npmjs.com/package/prisma-generator-handlebars

# おわりに

Prismaジェネレーターの作り方でした。やり方さえわかれば簡単に望み通りのものが作れるようになるので、作ってみてはいかがでしょうか。

# 参考

https://prismaio.notion.site/Prisma-Generators-a2cdf262207a4e9dbcd0e362dfac8dc0


