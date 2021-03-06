コンシステントハッシュ法 [Kar97]

# 20章データセンターでのロードバランシング

- 19章はデータセンター間でのユーザトラフィックのバランスに取り方をみた。
- 20章はデータセンター内のロードバランシングに焦点をあてた章
  - 1つのデータセンター内で一連のクエリを分配する処理のアルゴリズム
- 書いてあること
  - トランスポート、アプリケーション層の話
- 書いてないこと
  - ネットワークインターフェース、インターネット層の話
- クエリはデータセンター内にある複数のサービスを利用するためのもの
  - サービスは、均一で交換可能な大量のサーバプロセスとして実装されていて、さまざまなマシン上で動いている
- 典型的なサービスは100から1000のプロセスで構成されている
  - プロセスはバックエンドタスク(もしくはバックエンド)と呼ばれている
  - その他のタスクはクライアントタスクと呼ばれて、バックエンドタスクへの接続を保持する
- 受信した各クエリに対して、クライアントタスクは処理させるバックエンドを決定しないといけない
  - クライアントはTCP/UDPの組み合わせの上で実装されたプロトコルを使って、バックエンドと通信をする
- 多くのサービスを支えるGoogleの手法を一般化した形で紹介されている。そして実際にGoogle内部では多くの部分で適用されている。

- ペイロード(ヘッダを取り除いたデータ本体)

- 20.1 理想的なケース
  - 理想的なケースは、サービスの負荷は全てのバックエンドタスクに対して完璧に分散させられ、最小の負荷のバックエンドタスクと最大の負荷のバックエンドタスクのCPU消費量が完全に等しい時
  - 1つのデータセンターに対して送信できるトラフィックは、最も負荷の高いタスクがそのキャパシティの上限に達するまで。
  - 下の図の期間では過負荷のタスクを避けるために、データセンター間でのロードバランシングのアルゴリズムは、これ以上のトラフィックをデータセンターに送らないようにする。
![タスクごとの負荷分布の推移の2つのシナリオ](http://landing.google.com/sre/book/images/srle-2001.jpg)
  - 下の図の左のグラフはタスク0を除けば、使われていないキャパシティがありリソースの無駄になっている。
![図20-2 2  つのシナリオにおいて消費されているCPUと無駄になっているCPUのヒストグラム](http://landing.google.com/sre/book/images/srle-2002.jpg)
  - すべてのタスクiに対する(CPU[0] - CPU[i])=無駄になっているCPU
- 20.2 不良タスクの特定:フロー制御とレイムダック
  - クライアントからのリクエストを受け取るバックエンドタスクの決定方法を考える前に、まずはバックエンドのプール中で健全に動作していないタスクを特定し、避けなければならない。
- 20.2.1 健全ではないタスクに対するシンプルなアプローチ:フロー制御
  - 前提1:クライアントタスクはバックエンドタスクへの接続ごとに送信したアクティブなリクエスト数を追跡しているものとする
  - 前提2:このアクティブリクエスト数が設定された限度に達したら、クライアントはそのバックエンドを健全な状態ではないとし、それ以上はリクエストを送らない。
  - このフロー制御は、あるバックエンドタスクが高負荷でリクエストがつまったら、他のバックエンドにリクエストが渡る。
    - これではタスクのリソースに余裕があるけど、処理に時間がかかりレスポンスが返せていないものに対しても、フロー制御が働いてしまうことも起きる。
  - タスクの状態が健全ではないのか、レスポンスが遅いだけなのか判断できない問題がある。
  
- 20.2.2 不健全なタスクへの確実なアプローチ:レイムダック状態
  - クライアントからみたバックエンドタスクの状態3つ
  - 健全
    - バックエンドタスクは正しく初期化を終え、リクエストを処理している 
  - 接続拒否
    - バックエンドタスクが応答していない。ex)タスクがシャットダウン中etc
  - レイムダック
    - バックエンドタスクはポート上でリスニングしており、処理を行うこともできるものの、クライアントに対してリクエストの送信を止めるよう明示的に要求している
    - レイムダック状態では、タスクはアクティブなクライアントへブロードキャストする。
    - アクティブなクライアントでも、GoogleのRPC接続は定期的にUDPのヘルスチェックを送信するので、全クライアントへ1~2回のラウンドトリップで伝播される。
  - レイムダックでは、おおよそ以下の手順でアクティブなリクエストに対してエラーを返さずにバックエンドタスクをシャットダウンできるようになる。
  1. ジョブスケジューラはバックエンドタスクに SIGTERM シグナルを送信する
  2. バックエンドタスクはレイムダック状態に入り、クライアントに対して新しいリクエストは他のバックエンドに送るよう要求する。この処理は、SIGTERMのハンドラ中で明示的に呼ばれるRPC実装の中のAPI呼び出しを通じて行われます。
  3. バックエンドタスクがレイムダック状態に入る前に処理が始まっていた(あるいは、バックエンドはレイムダック状態に入っていたものの、そのことを検知する前にクライアント が送信した)リクエストは、通常通り処理される。
  4. レスポンスがクライアントに返されていくにつれて、レイムダック状態になっているバックエンドタスクのアクティブなリクエスト数は、徐々に 0 に近づいている。
  5. 設定された期間が経過すると、バックエンドタスクは正常終了するか、ジョブスケジューラによってkillされる。この期間には、典型的なすべてのリクストを十分終えられるだけの長さを取ります。この値はサービスによって異なりますが、簡単な目安としてはクライアントの複雑さに応じて10 秒から150 秒にすると良い。
  - この戦略で、リクエストの準備ができていないバックグラウンドタスクにもクライアントが接続できるようになる。
  - この戦略をとらないと、バックグラウンドタスクが接続要求するのはリクエストを処理する準備ができてからになってしまう。
  - 処理を始める準備ができたら、バックエンドタスクはクライアントに対して、そのことを明示的に知らせてくれる。
- 20.3 サブセットの設定によるコネクションプールの制限
  - ロードバランサーは健全性だけでなく、サブセットの設定も考慮に入れる必要がある。
  - サブセットの設定は、クライアントのタスクがやりとりをする潜在的なバックエンドタスクのプールを制限すること
  - サブセットの設定をする理由
    - 1つのクライアントから莫大な数のバックエンドタスクに接続したりすることをさけるため
    - 1つのバックエンドタスクが莫大な数のクライアントタスクから接続されたりするような状況をさけるため
    - クライアントとバックエンドタスクの接続を保つのにメモリとCPUが必要になるので、大量のリソースが使われることをさけるため
- 20.3.1 適切なサブセットの選択
  - 適切なサブセットの選択とは、各クライアントが接続するバックエンドタスク数(サブセット)の大きさと、選択のためのアルゴリズムを選ぶこと
  - サブセットを大きくする例
    - クライアントがバックエンドよりもはるかに少数の場合。
      - この場合、クライアントあたりのバックエンド数は、トラフィックを全く受け取らないバックエンドのタスクが生じないよう十分な大きさにすべき
    - クライアントのジョブにおいて、負荷の不均衡が頻繁に生じる(1つのクライアントタスクが他のクライアントに比べて多くのリクエストを送信するような)場合
      -  クライアントが時おり大量のリクエストを一気に送信するような状況では、そのクライアントに割り当てられたサブセットに集中するので、比較的多めに用意された利用可能なバックエンドタスク間で均等に負荷が配分されるようにするには、サブセットのサイズを大きくしなければいけない.
- 20.3.2 サブセットの選択アルゴリズム:ランダムなサブセットの選択
  - 各クライアントがバックエンドのリストを一旦ランダムにシャッフルして、名前解決可能で健全なバックエンドをリストから選択することでサブセットを埋める方法
  - シャッフルを一回行って、バックエンドをリストの先頭から選択していけば、再起動や障害が発生しているバックエンドを除外できる
  - 実際は、負荷を不均衡にしてしまう
    - 300のクライアント
    - 300のバックエンド
    - サブセットのサイズは30%(各クライアントは90のバックエンドに接続)
  - 以下の縦軸は接続数。接続数がバラけて負荷が不均衡になっている
![300クライアント、300バックエンド、サブセットのサイズは30%に設定した場合の接続の分布](http://landing.google.com/sre/book/images/srle-2003.jpg)
  - サブセットのサイズを10%など小さくするとさらに差が広がる
![300クライアント、300バックエンド、サブセットのサイズは10%に設定した場合の接続の分布](http://landing.google.com/sre/book/images/srle-2004.jpg)
  - 結論としては、ランダムなサブセットの選択で利用可能なすべてのタスク間で比較的均等に負荷分散させるには、サブセットのサイズは75%が必要。(300 * 0.75 = 225(クライアントごとのバックエンドが225になる))大規模システムにはタスクに対するクライアントの接続数の分散が大きくなり過ぎて現実的ではない。
- 20.3.3 サブセットの選択アルゴリズム:決定的なサブセット選択
  - ランダムなサブセット選択へのGoogleの解決策

```python
def Subset(backends, client_id, subset_size): 
    subset_count = len(backends) / subset_size # サブセット数(バックエンドタスク数を希望するサブセットのサイズで割った値)
   
    # クライアントをラウンドにグループ化する。
    # 各ラウンドは、シャッフルされた同じリストを使用する。 
    round = client_id / subset_count
    random.seed(round)
    random.shuffle(backends)

    # subset_idは現在のクライアントに対応する。 
    subset_id = client_id % subset_count
   
    start = subset_id * subset_size
    return backends[start:start + subset_size]
```

  - クライアントタスクを「ラウンド」に分割
  - round i にはsubset_count x i から始まる連続したバックグラウンドタスクが含まれる
  - subset_countは、サブセット数(バックエンドタスク数を希望するサブセットのサイズで割った値)
  - 各ラウンド内では、各バックエンドが厳密に一つのクライアントに割り当てられる。(ただし、最後のラウンドだけはそうならないことがある。クライアント数が足りないために割り当てられないバックグラウンドが生じるかもしれないため)
  - 例えば、バックエンドタスクが[0, 11]のように12あるとして、希望するサブセットのサイズが3であれば、各ラウンドには4つのバックエンドが含まれることになる。(subset_count = 12/3)
  - 以下はクライアントが10の時に生成するラウンド
    - round = 10 / 3 (client_id / subset_count)
    - ラウンド0: [0,6,3,5,1,7,11,9,2,4,8,10]
    - ラウンド1: [8,11,4,0,5,6,10,3,2,7,9,1]
    - ラウンド2: [8,3,7,2,1,4,9,10,6,5,0,11]
  - 注意すべきなのは、それぞれのラウンドはリスト全体の各バックエンドを 1 つのクライアントにしか割り当てていないこと。(クライアントがなくなった最後のラウンドを除く)
  - この例では、すべてのバックエンドが 2つか3つのクライアントを割り当てられることになる。
  - このリストはシャッフルしないと、クライアントに連続したバックエンドのタスクのグループが割り当てられてしまい、それらがまとめて一時的に利用できなくなるかもしれないから(例えば、バックエンドジョブが最初から最後に向かって順番にアップデートされるなどの理由で)
  - それぞれのラウンドのシャッフルでは異なるシードを使わないと、1つのバックエンドが障害を起こしたら、その影響は積み重なり、状況は悪化しかねないから。
  - サブセット中のN個のバックエンドがダウンしたら、それらの負荷は残りの(subnet_size - N)個のバックエンドに配分される
  - 各ラウンドごとに異なるシャッフルを使えば、同じラウンド内のクライアントは同じシャッフル済リストを使うが、ラウンドが異なるクライアントは異なるシャッフル済リストを使うことになる
  - 以下から、アルゴリズムはバックエンドのシャッフル済リストと、希望するサブセットのサイズに基づくサブセットの定義を構築する
    - Subset[0] = shuffled_backends[0]、経由はshuffled_backends[2]
    - Subset[1] = shuffled_backends[3]、経由は shuffled_backends[5]
    - Subset[2] = shuffled_backends[6]、経由は shuffled_backends[8]
    - Subset[3] = shuffled_backends[9]、経由は shuffled_backends[11]
  - shuffled_backend は各クライアントが生成したシャッフル後のリスト
  - クライアントタスクに割り当てるサブセットは、そのクライアントタスクのラウンド内での位置に対応するサブセット(サブセットが4つならclient[i]に対して(i % 4))
    - client[0], client[4], client[8]はsubset[0]を使用
    - client[1], client[5], client[9]はsubset[1]を使用
    - client[2], client[6], client[10]はsubset[2]を使用
    - client[3], client[7], client[11]はsubset[3]を使用
  - ラウンドが異なるクライアントの shuffled_backendsの値(subsetの値)はことなり、同じラウンド内のクライアントは異なるサブセットを使うので、接続の負荷は均一に配分される
  - バックエンドの総数が希望するサブセットのサイズで割りきれない場合には、いくつかのサブセットが他に比べてやや大きくなることになるが、ほとんどの場合1つのバックエンドに割り当てられるクライアント数の差異は、大きくても1にしかならない
  - 以下の図は300 クライアントがそれぞれ 300 中の 10 バックエンドに接続する先ほどの例での分布は非常に良い結果になっており、各バックエンドはぴったり同じ数の接続を受けている
![定的なサブセット選択による300クライアントから300中の10バックエンドへの接続の分布](http://landing.google.com/sre/book/images/srle-2005.jpg)
- 20.4 ロードバランシングのポリシー
  - ロードバランシングのポリシーとは
  - サブセット中のどのバックエンドタスクにリクエストを転送するかを選択するためにクライアントタスクが使う仕組み
  - バックエンドの状態をまったく考慮に入れないきわめてシンプルなもの(例えばラウンドロビン)
  - バックエンドに関する情報をもっと取り入れて動作するもの(例えば最小負荷ラウンドロビンや重み付きラウンドロビン)
- 20.4.1 シンプルなラウンドロビン
  - サブセット中の問題なく接続できた各バックエンドでレイムダック状態になってないものに、各クライアントがリクエストをラウンドロビンで送信していく
  - 最小と最大の負荷のタスクでCPU負荷に最大2倍の開きがでることがわかった。
  - この開きは以下のリスクなどで生じる
    - 小さなサブセット
    - ばらつきのあるクエリのコスト
    - マシンのばらつき
    - パフォーマンスに影響する予測不可能な要因
- 20.4.1.1 小さなサブセット
  - ラウンドロビンによる負荷の分配がうまくいかない最も単純な理由の一つは、すべてのクライアントが同じペースでリクエストを発行するわけではないということ
  - クライアントによってサブセットの大小によってバックエンドの負荷に差がでてしまう。
- 20.4.1.2 ばらつきのあるクエリのコスト
  - 多くのサービスで、リクエスト毎にその処理に必要なリソース量は大きく異ることは少ない
  - Googleでは最も負荷の大きいリクエストと小さいリクエストでは1000倍以上のCPUの消費に差がある
  - さらにクエリのコストが変動することもある。(昨日低負荷だったクエリが、次の日に高負荷になったり)
  - リクエスト毎にリソース要求に幅があると、バックエンドのタスクの中に他よりも負荷の大きいリクエストを受信するものが出てくる
  - 負荷の大きいリクエストとのリソースの競合によって他のリクエストのレイテンシにも悪影響がでる。
- 20.4.1.3 マシンのばらつき
  - 同じデータセンター内にある全てのマシンが同じだとは限らず、性能や処理能力が大きく異るかもしれない
  - 対応策として、グーグルではGCU(Google Compute Unit)という仮想的なCPUの処理速度の仮想的な単位をつくり、CPUの処理速度モデルの標準になった。各CPUはパフォーマンスにもとづいてGCUへのマッピングが管理されるようになった。
- 20.4.1.4 パフォーマンスに影響する予測不可能な要因
  - パフォーマンスに影響を与える予測不可能な要因の2つ
    - 近隣との対立 
      - 別のチームが走らせた無関係なプロセスから競合リソースの競合などで、プロセスのパフォーマンスは大きな影響を受けたりする
    - タスクの再起動
      - 再起動の際には、タスクは数分間にわたってかなりのリソースを必要とする。
      - 対策として、サーバ側に、再起動時に一定期間レイムダック状態にして、パフォーマンスが通常状態になるまで暖気するコードを追加する。
- 20.4.2 最小負荷ラウンドロビン
  - シンプルなラウンドロビンの代わりになるアプローチには、各クライアントタスクが自分のサブセット中の各バックエンドに対するアクティブなリクエスト数を追跡し、アクティブなリクエスト数が最小のタスク群の中でラウンドロビンを行うという方法がある
  - あるクライアントがバックエンドタスクとして t0 から t9 を使っており、現時点では各バックエンドに対するアクティブリクエスト数が以下のようになっているものとする

| t0 | t1 | t2 | t3 | t4 | t5 | t6 | t7 | t8 | t9 |
| -- | -- | -- | -- | -- | -- | -- | -- | -- | -- |
| 2  | 1  | 0  | 0  | 1  | 0  | 2  | 7  | 0  | 1  |

  - 新しいリクエストに対し、クライアントは接続数が最小になっているタスクだけに絞って対象となりうるバックエンドタスクのリストを作成し(t2、t3、t5、t7、t8)、そのリストの中からバックエンドを1つ選択。t2が選択されたものとする。クライアントの接続状態のテーブルは以下となる。

| t0 | t1 | t2 | t3 | t4 | t5 | t6 | t7 | t8 | t9 |
| -- | -- | -- | -- | -- | -- | -- | -- | -- | -- |
| 2  | 1  | 1  | 0  | 1  | 0  | 2  | 0  | 0  | 1  |

  - この時点でリクエストがいずれも終了していないとすれば、次のリクエストの際のバックエンドの候補のプールは t3、t5、t7、t8 になっている。新しいリクエストを 4 つ発行したところまで話を飛ばす。その間にまだいずれのリクエストも終了していないとすれば、接続状態のテーブルは以下のようになっている。

| t0 | t1 | t2 | t3 | t4 | t5 | t6 | t7 | t8 | t9 |
| -- | -- | -- | -- | -- | -- | -- | -- | -- | -- |
| 2  | 1  | 1  | 1  | 1  | 1  | 2  | 7  | 1  | 1  |

  - この時点で、バックエンドの候補の集合は t0 と t6 を除くすべてのタスクになっている。ただし、仮に t4 に対するリクエストが終了していれば、t4 は「アクティブなリクエストは 0」という状態になり、新しいリクエストは t4 に割り当てられることになる。
  - この実装のラウンドロビンは最小のアクティブリクエスト数のタスクの集合に対して。
  - このフィルタリングがないと、この分配のポリシーでもリクエストを十分に分散させることはできず、本来利用可能なはずのバックエンドタスクの一部が使われないままになってしまう状況を避けることはできない。
  - 最小負荷ラウンドロビンの2つの重要な制約
  - アクティブなリクエスト数は、バックエンドの処理能力をうまく示さないことがある
    - スペックの高いバックエンドと低いバックエンドそれぞれのリクエスト処理時間(ネットワークからのレスポンス:レイテンシ)が同程度だと、リクエスト数を2倍処理できるバックエンドがあったとしても最小負荷ラウンドロビンは、どちらのバックエンドの負荷も同じだと判断してしまう。 
  - それぞれのクライアントのアクティブなリクエスト数には、同じバックエンドへの他のクライ アントからのリクエストは含まれない
    - 各クライアントタスクのバックエンドタスクの状態に対する視界はきわめて限定的であり、自分自身のリクエストしか見えていない
- 20.4.3 重み付きラウンドロビン
  - 重み付きラウンドロビンは、バックエンドから提供される情報を判断のプロセスに取り入れることでシンプルで最小負荷のラウンドロビンを改造したもの。
  - 原理
    - 各クライアントタスクはサブセット中の各バックエンドの「処理能力」スコアを保持
    - リクエストはラウンドロビンで分配されるが、クライアントは重みに比例してバックエンドへリクエストを配分
    - レスポンスに、バックエンドはクエリとエラーの毎秒のレートに加えて、自身の利用状況を伝える
    - クライアントは定期的に処理能力スコアを調整して、バックエンドが処理に成功したリクエスト数や、その利用コストに基づいてバックエンドを選択
    - 失敗したリクエストは、将来の判断に影響するペナルティになる
  - 以下はバックエンドタスクのランダムなサブセットのCPUの利用状況で、クライアントが最小負荷ラウンドロビンから重み付きラウンドロビンに切り替えた時点のもので、最小負荷と最大負荷のタスク間の幅が劇的に狭まっている。
  - ![重み付きラウンドロビンの有効化の前後でのCPUの分布](http://landing.google.com/sre/book/images/srle-2006.jpg)
