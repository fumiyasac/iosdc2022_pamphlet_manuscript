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
- エラー発生時の振る舞いは正しいものであるか？
- クライアント側で実行する処理は正しいものであるか？
- 任意の処理が実行後に期待するメソッドが実行される想定となっているか？

等、**「様々な観点から見た上で正しく仕様が担保されている事が確認できる」** と嬉しいと思う様な場面に出会った経験は少なからずあるはずです。

個々の機能が正しく動作している事を担保する際には、ユニットテストを導入し活用することで、実装着中の段階で事前に仕様のミスリード・不十分な仕様把握・説明の解釈ミス等のリスクを防止する取り組みは、仕様把握や機能担保の面で有効な手段になりますし、その重要性については読者の皆様も十分にご理解頂けている部分かと思います。

単純にデータ取得が成功または失敗する場合の2パターンのケースを担保するだけでは不十分な場合、すなわち **「各々の処理内容をきちんと踏まえた形でのテストケースが必要になる」** 場合や **「パッと見ただけでは期待される結果がわかりにくい」** 場合などにおいては、その様な処理結果に応じた振る舞いがユニットテストで正しく担保されている状態が望ましい状態であると同時に、未来に関連する部分に対して改修が必要になる時にも非常に心強いはずです。

本稿では、ユニットテストをより活用する事によって、思わず見誤ってしまいそうな部分やミスリードに繋がってしまい易い部分を防ぐポイントとご一緒に、ユニットテストの「ありがたみ」を感じられる様ないくつかの簡単な事例を、RxSwift / Quick / Nimble / SwiftyMockyを利用を想定したコードを通じて簡単に解説できればと考えております。

対象の読者としましては、**「ユニットテストを利用して誤りやすいケースを見抜くヒントを知りたい方」** や **「仕様ないしは機能担保の観点でのユニットテスト活用の工夫を知りたい方」** を想定しています。決して特別な事はありませんが、ほんの少しでも皆様のご参考になる事ができれば嬉しく思います。

### 1. 以降で紹介する事例で利用するコードに関する補足事項

#### ⭐️1-1. 表示するView要素に関する処理が実行される想定かを確認する

![resume_capture_01.png](https://github.com/fumiyasac/iosdc2022_pamphlet_manuscript/blob/main/images/resume_capture_01.png)

#### ⭐️1-2. 表示するView要素に関する処理が実行される想定かを確認する

![resume_capture_02.png](https://github.com/fumiyasac/iosdc2022_pamphlet_manuscript/blob/main/images/resume_capture_02.png)

__【実行コマンドのおおまかな手順】__

```shell
# 1. SwiftyMocky CLIを準備する
$ brew install mint
$ mint install MakeAWishFoundation/SwiftyMocky

# 2. インストールができた事を確認する 👉 図解の様な形でUsage:やCommands:が表示される事を確認
$ swiftymocky

# 3. Mock自動生成処理を実行する 👉 Mock.generated.swiftに生成したMockが追記されていく形
$ swiftymocky generate
```

#### ⭐️1-3. 仕様で本当に抜け漏れがないかを確認する

![resume_capture_03.png](https://github.com/fumiyasac/iosdc2022_pamphlet_manuscript/blob/main/images/resume_capture_03.png)

### 2. 言葉ではシンプルだが実際に正しいか否か見えにくい場合

![resume_capture_04.png](https://github.com/fumiyasac/iosdc2022_pamphlet_manuscript/blob/main/images/resume_capture_04.png)

#### ⭐️2-1. 処理実行部分のコード抜粋

```swift
// ----------
// 準拠プロトコル名: GetSpecialContentBannersUseCase
// 依存関係: SpecialContentsBannerRepository
// ----------

// MEMO: SpecialContentBanner (※Equatable準拠)
// 👉バナー表示に必要なデータを格納しているModelクラス

// MARK: - GetSpecialContentBannersUseCaseImpl（実装クラス）での処理例

func getSpecialContentBanners() -> Single<[SpecialContentBanner]> {

    // バナー表示に必要なデータを全件取得後にソート処理を実行する
    return specialContentsBannerRepository.findAll()
        .flatMap { specialContentsBanners in
            let targetSpecialContentsBanners = specialContentsBanners
                // POINT(1): データを全件取得後にソート処理を実行する
                // 👉①優先度の値が小さい > ②日付が古い > ③IDの値が小さい（複数条件を設定）
                // ※ここはタプルを利用した書き方をしているが、コードをぱっと見ただけでは、少しわかりにくいかもしれない...
                .sorted {
                    ($0.priority, $0.date, $0.id.value) < ($1.priority, $1.date, $1.id.value)
                }
                // POINT(2): ソート処理をした後に先頭の5件を利用する
                .prefix(5)
            return Single.just(targetSpecialContentsBanners)
        }
}
```

#### ⭐️2-2. ユニットテスト時に重要な部分のコード抜粋

```swift
// ----------
// 準拠プロトコル名: GetSpecialContentBannersUseCase
// 依存関係: SpecialContentsBannerRepository
// テスト対象: GetSpecialContentBannersUseCaseImpl
// ----------

// MEMO: SpecialContentBanner
// 👉バナー表示に必要なデータを格納しているModelクラス

// MARK: - 実際のデータを想定したStubを作成

// MEMO: nowDate: 現在日付 / twoDayLaterDate: 2日後日付 / threeDayLaterDate: 3日後日付
let banner1 = SpecialContentsBanner(id: 1, priority: 3, date: nowDate)
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

### 3. 実は細かな点に気を配ってみると疑問が生まれる場合

![resume_capture_05.png](https://github.com/fumiyasac/iosdc2022_pamphlet_manuscript/blob/main/images/resume_capture_05.png)

#### ⭐️3-1. 処理実行部分のコード抜粋

```swift
// ----------
// 準拠プロトコル名: RelatedOrRecommendedAllShopsUseCase
// 依存関係: ShopRepository
// ----------

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

#### ⭐️3-2. ユニットテスト時に重要な部分のコード抜粋

```swift
// ----------
// 準拠プロトコル名: RelatedOrRecommendedAllShopsUseCase
// 依存関係: ShopRepository
// テスト対象: RelatedOrRecommendedAllShopsUseCaseImpl
// ----------

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
                .findAllRelatedById(.value(shopId), willReturn: Single.error(CommonError.notExistObject))
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

### 4. その他画面の中でテストがあると良さそうな部分を探り出す着想

画面要素構築や機能実装を進めていく前段として、重要になりそうな部分を書き出して自分なりにまとめることで、仕様ないしは挙動を明確にし、それらを担保をするためのユニットテストに繋げていく際のアプローチ例を簡単でありますがご紹介できればと思います。最初は大雑把な形でも構わないので、疑問点や間違えそうと感じた点を拙くとも自分の言葉にし、ユニットテストで予防可能な余地がある部分を少しずつ探していく様なアプローチをしていくと個人的には良さそうに思います。

#### ⭐️4-1. クライアント側で実行する処理イメージや振る舞いのポイントをまとめる

![idea_note_01.jpg](https://github.com/fumiyasac/iosdc2022_pamphlet_manuscript/blob/main/images/idea_note_01.jpg)

#### ⭐️4-2. 表示するView要素に関する振る舞いの想定をまとめる

![idea_note_02.jpg](https://github.com/fumiyasac/iosdc2022_pamphlet_manuscript/blob/main/images/idea_note_02.jpg)

私自身が現在も実践する習慣の一例としてUI実装を含めたある程度ボリュームがある機能開発をする際は、着手する前段として、仕様やデザインデータ等から得た情報を自分なりに図解や説明を利用しながら実装や処理に関するイメージをノートにできる限りまとめて行く所から始める様にしています。

現在与えられている情報の中から、可能な限りヒントになり得る部分を見つけ出しながら改めて言語化する作業は、大変な部分もある一方、

- エンジニア視点で発見できた、懸案事項やポイントを適切かつ丁寧にフィードバックをする必要がある場合
- デザイナーやPdMをはじめ、様々や役割を持つメンバーと協力しながら円滑に機能開発を進めていきたい場合

等においては、とても有意義な事であると共に事後共有としての資料やドキュメントを書く際であっても心強い武器になるのではないかと私は考えております。

### まとめ

言葉ではシンプルに表現できるが実装をしてみると想像以上に難易度が高くなる場合や、直感では正しいと思えた結果が実はそうではない場合においては、開発者の視点からではなかなか気がつきにくいケースは起こり得る可能性がありますし、時には開発に着手する前段階で **「提示されている仕様やデザインデータを適切に疑う」** 事が必要な場合においては、よりユニットテストが活きてくるのではないかと思います。

加えて、その有用性を生かしていくためにできるだけ実際に返却される想定と近しい形のテストコードを準備し、常に仕様と照らし合わせやすい形を作りながら進めていく様な方針で進めていく癖や習慣を少しずつ付けていく様にすると良さそうに思います。

特に今回ご紹介したケースについては、とても細かな部分ではありますが、私自身も開発中の実装でハマってしまった体験があったものや、結果的に仕様のミスリードを引き起こしてしまった経験があったものの中で、ユニットテストを活用することで事前に予防ができそうだと個人的に感じた一例になります。

- コードレビューを実施した際に見誤ってしまった、ないしは思わず見誤ってしまう可能性があったもの
- 機能実装の段階で、途中で実装の誤りや仕様のミスリードを指摘された経験があったもの
- 過去に複雑なロジックや仕様であった故に、不具合の原因となってしまったもの

これらの部分で、もしユニットテストで機能を担保できる可能性がありそうな部分についてケアができる様にしていく姿勢や、**「できる所から少しずつ手厚くして不具合や仕様漏れを防止できる様な形にしていく」** という本当に小さな習慣や心がけが大きな安心に繋がっていくのではないかと思います。

本稿の執筆に当たりましては、これまでに私が業務委託としてアプリ開発に携わらせて頂きました現場をはじめその他様々な機会を通じて得られた知見や体験等も踏まえたものを、ピックアップした上でまとめたものになりますので、この場をお借りして感謝を意を述べさせて頂きます。業務内でもユニットテストによる仕様担保があったお陰で、業務キャッチアップの為のソースコードリーディングを通じた仕様理解がとても素早く行う事ができたり、他にも既存機能を改修する際における既存仕様の把握や実装経緯を知る必要な場面でも有意義だった経験は後々に振り返ってみると多々あった様に思います。そして、これらの体験を通して本当に小さくかつ細かな部分に対する理解とケアの積み重ねが大きな恩恵を受けるための礎となることを改めて痛感した次第です。最後までお読み頂きまして本当にありがとうございました。

![resume_capture_06.png](https://github.com/fumiyasac/iosdc2022_pamphlet_manuscript/blob/main/images/resume_capture_06.png)

### 雑談. 数学の問題から見る間違えやすい事例

![mathmatics_example.jpg](https://github.com/fumiyasac/iosdc2022_pamphlet_manuscript/blob/main/images/mathmatics_example.jpg)

雑談として数学の問題を題材にした、直感で正しいと思える解答と論理的に正しい解答が異なってしまう例についても簡単にご紹介致します。

問題文を読むと何の変哲もないトランプを利用した確率の問題に見えてしまいそうですが、そう思ってしまうと不正解となってしまいます。問題文の中で注意するのは、 **「残りのカードを良く切ってから3枚抜き出したところ、3枚ともダイヤであった。」** の1文で、ポイントは1つ選んだ後に抜き出したカードの枚数とその時に選ばれたダイヤの枚数が等しい場合において、枚数が変化したならばその確率が変わる点です。（※いわゆる条件付き確率と言われるものです。）

しかしながら、与えられた情報を図解等を活用し改めて整理と言語化をしてすることで、前述した様なポイントを見抜く事や間違えやすい点を発見できる可能性が高くなるのではないかと思います。

機能開発の中でも、この問題と似た様な感じで、自分が思い描いていた形と実は違っていたという経験をした事は私自身もお恥ずかしながら何度もありました。落とし穴になりやすい部分がユニットテストによって仕様や機能をされていると本当に心強いですし、機能を担保するためのユニットテストを活用していく事は、とても有意義な取り組みであったと実際の業務を通して強く感じた次第です。

<hr>

### あとがきと原稿では書ききれなかった補足事項の紹介

ここまでお読み頂きまして、本当にありがとうございました。iOSDC Japan 2022はパンフレット原稿の寄稿のみの形となってしまいましたが、わずかながらでもこの内容が何かのお役に立つ事ができれば本当に嬉しく思います。私自身は決して大きな事はできませんが、些細なアウトプットであったとしても絶やさずに継続していきたいと思いますので、引き続きよろしくお願い致します。

ここからは、ページ数の関係で原稿に掲載する事ができなかった事項や、もう少し深堀りをしておきたいと感じていた事項について補足という形で触れていきます。

#### 参考1. Presentation層でのユニットテスト例

掲載している原稿内では、UseCase層（BusinessLogic層）に関するユニットテストに焦点を当てて解説をしてきましたが、ここではPresentation層のユニットテストに関しても少し触れていきたいと思います。

こちらは以前にお世話になった現場の中で教わった手法になりますが、後述するテストケースの事例の様な形でPresentation層のテストケースがあることによって、想定する画面の振る舞いについても検討がつきやすくなる形することもできるので、特に1つの画面内において画面状態や表示を切り替えるためのトリガーが多くある場合等においては、とても有益ではないかと感じています。

__【ViewController ⇄ Presenter間の処理における重要ポイントの抜粋】__

```swift
// ----------
// Presentation層のユニットテストをできる様な形とするために、下記の様なProtocolを定義します。
// (1) ●●●View:
// 👉 表示画面用のViewControllerに準拠させた上で、View要素構築や表示に関する処理を実行します。
// (2) ●●●Coodinator:
// 👉 表示画面用のViewControllerに準拠させた上で、画面遷移に関する処理を実行します。
// (3) ●●●Presenter:
// 👉 実行クラス(※SearchShopPresenterImpl)に準拠させた上で、必要なUseCase層（BusinessLogic層）の処理を実施したら、View要素の構築や表示または画面遷移処理を実行するという流れを作ります。
//
// Presentation層のユニットテストはどの様な形になるか？ 
// 👉 あるPresentation層に定義したメソッドを実行した際に、対象のView要素構築や表示に関する処理が実行されることを示せば良い
// ※ (1)及び(2)はMockに置き換える必要がある部分なので、自動Mock生成の対象とします。
// ----------

// MEMO: 対象のViewControllerクラス＆PresenterImplクラスへ準拠させるプロトコル例

// sourcery: AutoMockable
protocol SearchShopView: AnyObject {
    func setup(checkShowSpecial: Bool)
}

// sourcery: AutoMockable
protocol SearchShopCoodinator: ScreenCoordinator {
    func moveToSearchShopSpecial()
    func moveToSearchShopKeyowrd()
    func moveToSearchShopCategory()
    func moveToSearchShopLocation()
}

protocol SearchShopPresenter: AnyObject {
    func setup(
         view: SearchShopView,
         coodinator: SearchShopCoodinator
     )
    func viewDidLoadTrigger()
    func coodinateSearchShopSpecialTrigger()
    func coodinateSearchShopKeyowrdTrigger()
    func coodinateSearchShopCategoryTrigger()
    func coodinateSearchShopLocationTrigger()
}

// ----------
// SearchShopViewController.swiftにおけるポイント抜粋
// ※この事例ではRxCocoaはあまり利用しない方針を想定しています
// ----------
final class SearchShopViewController: UIViewController {

    // MARK: - Property

    private let presenter: SearchShopPresenter

    // ...(※省略: 必要なプロパティがあれば定義する)...

    // MARK: - Initializer

    init?(coder: NSCoder, presenter: SearchShopPresenter) {
        self.presenter = presenter
        super.init(coder: coder)
    }

    required init?(coder: NSCoder) {
        fatalError()
    }

    // MARK: - Override

    override func viewDidLoad() {
        super.viewDidLoad()

        // MEMO: View要素＆画面遷移関連の処理をPresenterで実行できる様にするための準備をする
        presenter.setup(
            view: self,
            coodinator: self
        )

        // MEMO: viewDidLoad実行時にPresenterで実行したい処理がある場合に実行する
        presenter.viewDidLoadTrigger()
    }    

    // MARK: - @IBAction

    // ボタンタップ時にPresenter処理を実行する
    @IBAction func searchShopSpecialButtonAction(_ sender: UIButton) {
        // MEMO: ボタンタップ時や各種Delegate処理時においてもなるべくPresenterを介して実行する形にする
        presenter.coodinateSearchShopSpecialTrigger()
    }

    // ...(※省略: 必要なメソッドがあれば定義する)...
}

// MARK: - SearchShopCoodinator

extension SearchShopViewController: SearchShopCoodinator {

    func moveToSearchShopSpecial() {
        // TODO: 特集検索画面へ遷移する
    }

    func moveToSearchShopKeyowrd() {
        // TODO: キーワード検索画面へ遷移する
    }

    func moveToSearchShopCategory() {
        // TODO: カテゴリー検索画面へ遷移する
    }

    func moveToSearchShopLocation() {
        // TODO: 位置検索画面へ遷移する
    }
}

// MARK: - SearchShopView

extension SearchShopViewController: SearchShopView {

    func setup(checkShowSpecial: Bool) {
        // TODO: 画面表示のハンドリング処理を実施する
    }
}

// ----------
// SearchShopPresenterImplにおけるポイント抜粋
// ----------
final class SearchShopPresenterImpl: SearchShopPresenter {

    // MARK: - Property

    private weak var view: SearchShopView?
    private weak var coodinator: SearchShopCoodinator?

    // MEMO: 初期化時に必要な責務に関する事例
    // ① CheckShowSpecialUseCase: return Single<Bool>
    // 👉 スペシャルコンテンツ一覧の表示可否を返す
    // ② ImmediateSchedulerType: MainScheduler.instance
    // 👉 この処理についてはメインスレッドで実行する
    private let checkShowSpecialUseCase: CheckShowSpecialUseCase
    private let mainScheduler: ImmediateSchedulerType

    private let disposeBag = DisposeBag()

    // MARK: - Initializer

    init(
        checkShowSpecialUseCase: CheckShowSpecialUseCase,
        mainScheduler: ImmediateSchedulerType
    ) {
        self.checkShowSpecialUseCase = checkShowSpecialUseCase
        self.mainScheduler = mainScheduler
    }

    // MARK: - Function

    func setup(
        view: SettingsView,
        coodinator: SettingsCoodinator
    ) {
        self.view = view
        self.coodinator = coodinator
    }

    func viewDidLoadTrigger() {
        checkShowSpecialUseCase.execute()
            .observe(
                on: mainScheduler
            ).subscribe(
                .onSuccess: { [weak self] checkShowSpecial in
                    guard let weakSelf = self else {
                        return
                    }
                    // MEMO: ViewController側のView要素構築や表示処理を実行する
                    weakSelf.view?.setup(checkShowSpecial: checkShowSpecial)
                },
                onFailure: { error in
                    print(error)
                }
            ).disposed(by: disposeBag)
    }

    func coodinateSearchShopSpecialTrigger() {
        // MEMO: 該当画面への画面遷移処理を実行する
        coodinator?.moveToSearchShopSpecial()
    }

    func coodinateSearchShopKeyowrdTrigger() {
        // MEMO: 該当画面への画面遷移処理を実行する
        coodinator?.moveToSearchShopKeyowrd()
    }

    func coodinateSearchShopCategoryTrigger() {
        // MEMO: 該当画面への画面遷移処理を実行する
        coodinator?.moveToSearchShopCategory()
    }

    func coodinateSearchShopLocationTrigger() {
        // MEMO: 該当画面への画面遷移処理を実行する
        coodinator?.moveToSearchShopLocation()
    }
}
```

__【ViewController ⇄ Presenter間の処理を検証するためのユニットテスト】__

```swift
// MARK: - SearchShopPresenterImplSpec（実装クラスにおけるテストコード）での処理例

final class SearchShopPresenterImplSpec: QuickSpec {

    // MARK: - Override

    override func spec() {
        
        // MEMO: テスト実行前の準備
        // 👉CheckShowSpecialUseCaseMockはライブラリ「SwiftyMocky」を利用して自動生成する
        // 👉SearchShopPresenterImplを初期化する際に必要な責務のクラスをMock化したものを適用する
        let checkShowSpecialUseCase = CheckShowSpecialUseCaseMock()
        let mainScheduler = CurrentThreadScheduler.instance

        // MEMO: ユニットテストと対応するテストケースの作成例
        describe("SearchShopPresenterImpl") {

            // MARK: - viewDidLoadTriggerを実行した際のテスト

            describe("#viewDidLoadTrigger") {
                // MEMO: SearchShopPresenterImplを初期化する際に必要なSearchShopView・SearchShopCoodinatorについてもクラスをMock化したものを適用する
                let view = SearchShopViewMock()
                let coodinator = SearchShopCoodinatorMock()
                let target = SearchShopPresenterImpl(
                    checkShowSpecialUseCase: checkShowSpecialUseCase,
                    mainScheduler: mainScheduler
                )
                target.setup(
                    view: view,
                    coodinator: coodinator
                )
                let checkShowSpecial = true 

                // POINT(1): Mock化したUseCaseクラスの返り値を設定する
                // 👉trueまたはfalserを返すことを想定 (※画面表示処理等に合わせて確認したいテストケースに応じて設定するのがポイント)
                beforeEach {
                    checkShowSpecialUseCase.given(
                        .execute(
                            willReturn: Single.just(checkShowSpecial)
                        )
                    )
                }

                // POINT(2): Presenter側の処理に対応するView側の処理が実行されていることを確認する
                // 👉.verifyメソッドを利用してSearchShopViewに定義したsetup(checkShowSpecial: Bool)が実行する想定
                it("画面描画処理のsetup(checkShowSpecial: Bool)が1回実行される") {
                    target.viewDidLoadTrigger()
                    view.verify(
                        .setup(checkShowSpecial: .value(checkShowSpecial)),
                        count: .once
                    )
                }
            }

            // MARK: - coodinateSearchShopSpecialTriggerを実行した際のテスト

            describe("#coodinateSearchShopSpecialTrigger") {
                // MEMO: viewDidLoadTrigger実行時のテストケースを同様にテスト実行前の準備をする
                let view = SearchShopViewMock()
                let coodinator = SearchShopCoodinatorMock()
                let target = SearchShopPresenterImpl(
                    checkShowSpecialUseCase: checkShowSpecialUseCase,
                    mainScheduler: mainScheduler
                )
                target.setup(
                    view: view,
                    coodinator: coodinator
                )

                // POINT(3): Presenter側の処理に対応するCoodinator側の処理が実行されていることを確認する
                // 👉.verifyメソッドを利用してSearchShopCoodinatorに定義したmoveToSearchShopSpecial()が実行する想定
                it("画面描画処理のmoveToSearchShopSpecial()が1回実行される") {
                    target.coodinateSearchShopSpecialTrigger()
                    coodinator.verify(
                        .moveToSearchShopSpecial(),
                        count: .once
                    )
                }
            }
        }
    }
}
```

#### 参考2. その他参考資料

__【AndroidアプリでにおけるUnitTestとの比較】__

- [Testing Android apps based on Dagger and RxJava Droidcon UK](https://www.slideshare.net/fabio_collini/testing-android-apps-based-on-dagger-and-rxjava-droidcon-uk)
- [Keddit — Part 9: Unit Test with Kotlin (Mockito, RxJava & Spek)](https://medium.com/android-news/keddit-part-9-unit-test-with-kotlin-mockito-spek-76709812e3b6)
- [Improve your tests with Kotlin in Android — (Pt.1)](https://proandroiddev.com/improve-your-tests-with-kotlin-in-android-pt-1-6d0b04017e80)
- [Improve your tests with Kotlin in Android — (Pt.2)](https://proandroiddev.com/improve-your-tests-with-kotlin-in-android-pt-2-f3594e5e7bfd)
- [AndroidのテストをSpek+Mockitoで書こう](https://qiita.com/k_keisuke/items/815ced486e8cdff8670d)

__【DIコンテナに関する事例紹介】__

- [自前でDIコンテナを作ってみる試みとRxSwiftを利用した構成への適用を試してみる](https://qiita.com/fumiyasac@github/items/8d6b77c3547b8b7839ad)
- [【実装MEMO】PropertyWrappersの機能を利用したDependency Injectionのコードに触れた際の備忘録](https://fumiyasakai.medium.com/%E5%AE%9F%E8%A3%85memo-propertywrappers%E3%81%AE%E6%A9%9F%E8%83%BD%E3%82%92%E5%88%A9%E7%94%A8%E3%81%97%E3%81%9Fdependency-injection%E3%81%AE%E3%82%B3%E3%83%BC%E3%83%89%E3%81%AB%E8%A7%A6%E3%82%8C%E3%81%9F%E9%9A%9B%E3%81%AE%E5%82%99%E5%BF%98%E9%8C%B2-b269bc914b7a)

#### 雑談. Single.zipについてのちょっとしたお話

RxSwiftを利用した実装に対して最初は強く感じていた抵抗感が薄れ、比較的複雑なロジックであってもある程度実装する事に慣れてきた際に遭遇したケースについて簡単に紹介できればと思います。

それはiOSアプリのTOP画面構造を改修する際のことでした。アプリにおけるTOP画面はいわば __「アプリで一番最初に目にする画面である」__ と同時に、様々なビジネスロジックを組み合わせた処理もあり、セクション毎にバラエティに富んだレイアウトもある、複雑なものになりやすい部分だと思います。私が実装していた際は、TOP画面用のPresentation層に対しての処理にて、10個のUseCase層（BusinessLogic層）の処理を必要なものでした。

私はその時に、`Single.zip`を利用してそのままUseCase層の処理を繋げようとしたらエラーが発生しました。改めて気になったので、RxSwiftの内部実装を調べてみると、1つの`Single.zip`で処理を並列に取り扱う事ができる最大値は8つという事でした。そしてよくよく考えてみると、全てのUseCase層における処理結果を待ち全ての処理が成功した場合にのみ画面表示処理をする仕様に対し、本当にそうあるべきなのか？という部分についても改めて考え直すきっかけとなりました。

- 処理が失敗しても、画面表示処理は継続するべきか？
- 処理が失敗したら、画面表示自体もエラーと見なすべきか？

という観点は、画面表示やUI実装にも深く関わる部分でもあるので、この部分は平素の開発においても配慮する必要がありそうに思います。

意外な所からRxSwiftで理解が不十分だった点や内部処理を追いかけていくと知らなかった点を改めて知る様になると、その奥深さに改めて驚いた次第です。

__【RxSwiftのコードを覗くとこんな感じになっていました】__

```swift
public static func zip<E1, E2, E3, E4, E5, E6, E7, E8>(
    _ source1: PrimitiveSequence<Trait, E1>, 
    _ source2: PrimitiveSequence<Trait, E2>, 
    _ source3: PrimitiveSequence<Trait, E3>, 
    _ source4: PrimitiveSequence<Trait, E4>, 
    _ source5: PrimitiveSequence<Trait, E5>, 
    _ source6: PrimitiveSequence<Trait, E6>,
    _ source7: PrimitiveSequence<Trait, E7>, 
    _ source8: PrimitiveSequence<Trait, E8>, 
    resultSelector: @escaping (E1, E2, E3, E4, E5, E6, E7, E8) throws -> Element) -> PrimitiveSequence<Trait, Element> {
    return PrimitiveSequence(
        raw: Observable.zip(
            source1.asObservable(), 
            source2.asObservable(), 
            source3.asObservable(), 
            source4.asObservable(), 
            source5.asObservable(), 
            source6.asObservable(), 
            source7.asObservable(), 
            source8.asObservable(), 
            resultSelector: resultSelector
        )
    )
}
```
