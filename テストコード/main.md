### はじめに

はじめましての方ははじめまして、そうでない方はこんにちは。ジュニアエンジニアの若松です。社内では「じぇっと」と呼ばれています。なぜそう呼ばれているかは追々お伝えできればいいかなと思います。

さて、今回は私が最近考察している「長期間愛されてきたプロダクトにおけるテストコード」というテーマについて書いていこうと思います。

多くの人に長年使われ、進化を続けてきたプロダクトには、共通して避けがたい性質があります。それは、**時間の経過とともに技術負債が蓄積しやすい** という点です。ビジネスの要求スピード、チームメンバーの入れ替わり、短期的な視点での開発優先など、様々な要因が絡み合うため、これはある種仕方のないことかもしれません。

数ある技術負債の中でも特に開発者の頭を悩ませ、プロダクトの健全性を蝕んでいくのが、**テストコードの形骸化**です。

- テストは存在するが、失敗しても誰も直さない
- 仕様変更のたびに壊れ、そのうちコメントアウトされる
- 「このテスト、一体何を保証したいんだっけ？」と誰も答えられない

結果として、テストがあるはずなのに安心してコードを変更できず、リファクタリングは勘と手動確認に頼らざるを得なくなり、デグレが発生する。そんな状況が、開発スピードの低下を招いていきます。

そこで本記事では、ジュニアエンジニアという視点から **「将来にわたって価値を生み続けるテストコードとその運用体制とは何か？」** を整理し、考察してみます。

### 「メンテナンスされないテスト」の3つの共通点

まず、長年の運用の中で放置されがちなテストコードには、開発リソースが確保できないという組織的な課題はさておき、コード自体にいくつかの共通点があると感じています。

#### 1. 可読性の欠如
一つ目は、テストコードがプロダクションコード並みに複雑化してしまう問題です。テストコードの中に条件分岐やループ、複雑なデータ生成ロジックが入り込み、「テストなのか実装なのかが分からない」状態になっているケースです。このようなテストは、一読しただけでは「どのような振る舞いを確認したいのか」が直感的に理解できません。結果として誰も手をつけたがらなくなり、次第に塩漬けにされてしまいます。

#### 2. 環境への依存
二つ目は、特定の状況下でしか成功しない、脆いテストです。特定の実行順序やデータベースの状態、あるいは特定の日時にしか成功しないテストは、開発者の信頼を著しく損ないます。特に「ローカル環境では成功するのにCI環境では失敗する」といった不安定なテストは、「たまに落ちるテスト」から、やがて「常に無視されるテスト」へと変わっていくのです。

#### 3. 意図の不明瞭さ
三つ目は、そのテストが「何を検証したいのか」が分からない問題です。テスト名から仕様が読み取れなかったり、アサーションで使われている期待値（マジックナンバー）の根拠が不明だったりすると、仕様変更時にそのテストを「修正すべきなのか、それとも消していいのか」さえ判断できません。これは、テストが本来持つべき「仕様を語る」という重要な役割を果たせていない状態と言えるでしょう。

### 考察：長期運用に耐えうるテスト設計の原則

では、「腐らないテストコード」とはどのようなものでしょうか。私が重要だと考えている4つの観点を以下に示します。

#### 1. 「仕様書」として機能するテスト
まず、ユニットテストを単なるコードの検証ツールとしてではなく、**「メソッドの振る舞いを記述した、最小単位の生きたドキュメント」** として捉えることが重要です。

メンテナンスされないテストの例：何を検証したいのか不明瞭
```php
public function testCalculation()
{
    $result = $this->service->calculate(100, 0.1);
    $this->assertEquals(110, $result);
}
```
このテストには以下の問題があります
- 110というマジックナンバーの根拠が不明
- テスト名から仕様が読み取れない
- 数ヶ月後に見返したとき、なぜ110なのか誰も答えられない

 仕様書として機能するテストの例：

 ```php
 public function test_消費税10パーセントを含めた金額が計算される()
{
    // 商品価格と消費税率を準備
    $価格 = 100;
    $消費税率 = 0.10; // 10%
    $期待される税込価格 = 110; // 100円 + 10円(消費税)
    
    // 税込価格を計算
    $実際の税込価格 = $this->priceCalculator->calculateWithTax($価格, $消費税率);
    
    // 正しく計算されていることを検証
    $this->assertEquals($期待される税込価格, $実際の税込価格);
}
```
改善されたこととして、
- テスト名を読むだけで「消費税10%を含めた金額計算」という仕様が分かる
- 各値の意味がコメントで明示され、計算ロジックが理解できる
- 消費税率が変更されたとき、このテストを修正すべきか判断できる

このように理想は、テスト名を読むだけでその機能の仕様が分かり、テストコードを追えば実装の意図まで理解できる状態です。このように、実装の詳細が変わってもビジネス上の仕様が変わらない限りは修正が不要なテストは、プロダクトの進化に合わせて価値を提供し続けます。

#### 2. DRYよりもDAMPを意識する
プロダクションコードでは「Don't Repeat Yourself (DRY)」が原則ですが、テストコードにおいては、分かりやすさを優先する **「Descriptive And Meaningful Phrases (DAMP)」** という考え方が有効です。

過度にDRYを意識した例：共通化により文脈が失われる
 ```php
 private function createUserWithStatus($age, $expectedStatus)
{
    $user = User::factory()->create(['age' => $age]);
    $this->userStatusService->updateStatus($user);
    $this->assertEquals($expectedStatus, $user->fresh()->status);
}

public function testUserStatusUpdate()
{
    $this->createUserWithStatus(25, 'adult');
    $this->createUserWithStatus(15, 'minor');
}
 ```
この例では共通化により重複は減りましたが、各テストケースの意図が見えづらくなっています。

DAMPを意識した例：多少の冗長さを許容し、意図を明確に

```php
public function test_成人ユーザーは成人ステータスに更新される()
{
    // 20歳以上のユーザーを準備
    $adultUser = User::factory()->create(['age' => 25]);
    
    // ステータス更新を実行
    $this->userStatusService->updateStatus($adultUser);
    
    // 成人ステータスであることを検証
    $this->assertEquals('adult', $adultUser->fresh()->status);
}

public function test_未成年ユーザーは未成年ステータスに更新される()
{
    // 20歳未満のユーザーを準備
    $minorUser = User::factory()->create(['age' => 15]);
    
    // ステータス更新を実行
    $this->userStatusService->updateStatus($minorUser);
    
    // 未成年ステータスであることを検証
    $this->assertEquals('minor', $minorUser->fresh()->status);
}
```

過度な共通化は、テストケースごとの固有の文脈を隠してしまい、かえって可読性を損なうことがあります。各テストが自己完結し、多少冗長であっても一つ読めばその意図が明確にわかる、そんな「説明的」なテストこそがメンテナンス性を高めます。

#### 3. テストの独立性を保つ
長期運用されるテストで頻繁に問題となるのが、「ローカルでは通るのにCIで落ちる」「テストの実行順序を変えると失敗する」といった不安定さです。こうしたテストは開発者の信頼を失い、やがてコメントアウトされるようになります。
この問題を防ぐには、各テストが **他のテストやデータベースの状態に依存せず、独立して動作する** ことが重要です。

暗黙の前提に依存した例：
```php
public function testPostCreation()
{
    // user_id=1のユーザーが既に存在する前提
    $post = Post::create(['title' => 'Test', 'user_id' => 1]);
    $this->assertDatabaseHas('posts', ['id' => 1]);
    $user = User::find(1);
    $this->assertEquals($user->id, $post->user_id);
}
```

このテストには以下の問題があります：
- user_id => 1のユーザーが存在することを暗黙的に前提としている
- データベースがクリーンな状態（id => 1が使える状態）を前提としている
- 他のテストの実行順序や状態によって結果が変わる可能性あり

独立性を保った例：
```php
public function test_ユーザーは投稿を作成できる()
{
    // Arrange: このテストで必要なデータを明示的に準備
    $user = User::factory()->create();
    $投稿タイトル = 'テスト投稿';
    
    // Act: 投稿を作成
    $post = Post::factory()->create([
        'title' => $投稿タイトル,
        'user_id' => $user->id,
    ]);
    
    // Assert: 作成した投稿が正しく保存されていることを検証
    $this->assertDatabaseHas('posts', [
        'id' => $post->id,
        'title' => $投稿タイトル,
        'user_id' => $user->id,
    ]);
}
```
AAA (Arrange-Act-Assert) パターンに従うことで、テストの独立性が自然と保たれます：

- Arrange（準備）：このテストで必要なデータをすべて明示的に作成する
- Act（実行）：テスト対象の処理を実行する
- Assert（検証）：期待する結果を検証する

この構造により：

- シードデータや他のテストに依存しない、自己完結したテストになる
- テストの実行順序を変えても、並列実行しても、いつでも同じ結果が得られる
- CI環境でも安定して動作し、「たまに落ちるテスト」が生まれにくい
- 数ヶ月後に見返しても、前提条件がすべてコード内に明記されているため理解しやすい

テストの独立性は、長期運用において信頼性を維持するための基盤です。各テストが自己完結していることで、安心してリファクタリングでき、CIの結果を信頼できるようになります。

#### 4. テスト容易性を意識したプロダクションコード
最後は、テストコードを書くことを契機に、そもそも **「テストが書きやすいプロダクションコード」** を意識して書いていく必要があるということである。

テストしづらいコードの例：
```php
class PostService
{
    public function getPopularPosts($limit = 10)
    {
        // Repositoryをクラス内で直接インスタンス化
        $repository = new PostRepository();
        
        return $repository->findPopularPosts($limit);
    }
    
    public function createPost($title, $content, $userId)
    {
        $repository = new PostRepository();
        
        $post = $repository->create([
            'title' => $title,
            'content' => $content,
            'user_id' => $userId,
            'published' => false,
        ]);
        
        // 通知サービスもクラス内で直接インスタンス化
        $notificationService = new NotificationService();
        $notificationService->notifyFollowers($post);
        
        return $post;
    }
}
```
このコードは以下の理由でテストしづらくなっています：

- PostRepositoryが内部で生成されるため、実際のデータベースアクセスが発生する
- テスト時に「特定のデータを返す」「エラーを発生させる」などの振る舞いを制御できない
- NotificationServiceも内部で生成されるため、テスト時に実際の通知処理が走る
- 複数の依存関係が密結合しており、単体テストが困難

テストしやすいコードの例：
```php
class PostService
{
    public function __construct(
        private PostRepositoryInterface $postRepository,
        private NotificationServiceInterface $notificationService
    ) {}
    
    public function getPopularPosts(int $limit = 10): Collection
    {
        return $this->postRepository->findPopularPosts($limit);
    }
    
    public function createPost(string $title, string $content, int $userId): Post
    {
        $post = $this->postRepository->create([
            'title' => $title,
            'content' => $content,
            'user_id' => $userId,
            'published' => false,
        ]);
        
        $this->notificationService->notifyFollowers($post);
        
        return $post;
    }
}
```

テストコード：
```php
public function test_人気の投稿を取得できる()
{
    // Arrange: リポジトリをモック化
    $expectedPosts = collect([
        new Post(['id' => 1, 'title' => '人気投稿1', 'likes_count' => 200]),
        new Post(['id' => 2, 'title' => '人気投稿2', 'likes_count' => 150]),
    ]);
    
    $repositoryMock = Mockery::mock(PostRepositoryInterface::class);
    $repositoryMock->shouldReceive('findPopularPosts')
        ->with(10)
        ->andReturn($expectedPosts);
    
    $notificationMock = Mockery::mock(NotificationServiceInterface::class);
    
    // 依存関係を外部から注入
    $service = new PostService($repositoryMock, $notificationMock);
    
    // Act: 人気の投稿を取得
    $posts = $service->getPopularPosts(10);
    
    // Assert: 期待する投稿が返されることを検証
    $this->assertCount(2, $posts);
    $this->assertEquals('人気投稿1', $posts[0]->title);
}

public function test_投稿作成時にフォロワーに通知される()
{
    // Arrange: リポジトリと通知サービスをモック化
    $user = User::factory()->make(['id' => 1]);
    $createdPost = new Post([
        'id' => 1,
        'title' => 'テスト投稿',
        'content' => 'テスト本文',
        'user_id' => $user->id,
    ]);
    
    $repositoryMock = Mockery::mock(PostRepositoryInterface::class);
    $repositoryMock->shouldReceive('create')
        ->once()
        ->andReturn($createdPost);
    
    $notificationMock = Mockery::mock(NotificationServiceInterface::class);
    $notificationMock->shouldReceive('notifyFollowers')
        ->once()
        ->with(Mockery::on(fn($post) => $post->id === 1));
    
    // 依存関係を外部から注入
    $service = new PostService($repositoryMock, $notificationMock);
    
    // Act: 投稿を作成
    $post = $service->createPost('テスト投稿', 'テスト本文', $user->id);
    
    // Assert: 投稿が作成され、通知が送信されることを検証
    $this->assertEquals('テスト投稿', $post->title);
}
```
改善点：

- Repositoryや通知サービスを外部から注入できるため、テスト時にモックと差し替え可能
- データベースアクセスや通知送信などの副作用を制御でき、高速で安定したテストが実行できる
- 「人気の投稿が0件の場合」「リポジトリがエラーを返す場合」など、様々なケースを確実にテストできる
- 各クラスの責務が明確になり、テストすべき範囲が明確になる

このように、副作用を極力減らし、依存性を注入可能にすることで、テストの記述は格段に容易になります。テストしやすいコードは、結果として凝集度が高く、疎結合で、理解しやすいコードになる傾向があります。

### チームで「品質」を育てるアプローチ

優れたテストコードは、個人の努力だけで維持できるものでもないかと思います。チーム全体で品質を育む文化と仕組みも不可欠だと思っています。

#### 文化とマインドセットの醸成
良いテスト文化を育むには、まずチーム全体でのマインドセットの共有が欠かせません。単に「カバレッジのためにテストを書く」のではなく、「ソフトウェアの品質を担保し、未来の開発速度を維持するために書く」 という目的意識を揃えることが第一歩です。

開発者自身が「テストのおかげでバグを未然に防げた」「安心してリファクタリングできた」という成功体験を積むことは、テストへの大きなモチベーションになります。さらに、開発者だけでなく周囲のメンバーもテストの価値を理解し、共に品質に向き合う姿勢が欠かせません。

勉強会やペアプロで知見を広げ、失敗を責めずに改善を促す心理的安全性を土台に据える。そうすることで、職種を越えて誰もがテストを自分事として捉えられるようになり、文化は真に定着していきます。

#### ビジネスへの関心と仕様の理解
「仕様書としてのテスト」を実現するには、**エンジニアがソフトウェアで解決しようとしているビジネス課題にどれだけ関心を持っているかが鍵** となります。企画者やプロダクトマネージャーと密に連携し、ビジネスロジックやユーザーの操作フローを深く理解することで、手動で行っていた面倒なテストを自動化したり、ビジネスルールそのものをテストコードとして表現したりできます。このように、ビジネスの文脈を知り得たことをテストコード中に落とし込めるようなチーム体制を築くことが、価値の高いテスト資産を構築する上で重要です。

#### CI/CDによる継続的な運用
最後に、テストを形骸化させないための仕組み作りも不可欠です。テストの実行はCI（継続的インテグレーション）によって自動化するべきです。人間は退屈な繰り返し作業には耐えられず、手動実行はすぐに忘れ去られてしまいます。また、テストコードもプロダクションコードやドキュメントと同様に、常に更新が必要な「管理対象」であるという認識を持つことが重要です。コードとテストが常に一致し、信頼できる状態を保つ仕組みが、テストの価値を維持します。

### おわりに

現実的には、すべてのコードをテストで網羅し、その品質を100点満点に保つことは困難です。納期、人員、優先度といったビジネス上の制約の中で、私たちは常に選択を迫られます。だからこそ、今後も継続的に変更が発生しうる箇所や、ビジネスインパクトが大きくバグは許されない箇所から優先的に、質の高いテストを整備していくメリハリが大切になります。

技術負債に向き合い、テストコードを改善していくことは、単なる「掃除」ではありません。それは、**プロダクトへの理解を深める行為** そのものだと私は考えています。対象となるコードの意図や仕様を理解していなければ、意味のあるテストは書けません。自身が書いたコードはもちろん、他者が書いたコードのテストを記述・修正することで、プロダクトへの解像度は着実に上がっていきます。

アサインされたばかりのジュニアエンジニアだからこそ持てる「無知」や「素朴な違和感」を武器に変え、未来の自分とチームを助ける、価値あるテストコードを残していきたいと考えています。