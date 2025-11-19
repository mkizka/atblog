---
createdAt: '2024-10-12T13:05:34.790Z'
theme: github-light
title: AT Protocol対応サービス開発メモ
visibility: public
---
## Linkatというリンク集サービスを作りました
https://linkat.blue というBlueskyアカウントでいわゆるリンク集を作れるサービスを作りました。このサービスはAT Protocol(以降atproto)に対応しており、作ったリンク集は利用者のPDSに保存されます。

実装にあたっては公式ガイドのstatusphereというサンプル実装を参考にしました。atproto対応サービスのNode.jsを使った基本的な実装方法はこれを読めば分かります。

https://atproto.com/ja/guides/applications

本記事ではガイドを読んで実際に実装してみて、詰まったことや実装時の工夫をいくつかまとめて紹介します。

### GitHubリポジトリ
実装は以下で公開しています。スターください。

https://github.com/mkizka/linkat

主に[Remix](https://remix.run)を使用しています。

## atprotoのローカル開発環境を使うと便利
atproto開発にあたって公式のFirehoseにいきなり繋げてもいいのですが、作ったデータはすべて公開されるので一応隠しておきたい場合や、雑にE2Eテストを作って回したい時はローカル開発環境が便利です。

ローカル開発環境の構築には以下記事がそのまま使えました。役立つ記事をありがとうございます。

[Blueskyの開発用サーバーの起動方法｜nabeyang ](https://zenn.dev/nnabeyang/articles/b926c1e2f663f4)

自分の場合はさらに[giget](https://www.npmjs.com/package/giget)というライブラリを使ってatprotoリポジトリを開発前にダウンロードするスクリプトを書き、開発サーバー起動時にセットアップするようにしています([実装](https://github.com/mkizka/linkat/blob/baf070945dea2ee01dca92f6abe625c1221e888e/scripts/postinstall.sh))。

なお、ローカル開発環境では以下のアドレスがそれぞれ各システムに対応しています。開発時はこれらを使用します。

- PDS ... http://localhost:2583
- AppView ... http://localhost:2584
- plc.directory ... http://localhost:2582
- Firehose ... ws://localhost:2583
  - これだけログに記載がないので合ってるか分かってません

例えばOAuthクライアントでは以下のように設定します。

```ts
  const oauthClient = new NodeOAuthClient({
    clientMetadata: {
      (省略)
    },
    plcDirectoryUrl: "http://localhost:2582",
    // @ts-expect-error: なぜか型定義上は設定出来ないようになっている
    handleResolver: "http://localhost:2584",
    (省略)
  });
```

`handleResolver`はローカル開発用のハンドルである`alice.test`を解決するために指定する必要がありました。

## 独自lexiconを使ったAgentクラスを作る例
statusphereではatprotoのxrpcを呼び出すのに@atproto/apiのAgentクラスがそのまま使われていますが、lex-cliを使用して生成したメソッドが使えるクラスを作っておくと便利でした。以下のような感じで書けます。

```ts

export class LinkatAgent extends Agent {
  blue: BlueNS;

  constructor(options: ConstructorParameters<typeof Agent>[0]) {
    super(options);
    this.blue = new BlueNS(this);
  }

  async getBoard(
    params: Omit<Parameters<typeof this.blue.linkat.board.get>[0], "rkey">,
  ) {
    return await this.blue.linkat.board.get({
      ...params,
      rkey: "self",
    });
  }

  async updateBoard(board: unknown) {
    // blue.linkat.boardにはなぜかputがないので、com.atproto.repoを使う
    return await this.com.atproto.repo.putRecord({
      repo: this.assertDid,
      validate: false,
      collection: "blue.linkat.board",
      rkey: "self",
      record: boardScheme.parse(board),
    });
  }

  async deleteBoard() {
    return await this.blue.linkat.board.delete({
      repo: this.assertDid,
      rkey: "self",
    });
  }
}

```

`blue.linkat.board`ではrkeyが`self`固定なのですが、createメソッド以外はそれを想定したコードが生成されない(@atproto/lex-cli@0.5.0時点)ため個別に指定しています。

あとなぜか生成されるコードにはputRecordに相当するものがなく、`com.atproto.repo.putRecord`を使っています。

## OAuthの認証情報をRemixでセッション管理する方法
OAuthでログインに成功した後、`oauthClient.callback`がログインユーザーの情報を返してきます。

statusphereでは以下のように実装されています。

https://github.com/bluesky-social/statusphere-example-app/blob/9cd25e3c8db23eb614ad40d9da36898a6f177070/src/routes.ts#L71-L90

ログインユーザーのdidをセッションとして保存していることが分かります。私はこれをRemixのセッション管理機能を使って実装してみました。

https://remix.run/docs/en/main/utils/sessions

これは app/routes/oauth.callback.tsx (OAuthの認証画面でAcceptした後リダイレクトされるページ)の実装です。

```ts
export async function loader({ request }: LoaderFunctionArgs) {
  const remixSession = await getSession(request.headers.get("Cookie"));
  try {
    const oauthClient = await createOAuthClient();
    const { session: oauthSession } = await oauthClient.callback(
      new URL(request.url).searchParams,
    );
    remixSession.set("did", oauthSession.did);
    return redirect("/edit", {
      headers: {
        "Set-Cookie": await commitSession(remixSession),
      },
    });
  } catch (error) {
    logger.error("OAuthコールバックに失敗しました", { error });
    return redirect("/login");
  }
}
```

セッション情報からdidを取り出しCookieに詰めて返しています。取り出すときはこんな感じです。

```ts
export const getSessionUserDid = async (request: Request) => {
  const session = await getSession(request.headers.get("Cookie"));
  if (!session.data.did) {
    return null;
  }
  return session.data.did;
};
```

## OAuthの`token_endpoint_auth_method`に`private_key_jwt`を使う
statusphereでは`token_endpoint_auth_method`は`none`になっていますが、公式ドキュメントを見ると

> confidential clients must include token_endpoint_auth_method as `private_key_jwt` in their client metadata document
> https://atproto.com/ja/specs/oauth

とあります。私はOAuthについてはかなり初心者ですが、ここで言うconfidential clientsはサーバーで実行するアプリのことを指しているはずなので、そのような場合は`private_key_jwt`にすべきのようです。

実装方法についてはstatusphereの過去コミットを参考になります。

https://github.com/bluesky-social/statusphere-example-app/commit/fb74afee51e05bcd061e40de26bcb334e60c82f5

## `unauthenticatedCommits: true`を指定しないと@atproto/syncのFirehoseが遅い
statusphereのFirehoseクラスを使った実装では`unauthenticatedCommits`が指定されていません。

このオプションはFirehoseからのイベントの内容の検証を無効化するものですが、検証の処理がかなり遅いため、PDSにレコードを追加してもいつまで経ってもイベントハンドラにイベントが渡ってこないということがありました。

処理が遅い原因については(100%推測ですが)少しメモがあるのでこちらを読んでください。
https://bsky.app/profile/mkizka.dev/post/3l3kjdvfckp2g

## おわり
思いついた順で書きましたが、これから開発を始める方の参考になれば嬉しいです。
