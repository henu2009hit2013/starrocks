---
displayed_sidebar: docs
---
import Experimental from '../_assets/commonMarkdown/_experimental.mdx'

# Apache® Pulsar™ からのデータを継続的にロードする

<Experimental />

StarRocks バージョン 2.5 から、Routine Load は Apache® Pulsar™ からのデータを継続的にロードすることをサポートしています。Pulsar は、ストアとコンピュートの分離アーキテクチャを持つ、分散型のオープンソースの pub-sub メッセージングおよびストリーミングプラットフォームです。Routine Load を介して Pulsar からデータをロードすることは、Apache Kafka からデータをロードすることに似ています。このトピックでは、CSV 形式のデータを例に、Routine Load を介して Apache Pulsar からデータをロードする方法を紹介します。

## サポートされているデータファイル形式

Routine Load は、Pulsar クラスターから CSV および JSON 形式のデータを消費することをサポートしています。

> NOTE
>
> CSV 形式のデータについては、StarRocks は 50 バイト以内の UTF-8 エンコードされた文字列をカラムセパレータとしてサポートしています。一般的に使用されるカラムセパレータには、カンマ (,) 、タブ、パイプ (|) があります。

## Pulsar に関連する概念

**[Topic](https://pulsar.apache.org/docs/2.10.x/concepts-messaging/#topics)**

Pulsar のトピックは、プロデューサーからコンシューマーにメッセージを送信するための名前付きチャネルです。Pulsar のトピックは、パーティション付きトピックと非パーティション付きトピックに分かれています。

- **[パーティション付きトピック](https://pulsar.apache.org/docs/2.10.x/concepts-messaging/#partitioned-topics)** は、複数のブローカーによって処理される特別なタイプのトピックであり、より高いスループットを可能にします。パーティション付きトピックは実際には N 個の内部トピックとして実装されており、N はパーティションの数です。
- **非パーティション付きトピック** は、単一のブローカーによってのみ提供される通常のタイプのトピックであり、トピックの最大スループットを制限します。

**[Message ID](https://pulsar.apache.org/docs/2.10.x/concepts-messaging/#messages)**

メッセージのメッセージ ID は、メッセージが永続的に保存されるとすぐに [BookKeeper インスタンス](https://pulsar.apache.org/docs/2.10.x/concepts-architecture-overview/#apache-bookkeeper) によって割り当てられます。メッセージ ID は、台帳内のメッセージの特定の位置を示し、Pulsar クラスター内で一意です。

Pulsar は、コンシューマーが consumer.*seek*(*messageId*) を通じて初期位置を指定することをサポートしています。しかし、Kafka コンシューマーオフセットが長整数値であるのに対し、メッセージ ID は `ledgerId:entryID:partition-index:batch-index` の4つの部分で構成されています。

したがって、メッセージから直接メッセージ ID を取得することはできません。その結果、現在、**Routine Load は Pulsar からデータをロードする際に初期位置を指定することをサポートしておらず、パーティションの開始または終了からデータを消費することのみをサポートしています。**

**[Subscription](https://pulsar.apache.org/docs/2.10.x/concepts-messaging/#subscriptions)**

サブスクリプションは、メッセージがコンシューマーにどのように配信されるかを決定する名前付きの設定ルールです。Pulsar は、コンシューマーが複数のトピックに同時にサブスクライブすることもサポートしています。トピックには複数のサブスクリプションを持つことができます。

サブスクリプションのタイプは、コンシューマーが接続する際に定義され、すべてのコンシューマーを異なる設定で再起動することで変更できます。Pulsar には4つのサブスクリプションタイプがあります：

- `exclusive` (デフォルト): 単一のコンシューマーのみがサブスクリプションに接続できます。1人の顧客のみがメッセージを消費できます。
- `shared`: 複数のコンシューマーが同じサブスクリプションに接続できます。メッセージはコンシューマー間でラウンドロビン配信され、特定のメッセージは1つのコンシューマーにのみ配信されます。
- `failover`: 複数のコンシューマーが同じサブスクリプションに接続できます。非パーティション付きトピックまたはパーティション付きトピックの各パーティションに対してマスターコンシューマーが選ばれ、メッセージを受信します。マスターコンシューマーが切断されると、すべての（未確認および後続の）メッセージが次のコンシューマーに配信されます。
- `key_shared`: 複数のコンシューマーが同じサブスクリプションに接続できます。メッセージはコンシューマー間で配信され、同じキーまたは同じ順序キーを持つメッセージは1つのコンシューマーにのみ配信されます。

> Note:
>
> 現在、Routine Load は exclusive タイプを使用しています。

## Routine Load ジョブを作成する

以下の例は、Pulsar で CSV 形式のメッセージを消費し、Routine Load ジョブを作成して StarRocks にデータをロードする方法を説明します。詳細な手順と参考情報については、[CREATE ROUTINE LOAD](../sql-reference/sql-statements/loading_unloading/routine_load/CREATE_ROUTINE_LOAD.md) を参照してください。

```SQL
CREATE ROUTINE LOAD load_test.routine_wiki_edit_1 ON routine_wiki_edit
COLUMNS TERMINATED BY ",",
ROWS TERMINATED BY "\n",
COLUMNS (order_id, pay_dt, customer_name, nationality, temp_gender, price)
WHERE event_time > "2022-01-01 00:00:00",
PROPERTIES
(
    "desired_concurrent_number" = "1",
    "max_batch_interval" = "15000",
    "max_error_number" = "1000"
)
FROM PULSAR
(
    "pulsar_service_url" = "pulsar://localhost:6650",
    "pulsar_topic" = "persistent://tenant/namespace/topic-name",
    "pulsar_subscription" = "load-test",
    "pulsar_partitions" = "load-partition-0,load-partition-1",
    "pulsar_initial_positions" = "POSITION_EARLIEST,POSITION_LATEST",
    "property.auth.token" = "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUJzdWIiOiJqaXV0aWFuY2hlbiJ9.lulGngOC72vE70OW54zcbyw7XdKSOxET94WT_hIqD5Y"
);
```

Routine Load が Pulsar からデータを消費するために作成されると、`data_source_properties` を除くほとんどの入力パラメータは Kafka からデータを消費する場合と同じです。`data_source_properties` を除くパラメータの説明については、[CREATE ROUTINE LOAD](../sql-reference/sql-statements/loading_unloading/routine_load/CREATE_ROUTINE_LOAD.md) を参照してください。

`data_source_properties` に関連するパラメータとその説明は次のとおりです：

| **Parameter**                               | **Required** | **Description**                                              |
| ------------------------------------------- | ------------ | ------------------------------------------------------------ |
| pulsar_service_url                          | Yes          | Pulsar クラスターに接続するために使用される URL。形式: `"pulsar://ip:port"` または `"pulsar://service:port"`。例: `"pulsar_service_url" = "pulsar://``localhost:6650``"` |
| pulsar_topic                                | Yes          | サブスクライブされたトピック。例: "pulsar_topic" = "persistent://tenant/namespace/topic-name" |
| pulsar_subscription                         | Yes          | トピックに設定されたサブスクリプション。例: `"pulsar_subscription" = "my_subscription"` |
| pulsar_partitions, pulsar_initial_positions | No           | `pulsar_partitions` : トピック内のサブスクライブされたパーティション。`pulsar_initial_positions`: `pulsar_partitions` で指定されたパーティションの初期位置。初期位置は `pulsar_partitions` のパーティションに対応している必要があります。有効な値:`POSITION_EARLIEST` (デフォルト値): サブスクリプションはパーティション内の最も早く利用可能なメッセージから開始します。`POSITION_LATEST`: サブスクリプションはパーティション内の最新の利用可能なメッセージから開始します。注意: `pulsar_partitions` が指定されていない場合、トピックのすべてのパーティションがサブスクライブされます。`pulsar_partitions` と `property.pulsar_default_initial_position` の両方が指定されている場合、`pulsar_partitions` の値が `property.pulsar_default_initial_position` の値を上書きします。`pulsar_partitions` も `property.pulsar_default_initial_position` も指定されていない場合、サブスクリプションはパーティション内の最新の利用可能なメッセージから開始します。例:`"pulsar_partitions" = "my-partition-0,my-partition-1,my-partition-2,my-partition-3", "pulsar_initial_positions" = "POSITION_EARLIEST,POSITION_EARLIEST,POSITION_LATEST,POSITION_LATEST"` |

Routine Load は、Pulsar 用の以下のカスタムパラメータをサポートしています。

| Parameter                                | Required | Description                                                  |
| ---------------------------------------- | -------- | ------------------------------------------------------------ |
| property.pulsar_default_initial_position | No       | トピックのパーティションがサブスクライブされたときのデフォルトの初期位置。このパラメータは `pulsar_initial_positions` が指定されていない場合に有効です。その有効な値は `pulsar_initial_positions` の有効な値と同じです。例: `"``property.pulsar_default_initial_position" = "POSITION_EARLIEST"` |
| property.auth.token                      | No       | Pulsar がセキュリティトークンを使用してクライアントを認証する場合、あなたの身元を確認するためにトークン文字列が必要です。例: `"p``roperty.auth.token" = "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUJzdWIiOiJqaXV0aWFuY2hlbiJ9.lulGngOC72vE70OW54zcbyw7XdKSOxET94WT_hIqD"` |

## ロードジョブとタスクを確認する

### ロードジョブを確認する

[SHOW ROUTINE LOAD](../sql-reference/sql-statements/loading_unloading/routine_load/SHOW_ROUTINE_LOAD.md) ステートメントを実行して、ロードジョブ `routine_wiki_edit_1` のステータスを確認します。StarRocks は、実行状態 `State`、統計情報（消費された総行数とロードされた総行数を含む）`Statistics`、およびロードジョブの進行状況 `progress` を返します。

Pulsar からデータを消費する Routine Load ジョブを確認する際、`progress` を除くほとんどの返されるパラメータは Kafka からデータを消費する場合と同じです。`progress` はバックログ、つまりパーティション内の未確認メッセージの数を指します。

```Plaintext
MySQL [load_test] > SHOW ROUTINE LOAD for routine_wiki_edit_1 \G
*************************** 1. row ***************************
                  Id: 10142
                Name: routine_wiki_edit_1
          CreateTime: 2022-06-29 14:52:55
           PauseTime: 2022-06-29 17:33:53
             EndTime: NULL
              DbName: default_cluster:test_pulsar
           TableName: test1
               State: PAUSED
      DataSourceType: PULSAR
      CurrentTaskNum: 0
       JobProperties: {"partitions":"*","rowDelimiter":"'\n'","partial_update":"false","columnToColumnExpr":"*","maxBatchIntervalS":"10","whereExpr":"*","timezone":"Asia/Shanghai","format":"csv","columnSeparator":"','","json_root":"","strict_mode":"false","jsonpaths":"","desireTaskConcurrentNum":"3","maxErrorNum":"10","strip_outer_array":"false","currentTaskConcurrentNum":"0","maxBatchRows":"200000"}
DataSourceProperties: {"serviceUrl":"pulsar://localhost:6650","currentPulsarPartitions":"my-partition-0,my-partition-1","topic":"persistent://tenant/namespace/topic-name","subscription":"load-test"}
    CustomProperties: {"auth.token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUJzdWIiOiJqaXV0aWFuY2hlbiJ9.lulGngOC72vE70OW54zcbyw7XdKSOxET94WT_hIqD"}
           Statistic: {"receivedBytes":5480943882,"errorRows":0,"committedTaskNum":696,"loadedRows":66243440,"loadRowsRate":29000,"abortedTaskNum":0,"totalRows":66243440,"unselectedRows":0,"receivedBytesRate":2400000,"taskExecuteTimeMs":2283166}
            Progress: {"my-partition-0(backlog): 100","my-partition-1(backlog): 0"}
ReasonOfStateChanged: 
        ErrorLogUrls: 
            OtherMsg:
1 row in set (0.00 sec)
```

### ロードタスクを確認する

[SHOW ROUTINE LOAD TASK](../sql-reference/sql-statements/loading_unloading/routine_load/SHOW_ROUTINE_LOAD_TASK.md) ステートメントを実行して、ロードジョブ `routine_wiki_edit_1` のロードタスクを確認します。例えば、いくつのタスクが実行中か、消費されている Kafka トピックのパーティションと消費の進行状況 `DataSourceProperties`、および対応する Coordinator BE ノード `BeId` などです。

```SQL
MySQL [example_db]> SHOW ROUTINE LOAD TASK WHERE JobName = "routine_wiki_edit_1" \G
```

## ロードジョブを変更する

ロードジョブを変更する前に、[PAUSE ROUTINE LOAD](../sql-reference/sql-statements/loading_unloading/routine_load/PAUSE_ROUTINE_LOAD.md) ステートメントを使用して一時停止する必要があります。その後、[ALTER ROUTINE LOAD](../sql-reference/sql-statements/loading_unloading/routine_load/ALTER_ROUTINE_LOAD.md) を実行できます。変更後、[RESUME ROUTINE LOAD](../sql-reference/sql-statements/loading_unloading/routine_load/RESUME_ROUTINE_LOAD.md) ステートメントを実行して再開し、[SHOW ROUTINE LOAD](../sql-reference/sql-statements/loading_unloading/routine_load/SHOW_ROUTINE_LOAD.md) ステートメントを使用してそのステータスを確認できます。

Routine Load が Pulsar からデータを消費するために使用される場合、`data_source_properties` を除くほとんどの返されるパラメータは Kafka からデータを消費する場合と同じです。

**次の点に注意してください**：

- `data_source_properties` に関連するパラメータの中で、現在サポートされているのは `pulsar_partitions`、`pulsar_initial_positions`、およびカスタム Pulsar パラメータ `property.pulsar_default_initial_position` と `property.auth.token` のみです。`pulsar_service_url`、`pulsar_topic`、および `pulsar_subscription` は変更できません。
- 消費するパーティションと一致する初期位置を変更する必要がある場合は、Routine Load ジョブを作成する際に `pulsar_partitions` を使用してパーティションを指定し、指定されたパーティションの初期位置 `pulsar_initial_positions` のみを変更できることを確認する必要があります。
- Routine Load ジョブを作成する際にトピック `pulsar_topic` のみを指定し、パーティション `pulsar_partitions` を指定しない場合、トピックのすべてのパーティションの開始位置を `pulsar_default_initial_position` を介して変更できます。