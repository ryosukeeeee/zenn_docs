---
title: "serverless-rustプラグインでRustアプリをデプロイするためのworkaround"
emoji: "✏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust","AWS","Lambda"]
published: false
---

# はじめに


Rustで書かれたアプリーケーションのためのLambdaランタイムを提供する
[lambda-runtime](https://github.com/awslabs/aws-lambda-rust-runtime) crateですが、
2021年3月に0.3.0がリリースされました🚀

https://github.com/awslabs/aws-lambda-rust-runtime

なんですが、0.2.0のころには使えていた[serverless-rust](https://github.com/softprops/serverless-rust)というserverless frameworkのプラグインが使えなくなって困ったので、どうやって解決したかをまとめました。

https://github.com/softprops/serverless-



# ビルド用のDockerイメージの準備

serverless-rustはRustアプリーケーションをビルドするために、Rustツールチェーンがプリインストールされた[softprops/lambda-rust](https://github.com/softprops/lambda-rust)の最新版イメージをビルドに使います。
ところが、Docker Hubで公開されている`softprops/lambda-rust:latest`の
Rustのバージョンが古い（1.45）ので、`lambda_runtime "0.3.0"`のビルドに失敗します。

::: message
ビルドエラーの詳細は↓こちらの折り畳みにまとめてあります。
:::

:::details ビルドエラー詳細

`softprops/lambda-rust:latest`でビルドしたときのログ

```shell
docker run --rm -v ${PWD}:/code -v ${HOME}/.cargo/registry:/cargo/registry -v ${HOME}/.cargo/git:/cargo/git softprops/lambda-rust:latest
    Updating crates.io index
 Downloading crates ...
  Downloaded num-integer v0.1.44
  Downloaded pin-utils v0.1.0
  Downloaded proc-macro-hack v0.5.19
  （省略）
   Compiling num_cpus v1.13.0
   Compiling socket2 v0.4.0
   Compiling time v0.1.44
error[E0658]: `match` is not allowed in a `const fn`
   --> /root/.cargo/registry/src/github.com-1ecc6299db9ec823/socket2-0.4.0/src/lib.rs:156:9
    |
156 | /         match address {
157 | |             SocketAddr::V4(_) => Domain::IPV4,
158 | |             SocketAddr::V6(_) => Domain::IPV6,
159 | |         }
    | |_________^
    |
    = note: see issue #49146 <https://github.com/rust-lang/rust/issues/49146> for more information

error: aborting due to previous error

For more information about this error, try `rustc --explain E0658`.
error: could not compile `socket2`.

To learn more, run the command again with --verbose.
warning: build failed, waiting for other jobs to finish...
error: build failed
```

socket2はhyperが依存しているcrate

```shell
cargo tree -i socket2
socket2 v0.4.0
└── hyper v0.14.7
    └── lambda_runtime v0.3.0
        └── rust_lambda_sample v0.1.0 
```


コンパイルエラーが発生する箇所はここ

```rust
impl Domain {
    /// Domain for IPv4 communication, corresponding to `AF_INET`.
    pub const IPV4: Domain = Domain(sys::AF_INET);


    /// Domain for IPv6 communication, corresponding to `AF_INET6`.
    pub const IPV6: Domain = Domain(sys::AF_INET6);


    /// Returns the correct domain for `address`.
    pub const fn for_address(address: SocketAddr) -> Domain {
        match address {
            SocketAddr::V4(_) => Domain::IPV4,
            SocketAddr::V6(_) => Domain::IPV6,
        }
    }
}
```

https://github.com/rust-lang/socket2/blob/b1479fffa0749147eabd15ad9038d0f9a0cc7825/src/lib.rs#L147-L161

Rust 1.46.0から`const fn`の中で`match`が使えるようになった。
すなわち、Rust 1.46.0以上では上記のコードは合法である。

参考: [Announcing Rust 1.46.0 | Rust Blog](https://blog.rust-lang.org/2020/08/27/Rust-1.46.0.html)

:::


[lambda-rust](https://github.com/softprops/lambda-rust)のGitHubのmasterブランチはRust 1.51に対応しています（2021年5月26日時点）。
（commit sha: `e6137ddbac36d104236407eb537c6c03a16a30fa`）

ということで、GitHub上のDockerfileを参照してイメージをビルドします。

```sh
docker build -t softprops/lambda-rust:1.51 https://github.com/softprops/lambda-rust.git#e6137ddbac36d104236407eb537c6c03a16a30fa
```

# ビルド&デプロイ

::: message 
serverless frameworkを使わずに、Lambda関数をビルドしたい方は↓の折り畳みをご覧ください。
:::

::: details serverless frameworkを使わずにAWS CLIでビルドする方法

プロジェクトルートで↓のコマンドを実行します。ビルドにはかなり時間がかかるので気長に待ちましょう。

```shell
docker run --rm \
  -v ${PWD}:/code \
  -v ${HOME}/.cargo/registry:/cargo/registry \
  -v ${HOME}/.cargo/git:/cargo/git \
  softprops/lambda-rust:1.51
```

ビルドが終わると、`./target/lambda/release/`にzipファイルが作られています。
これをlambdaにデプロイします。


```shell
aws lambda create-function \
  --function-name rustTest \
  --zip-file fileb://./target/lambda/release/rust_lambda_sample.zip \
  --role arn:aws:iam::0123456789012345:role/YourLambdaRole \
  --runtime provided.al2 \
  --handler doesnt.matter \
  --profile yourprofile
```

デプロイに成功したら、呼び出してみましょう。

```shell
aws lambda invoke \
  --cli-binary-format raw-in-base64-out \
  --function-name rustTest \
  --payload '{"command": "Hello World"}' \
  --profile yourprofile \
  output.json
cat output.json
```

:::


npmで配布されているserverless-rustの最新バージョン（2021年5月26日時点）は0.3.8です。
このバージョンではlambdaランタイムに`Amazon Linux`を指定しているので、実行時にエラーが発生します。

::: message alert
ランタイムにAmazon Linux 2を指定する必要があります
:::

serverless-rustのGitHubリポジトリでは開発が進んでおり、[masterブランチ](https://github.com/softprops/serverless-rust/tree/ebe43ceacfb7f770569e98e3dfb4bbb6eba0d88d)（commit sha: ebe43ceacfb7f770569e98e3dfb4bbb6eba0d88d）を使うとランタイムに`Amazon Linux 2`を設定することができます。


Rustアプリプロジェクトのルートでserverless-rustのGitHubリポジトリからコミットを指定してインストールしましょう。

```shell
npm i -D https://github.com/softprops/serverless-rust#ebe43ceacfb7f770569e98e3dfb4bbb6eba0d88d
```


ビルドに使うDockerイメージをカスタマイズするために、serverless.ymlに↓を追加します。

```yml
custom:
  rust:
    # custom docker tag
    dockerTag: '1.51'
    #  custom docker image
    dockerImage: 'softprops/lambda-rust'
```

ここまでの準備が完了していれば、デプロイできます。

```shell
sls deploy
```

# おわりに

これで手軽にLambdaにRustアプリケーションをデプロイできるようになりました。これをきっかけにRustユーザーが増えてくれたら嬉しいです。

`lambda-rust`も`serverless-rust`もGitHubでは`lambda-runtime: 0.3.0`への対応が済んでいるので、Docker Hubやnpmといった配信基盤にいつかは反映されるでしょう。ここで紹介した手順はそのときまでの期間限定の対応です。

# 参照
https://github.com/awslabs/aws-lambda-rust-runtime

https://github.com/softprops/lambda-rust

https://github.com/softprops/serverless-rust
