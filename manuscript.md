## 不具合や仕様もれを減らすための勘所とユニットテストで学ぶ簡単事例集

<p align="right">
    <strong>酒井文也（Fumiya Sakai） Twitter &amp; github: @fumiyasac</strong>
</p>

<hr>

### はじめに

アプリ開発の過程で、仕様のミスリード・不十分な仕様把握・説明の解釈ミス等で本来の意図と実際の成果物で齟齬が生まれてしまう事は、決して特別な事ではなく常に起こり得る可能性があります。

開発者がアプリが持っている機能を全て把握できているのであれば、そのような懸案は比較的少ないかもしれません。しかしながら、アプリの開発規模が大きい場合や、機能を実現する過程の中で複雑なロジックを実装する必要がある場合においては、その機能や実装を全て把握するのは困難になることが多いと思います。

加えて、規模の大小を問わず平素における機能開発時や修正対応時においても、

- 必要な処理順番は正しく実装されているか？
- 当該処理の結果は後続処理への影響を及ぼすものか？
- 各責務において返却されるデータの形は正しいか？
- Error発生時の振る舞いは正しいものであるか？
- クライアント側で実行する処理は正しいものであるか？
- 任意の処理が実行後に期待するメソッドが実行される想定となっているか？

等、**「様々な観点から見た上で正しく仕様が担保されている事が確認できる」** と嬉しいと思う様な場面に出会った経験は少なからずあるはずです。

個々の機能が正しく動作している事を担保する際には、ユニットテストを導入し活用することで、実装着中の段階で事前に仕様のミスリード・不十分な仕様把握・説明の解釈ミス等のリスクを防止する取り組みは、仕様把握や機能担保の面で有効な手段になりますし、その重要性については読者の皆様も十分にご理解頂けている部分かと思います。

単純にデータ取得が成功または失敗する場合の2パターンのケースを担保するだけでは不十分な場合、すなわち **「各々の処理内容をきちんと踏まえた形でのテストケースが必要になる」** 場合や **「パッと見ただけでは期待される結果がわかりにくい」** 場合などにおいては、その様な処理結果に応じた振る舞いがユニットテストで正しく担保されている状態が望ましい状態であると同時に、未来に関連する部分に対して改修が必要になる時にも非常に心強いはずです。

本稿では、ユニットテストをより活用する事によって、思わず見誤りやすくミスリードに繋がってしまい易い部分を防ぐポイントとご一緒に、ユニットテストの「ありがたみ」を感じられる様ないくつかの簡単な事例を、RxSwift / Quick / Nimble / SwiftyMockyを利用を想定したコードを通じて簡単に解説できればと考え得ております。

対象の読者としましては、**「ユニットテストを利用して誤りやすいケースを見抜くヒントを知りたい方」** や **「仕様ないしは機能担保の観点でのユニットテスト活用の工夫を知りたい方」** を想定しています。決して特別な事はありませんが、ほんの少しでも皆様のご参考になる事ができれば嬉しく思います。

### 1. 以降で紹介する事例で利用するコードに関する補足事項

#### ⭐️1-1. 表示するView要素に関する処理が実行される想定かを確認する

![resume_capture_01.png](https://github.com/fumiyasac/iosdc2022_pamphlet_manuscript/blob/main/images/resume_capture_01.png)

#### ⭐️1-2. 表示するView要素に関する処理が実行される想定かを確認する

![resume_capture_02.png](https://github.com/fumiyasac/iosdc2022_pamphlet_manuscript/blob/main/images/resume_capture_02.png)

```shell
# 1. SwiftyMocky CLIを準備する
$ brew install mint
$ mint install MakeAWishFoundation/SwiftyMocky
# 2. SwiftyMocky CLIを準備する
```

```swift

```

### 2. 仕様で本当に抜け漏れがないかを確認する場合

![resume_capture_03.png](https://github.com/fumiyasac/iosdc2022_pamphlet_manuscript/blob/main/images/resume_capture_03.png)

**【処理実行部分のコード抜粋】**

```swift
// Comment. 
```

**【ユニットテスト時に重要な部分のコード抜粋】**

```swift
// Comment. 
```

### 3. 言葉ではシンプルだが実際に正しいか否か見えにくい場合

![resume_capture_04.png](https://github.com/fumiyasac/iosdc2022_pamphlet_manuscript/blob/main/images/resume_capture_04.png)

**【処理実行部分のコード抜粋】**

```swift
// 準拠プロトコル名: GetSpecialContentBannersUseCase
// 依存関係: SpecialContentsBannerRepository

// MEMO: SpecialContentBanner (※Equatable準拠)
// 👉バナー表示に必要なデータを格納しているModelクラス

// MARK: - GetSpecialContentBannersUseCaseImpl（実装クラス）での処理例

func getSpecialContentBanners() -> Single<[SpecialContentBanner]> {

    // バナー表示に必要なデータを全件取得後にソート処理を実行する
    return specialContentsBannerRepository.findAll()
        .flatMap { specialContentsBanners in
            let targetSpecialContentsBanners = specialContentsBanners
                // POINT(1): データを全件取得後にソート処理を実行する
                // 👉①優先度の値が小さい > ②日付が古い > ③IDの値が小さい（※複数条件を設定）
                .sorted {
                    ($0.priority, $0.date, $0.id.value) < ($1.priority, $1.date, $1.id.value)
                }
                // POINT(2): ソート処理をした後に先頭の5件を利用する
                .prefix(5)
            return Single.just(targetSpecialContentsBanners)
        }
}
```

**【ユニットテスト時に重要な部分のコード抜粋】**

```swift
// 準拠プロトコル名: GetSpecialContentBannersUseCase
// 依存関係: SpecialContentsBannerRepository
// テスト対象: GetSpecialContentBannersUseCaseImpl

// MEMO: SpecialContentBanner
// 👉バナー表示に必要なデータを格納しているModelクラス

// MARK: - 実際のデータを想定したStubを作成

// MEMO: nowDate: 現在日付 / twoDayLaterDate: 2日後日付 / threeDayLaterDate: 3日後日付
let banner1 = SpecialContentsBanner(id: 1,　priority: 3, date: nowDate)
let banner2 = SpecialContentsBanner(id: 2, priority: 1, date: twoDayLaterDate)
let banner3 = SpecialContentsBanner(id: 3, priority: 1, date: threeDayLaterDate)
let banner4 = SpecialContentsBanner(id: 4, priority: 1, date: twoDayLaterDate)
let banner5 = SpecialContentsBanner(id: 5, priority: 2, date: twoDayLaterDate)
let banner6 = SpecialContentsBanner(id: 6, priority: 2, date: twoDayLaterDate)

// MARK: - GetSpecialContentBannersUseCaseImplSpec（実装クラスにおけるテストコード）での処理例

// MEMO: テスト実行前の準備
// 👉SpecialContentsBannerRepositoryMockはライブラリ「SwiftyMocky」を利用して自動生成する
// 👉GetSpecialContentBannersUseCaseImplを初期化する際に必要な責務のクラスをMock化したものを適用する
let specialContentsBannerRepository = SpecialContentsBannerRepositoryMock()
let getSpecialContentBannersUseCaseImpl = GetSpecialContentBannersUseCaseImpl(
    specialContentsBannerRepository: specialContentsBannerRepository
)

// MEMO: ユニットテストと対応するテストケースの作成例
describe("#getSpecialContentBanners") {

    // MARK: - getSpecialContentBannersを実行した際のテスト

    context("表示バナー用データが意図した順序かを確認するテスト") {

        beforeEach {
            // POINT(1): Mock化したRepositoryクラスの返り値を設定する
            // 👉ID順で全件データを返すことを想定 (※できれば本番環境に近いStubが望ましい)
            specialContentsBannerRepository.given(
                .findAll(
                    willReturn: Single.just([banner1, banner2, banner3, banner4, banner5, banner6])
                )
            )
        }

        it("IDが2→4→3→5→6の順番に表示バナー用データが5件取得できること") {
            // POINT(2): 条件に応じたソート処理が正しく実行されていることを確認する
            // 👉この場合は意図した順番に並ぶデータと等しいことを示せば良い
            let single = getSpecialContentBannersUseCaseImpl.getSpecialContentBanners()
            let expected = [banner2, banner4, banner3, banner5, banner6]
            let actual = try! single.toBlocking().first()
            expect(actual).to(equal(expected))
        }
    }

    // ...(※省略: 他に確認が必要なテストケースがある場合は追記する)...
}
``` 

### 4. 実は細かな点に気を配ってみると疑問が生まれる場合

![resume_capture_05.png](https://github.com/fumiyasac/iosdc2022_pamphlet_manuscript/blob/main/images/resume_capture_05.png)

**【処理実行部分のコード抜粋】**

```swift
// 準拠プロトコル名: RelatedOrRecommendedAllShopsUseCase
// 依存関係: ShopRepository

// MEMO: RelatedOrRecommendedShopsDto (※Equatable準拠)
// 👉関連店舗データ一覧またはおすすめ店舗データ一覧を格納するためのDtoクラス

// MARK: - RelatedOrRecommendedAllShopsUseCaseUseCaseImpl（実装クラス）での処理例

// ※この処理ではRxSwiftの.catchを利用することで最初の処理でエラーとなっても次の処理を実行する点がポイント
func getRelatedOrRecommendedAllShops(shopId: ShopId) -> Single<RelatedOrRecommendedShopsDto> {
    // まずはshopIdに紐づく関連店舗一覧データを全件取得する
    return shopRepository.findAllRelatedById(shopId)
        .flatMap { [weak self] relatedShops in
            guard let weakSelf = self else {
                return Single.error(CommonError.notExistSelf)
            }
            // POINT(1): 関連店舗一覧データをRelatedOrRecommendedShopsDtoに変換する処理
            // 👉getDtoByRelatedShopsメソッドで実行する処理の概要
            // ・関連店舗が0件: エラーとして扱い引き続き`.catch`の処理を実行する
            // ・関連店舗が1件以上: RelatedOrRecommendedShopsDtoに変換して返却する
            return weakSelf.getDtoByRelatedShops(relatedshops: relatedShops)
        }
        .catch { [weak self] _ in
            guard let weakSelf = self else {
                return Single.error(CommonError.notExistSelf)
            }
            // POINT(2): 関連店舗一覧データをRelatedOrRecommendedShopsDtoに変換する処理
            // 👉getDtoByRecommendedShopsメソッドで実行する処理の概要
            // ・おすすめ店舗が0件: このメソッドの処理結果全体のエラーとする
            // ・おすすめ店舗が1件以上: RelatedOrRecommendedShopsDtoに変換して返却する
            return weakSelf.getDtoByRecommendedShops()
        }
}

private func getDtoByRelatedShops(relatedshops: relatedShops) -> Single<RelatedOrRecommendedShopsDto> {
    if relatedShops.isEmpty {
        return Single.error(CommonError.notExistObject)
    } else {
        return Single.just(RelatedOrRecommendedShopsDto(shops: relatedShops))
    }
}

private func getDtoByRecommendedShops() -> Single<RelatedOrRecommendedShopsDto> {
    return shopRepository.findRecommended()
        .flatMap { [weak self] recommendedShops in
            guard let weakSelf = self else {
                return Single.error(CommonError.notExistSelf)
            }
            if recommendedShops.isEmpty {
                return Single.error(CommonError.notExistObject)
            } else {
                return Single.just(RelatedOrRecommendedShopsDto(shops: recommendedShops))
            }
        }
}
```

**【ユニットテスト時に重要な部分のコード抜粋】**

```swift
// 準拠プロトコル名: RelatedOrRecommendedAllShopsUseCase
// 依存関係: ShopRepository
// テスト対象: RelatedOrRecommendedAllShopsUseCaseImpl

// MEMO: RelatedOrRecommendedShopsDto (※Equatable準拠)
// 👉関連店舗データ一覧またはおすすめ店舗データ一覧を格納するためのDtoクラス

// MARK: - 実際のデータを想定したStubを作成

let recommendedShop1 = Shop(id: 1,　shopName: "●●●", ... , isPremium: false)
let recommendedShop2 = Shop(id: 2,　shopName: "▲▲▲", ... , isPremium: true)
let recommendedShop3 = Shop(id: 3,　shopName: "◆◆◆", ... , isPremium: false)
let recommendedShop4 = Shop(id: 4,　shopName: "■■■", ... , isPremium: true)
let recommendedShop5 = Shop(id: 5,　shopName: "★★★", ... , isPremium: false)

// MARK: - RelatedOrRecommendedAllShopsUseCaseImplSpec（実装クラスにおけるテストコード）での処理例

// MEMO: テスト実行前の準備
// 👉ShopRepositoryMockはライブラリ「SwiftyMocky」を利用して自動生成する
// 👉RelatedOrRecommendedAllShopsUseCaseImplを初期化する際に必要な責務のクラスをMock化したものを適用する
let shopRepository = ShopRepositoryMock()
let relatedOrRecommendedAllShopsUseCaseImpl = RelatedOrRecommendedAllShopsUseCaseImpl(
    shopRepository: shopRepository
)

// MEMO: ユニットテストと対応するテストケースの作成例
describe("#getRelatedOrRecommendedAllShops") {

    // MARK: - getSpecialContentBannersを実行した際のテスト

    let shopId = 9999
    let recommendedShops = [recommendedShop1, recommendedShop2, recommendedShop3, recommendedShop4, recommendedShop5] 

    context("店舗IDに紐づく関連店舗一覧はエラーではあるが、おすすめ店舗一覧が取得できた場合のテスト") {

        beforeEach {
            // POINT(1): Mock化したRepositoryクラスの返り値を設定する
            // 👉関連店舗一覧: Single.error / おすすめ店舗一覧: Single.just(recommendShops)
            shopRepository.given(
                .findAllRelatedById(.value(shopId), willReturn: SSingle.error(CommonError.notExistObject))
            )
            shopRepository.given(
                .findRecommended(willReturn: Single.just(recommendShops))
            )
        }

        it("おすすめ店舗一覧が格納されたRelatedOrRecommendedShopsDtoが返る") {
            // POINT(2): おすすめ店舗一覧を格納したRelatedOrRecommendedShopsDtoに変換する処理
            // 👉おすすめ店舗一覧取得処理がエラーでも後続のおすすめ店舗一覧取得処理が正常なら表示用データが返却されることを示せば良い
            let single = relatedOrRecommendedAllShopsUseCaseImpl.getRelatedOrRecommendedAllShops(shopId: shopId)
            let expected = RelatedOrRecommendedShopsDto(shops: recommendedShops)
            let actual = try! single.toBlocking().first()
            expect(actual).to(equal(expected))
        }
    }

    // ...(※省略: 他に確認が必要なテストケースがある場合は追記する)...
}
``` 

### 5. その他画面の中でテストがあると良さそうな部分を探り出す着想

ここからは補足として、この部分については挙動ないしは仕様の担保するためのユニットテストがあると嬉しいかもしれない事例について簡単ではありますがご紹介できればと思います。

#### ⭐️5-1. 表示するView要素に関する処理が実行される想定かを確認する

（※ノート図解が入ります）

#### ⭐️5-2. クライアント側で実行する処理が実行される想定かを確認する

（※ノート図解が入ります）

私自身が現在も実践する習慣として、UI実装を含めたある程度ボリュームがある機能開発をする際は、着手する前段として、仕様やデザインデータ等から得た情報を自分なりに図解や説明を利用しながら実装や処理に関するイメージをノートにできる限りまとめて行く所から始める様にしています。

現在与えられている情報の中から、可能な限りヒントになり得る部分を見つけ出しながら改めて言語化する作業は、大変な部分もある一方、

- エンジニア視点で発見できた、懸案事項やポイントを適切かつ丁寧にフィードバックをする必要がある場合
- デザイナーやPdMをはじめ、様々や役割を持つメンバーと協力しながら円滑に機能開発を進めていきたい場合

等においては、とても有意義な事であると共に、事後共有としての資料やドキュメントを書く際であっても心強い武器になるのではないかと私は考えております。
        
### 雑談. 数学の問題から見る間違えやすい事例

更なる補足になりますが数学の問題を題材にした、直感で正しいと思える解答と論理的に正しい解答が異なってしまう例についても簡単にご紹介致します。

（※ノート図解が入ります）

問題文を読むと何の変哲もないトランプを利用した確率の問題に見えてしまいそうですが、そう思ってしまうと不正解となってしまいます。問題文の中で注意するのは、 **「残りのカードを良く切ってから3枚抜き出したところ、3枚ともダイヤであった。」** の1文で、ポイントは1つ選んだ後に抜き出したカードの枚数とその時に選ばれたダイヤの枚数が等しい場合において、枚数が変化したならばその確率が変わる点です。（※いわゆる条件付き確率と言われるものです。）

しかしながら、与えられた情報を図解等を活用し改めて整理と言語化をしてすることで、前述した様なポイントを見抜く事や間違えやすい点を発見できる可能性が高くなるのではないかと思います。

機能開発の中でも、この問題と似た様な感じで、自分が思い描いていた形と実は違っていたという経験をした事は私自身もお恥ずかしながら何度もありました。落とし穴になりやすい部分がユニットテストによって仕様や機能をされていると本当に心強いですし、機能を担保するためのユニットテストを活用していく事は、とても有意義な取り組みであったと実際の業務を通して強く感じた次第です。

### おわりに

言葉ではシンプルに表現できるが実装をしてみると想像以上に難易度が高くなる場合や、直感では正しいと思えた結果が実はそうではない場合においては、開発者の視点からではなかなか気がつきにくいケースは起こり得る可能性がありますし、時には開発に着手する前段階で **「提示されている仕様やデザインデータを適切に疑う」** 事が必要な場合においては、よりユニットテストが活きてくるのではないかと思います。

できるだけ実際に返却される想定と近しい形のテストコードを準備し、常に仕様と照らし合わせやすい形を作りながら進めていく様な方針で進めていく癖や習慣を少しずつ付けていく様にすると良さそうに思います。

特に今回ご紹介したケースについては、とても細かな部分ではありますが、私自身も開発中の実装でハマってしまった体験があったものや、結果的に仕様のミスリードを引き起こしてしまった経験があったものの中で、ユニットテストを活用することで事前に予防ができそうだと個人的に感じた一例になります。

- コードレビューを実施した際に見誤ってしまった、ないしは思わず見誤ってしまう可能性があったもの
- 機能実装の段階で、途中で実装の誤りや仕様のミスリードを指摘された経験があったもの
- 過去に複雑なロジックや仕様であった故に、不具合の原因となってしまったもの

これらの部分で、もしユニットテストで機能を担保できる可能性がありそうな部分についてケアができる様にしていく姿勢や、**「できる所から少しずつ手厚くして不具合や仕様漏れを防止できる様な形にしていく」** という本当に小さな習慣や心がけが、大きな安心に繋がっていくのではないかと思います。

本稿の執筆に当たりましては、これまでに私が業務委託としてアプリ開発に携わらせて頂きました現場をはじめその他様々な機会を通じて得られた知見や体験等も踏まえたものを、ピックアップした上でまとめたものになりますので、この場をお借りして感謝を意を述べさせて頂きます。

業務内でもユニットテストによる仕様担保があったお陰で、業務キャッチアップの為のソースコードリーディングを通じた仕様理解がとても素早く行う事ができたり、他にも既存機能を改修する際における既存仕様の把握や実装経緯を知る必要な場面でも有意義だった経験は振り返ると多々ありました。そして、この様な体験を通して、本当に小さくかつ細かな部分に対する理解とケアの積み重ねが大きな恩恵を受けるための礎となることを改めて痛感した次第です。最後までお読み頂きまして本当にありがとうございました。

![resume_capture_06.png](https://github.com/fumiyasac/iosdc2022_pamphlet_manuscript/blob/main/images/resume_capture_06.png)
