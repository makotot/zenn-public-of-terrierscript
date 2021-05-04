---
title: expo(React Native)における私的バージョニングルール検討（あるいは如何にしてOTAと付き合うか）
emoji: '🥑'
type: idea
topics:
  - reactnative
  - expo
  - javascript
  - versioning
published: true
terrier_export_from: https://note.com/terrierscript/n/n3baa3404d76b
---

:::message
この記事は2020/08/27にnoteに投稿した記事を移植したものです
:::

expoを使い始めて、React Nativeで何度か挫折していた自分にとって基本的に本当に最高で最高なのだけど、どうしてもreleaseChannelとかリリース管理周りだけちょっと困りどころがあったのでどう管理すると良いかについて考えてみた。

## 困りごと

expoにおいて本番ビルドをする場合、ビルドと同じタイミングでpublishされてしまう。expoはOTAの仕組みがあるため、更新がかかってしまう。

つまり同じreleaseChannelにしてしまうとストアに審査用のビルドをするとその時点でOTAがかかってしまいうる

[設定によってOTAをdisableにすることも可能](https://docs.expo.io/versions/latest/config/app/)だが、いざというときには便利なので完全にオフにしてしまうのももったいない。

それほどモバイルアプリに明るいわけではないので手探りではあるものの、とりあえず自分なりに「こういうバージョニングにしたらいいとこ取りが出来るんではないか？」を考えてみた。もっと良いのがあったら教えてほしい

## オレオレexpoバージョニングルール

ユーザー向けのアプリであればライブラリとは違い、それほど厳密にセマンティックバージョニングに寄せる意味も無いのだが、せっかく馴染みのあるルールなのでこれを一部をベースにカスタマイズする形を考えてみた。

x.y.zのような形式になぞらえると下記のようになる。

- x(major) - 破壊的変更。API等互換性がなくなる場合など。
- y(minor) - ストアリリースのたびに更新する。日付などでも良いかも？
- z(patch) - ストアリリースを伴わないビルド更新。審査用再ビルドやOTAでのアップデート

majorバージョンはほとんどセマンティックバージョニングと同じで、何らかの破壊的変更を伴う場合に利用する。ライブラリではないのでそう多くはないが、例えばサーバーのAPIバージョンに互換性が無い場合などだろう。

minorバージョンがセマンティックバージョンと大きく変えている。どうしてもモバイルアプリの場合、ストアリリースという大きな周期があるので、これは外したくない。なのでminorバージョンを利用してみることにした。もし毎週リリースなどをしているのであれば、日付などにしても良いかもしれない。

patchバージョンはOTAに利用するために開けておく。再審査や審査前のビルド時に更新することもあるだろう。

これに習うとおそらくmajorが最も利用されるので、1.234.2などになっていくだろう。minorを日付にするなら1.20200901.1などだろう。

## Release Channelの運用

次はこのルールに合わせてpatch更新のpublishだけOTAが適応されるrelease-channel運用を設定してみる。

自前でも良かったが[semver-extract](https://www.npmjs.com/package/semver-extract)というライブラリがソースも小さくちょうど良かったのでこれを使ってみる。下記のような具合でバージョンをechoしてくれる

```jsx
$ semver-extract --pjson --minor -x
1.2.x
```

これをpackage.jsonに仕込む

```jsx

"scripts": {
	"release-channel": "echo v$(semver-extract --pjson --minor -x)",
	"build:ios": "expo build:ios --release-channel=production-$(npm run release-channel --silent)"
}
```

このRelease Channelのルールで進めれば、 produciton-v1.2 などで更新されるので、 1.2.3 → 1.3.0 のようなストアリリースをまたぎたいような場合はOTAによる更新は起きない。一方で 1.2.3 に問題があった場合は 1.2.4 にする、のような場合はOTAを利用もできる。

# おまけ: バージョンに絡むtips

## おまけ1: ProductionとStagingはどうする？

これはもうslugやbundleIdentifierごと分けてしまったほうが楽かなと思った。

staging用にoverrideしたい部分をjsonに記載しておき、app.config.jsをカスタマイズして上書きするようにしてみる

```jsx
// app-staging.json
{
  "name": "My App [Staging]",
  "displayName": "My App [Staging]",
  "expo": {
    "name": "My App [Staging]",
    "description": "My App [Staging]",
    "slug": "staging-my-app",
    "ios": {
      "bundleIdentifier": "my.app.staging",
    }
  }
}
```

```jsx
// app.config.js
import merge from "deepmerge"
import stagingConfig from "./app-staging.json"

const overrideEnv = (baseConfig) => {
  if (process.env.BUILD_ENV === "staging") {
    return merge.all([baseConfig, stagingConfig])
  }
  return baseConfig
}
export default ({ config }) => {
  const conf = overrideEnv(config)
  return conf
}
```

app.config.jsでreleaseChannelを取得できれば良いのだが、残念ながら出来ないようなので package.jsonで BUILD_ENV を渡しておこう

```jsx
// package.json
"scripts": {
  "build:ios:staging": "BUILD_ENV=staging expo build:ios --release-channel=staging-$(npm run release-channel --silent)"
}
```

## おまけ2: react-native-versionをpostversionに仕掛けておくと便利

普通にやっていると app.json のbuildNumberとversionCodeを手動更新しなきゃいけなくてめんどくさいなと思っていたのだが[react-native-version](https://github.com/stovmascript/react-native-version)を使うとほとんど意識しなくて良くなった

[https://github.com/stovmascript/react-native-version](https://github.com/stovmascript/react-native-version)

```jsx
"scripts": {
	"postversion": "react-native-version"
}
```

これで yarn version や npm version をするとその後更新される。便利

## まとめ

基本的にこれはexpoを前提に考えてみたバージョンだが、OTAがあるような場合ならそこそこ使えるかもしれない。

expoは最高。