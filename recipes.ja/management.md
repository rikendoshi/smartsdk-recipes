# デプロイされたサービスの管理

このページでは、レシピを使ってデプロイされたサービスの管理に共通の
メカニズムについて説明します。

## マルチサイト・デプロイメント


特定の領域 (*A1, A2, ..., AN*) にノードが分散しているクラスタがあり、
サービスのレプリカ *S* の展開を特定の領域でのみ行いたい場合は、
**配置制約** (placement constraints) を使うことでこれを達成できます。

たとえば、マラガ、マドリード、チューリッヒに VMS で構成されるクラスタが
あるが、法的規制のためにデータベースのレプリカのデプロイメントをスペインの
境界内に限定したいとします。
（**免責事項**：これは単なる簡略化です。データ保護の規制を正しく遵守する
方法について常に通知する必要があります。）

まず、デプロイメントを実行したいクラスタのノードにラベルを定義する必要が
あります。この場合のあなたのラベルは地域によって定義することができます。
任意のSwarm Managerノードに接続して、次のコマンドを実行します。


```bash
docker node update --label-add region=ES malaga-0
docker node update --label-add region=ES madrid-0
docker node update --label-add region=ES madrid-1

docker node update --label-add region=CH zurich-0
docker node update --label-add region=CH zurich-1
```

必要があります。つまり、デプロイする前にレシピを編集する必要があります。
例えば、
[MongoDB](https://github.com/smartsdk/smartsdk-recipes/blob/master/recipes/utils/mongo-replicaset/docker-compose.yml)
の場合、`deploy` 部分は次のようになります :

```yaml
mongo:
  ...
  deploy:
    placement:
      constraints:
        - region == ES
```

これはほんの簡単な例です。複数のタグを持つことができ、それらを組み合わせて
すべてのタグを持つノードで配置が行われるようにします。この機能の詳細に
ついては、
[サービス配置に関する公式のdockerのドキュメント](https://docs.docker.com/engine/swarm/services/#control-service-placement)
を参照してください。

**注 :** これらの機能が動作するためには、swarm cluster と Docker 17.04
以降のマネージャ・ノードにアクセスする必要があります。

## スケーラビリティ

ご存知のとおり、各レシピは一連のサービスをデプロイします。各サービスは、
**ステートレス**または**ステートフル**の2つのタイプのいずれかになります。
サービスの種類を区別する方法は、サービスの実装を調べて、サービス自体に
データの永続性が必要かどうかを判断することです。

ポイントは、サービスを拡張する方法はサービスの種類によって異なります。

Docker を使用して**ステートレス**・サービスを拡張するのはとても簡単です。
レプリカの数を増やすだけで済みます。制約違反がないと仮定すると
(前のセクションを参照)、**ステートレス**・サービスごとに動的に多かれ
少なかれレプリカを設定することができます。

```bash
docker service scale orion=5
```

スケーリング・プロセスに関する詳細は、
[ここ](https://docs.docker.com/engine/swarm/swarm-tutorial/scale-service/)
にあります。

**ステートフル**・サービスの拡張に関しては、特効薬はありません。
議論されているサービスに常に依存します。

たとえば、Docker は2つのタイプのサービス展開を処理します :
[ここ](https://docs.docker.com/engine/swarm/how-swarm-mode-works/services/#replicated-and-global-services)
に説明があるように、**レプリケート**と**グローバル**です。
**レプリケート**・サービスは前の例のように拡張できますが、
**グローバル**・サービスを拡張する唯一の方法 (ノードあたり最大1つの
インスタンスがあることを意味します) は、ノードを追加することです。
**グローバル**・サービスを縮小するには、ノードを削除するか制約を適用する
ことができます。上記の*マルチサイト・デプロイメント*のセクションを
参照してください。

どちらの場合も、**ステートフル**・サービスでは、誰かがすべてのインスタンス
間でデータ層を調整し、レプリケーション、パーティショニング、および
分散システムで通常見られるあらゆる種類の問題に対処する責任があります。

最後に、これらのスケーラビリティを考慮するために、レシピはどのサービスが
**ステートレス**でどれがそうでないかを適切に文書化する必要があります。
また、それらの**ステートフル**・サービスをどのように拡張できるかについての
メモも含めるべきです。
