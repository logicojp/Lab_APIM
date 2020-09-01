# API Management - Hands-on Lab Script - part 4


- [Part 1 - API Managementインスタンスの作成](apimanagement-1.md)
- [Part 2 - 開発者ポータルと製品(Product)の管理](apimanagement-2.md)
- [Part 3 - APIの追加](apimanagement-3.md)
- Part 4 - キャッシュとポリシー
- [Part 5 - バージョンとリビジョン](apimanagement-5.md)
- [Part 6 - 分析と監視](apimanagement-6.md)
- [Part 7 - セキュリティ](apimanagement-7.md)
- [Part 8 - DevOps](apimanagement-8.md)

#### キャッシュ

Azure API Managementでは、レスポンスをキャッシュするよう設定できます。これにより、頻繁に変更されないデータのAPI待ち時間、帯域幅の消費、およびWebサービスの負荷を大幅に削減できます。

- Azure管理ポータルを使い、キャッシュポリシーをRandomColor APIに対して設定します。
  - キャッシュに保持する期間は15秒
  - Azure管理ポータルではシンプルなキャッシュの構成が未実装です。ポリシー式を使って実装する方法を後ほどご紹介します。

![](Images/APIMEnableCaching.png)

![](Images/APIMEnableCaching2.png)

![](Images/APIMEnableCaching3.png)

- Color WebsiteをUnlimited URLを使うように構成
- [Start] を選択
- 各15秒間で同じ色が設定されていることに注意してください。

![](Images/APIMColorWebCaching.png)

## ポリシー式

ポリシー式を使ってトラフィックを制御したりバックエンドAPIの振る舞いを変更したりします。C#のステートメントとAPIコンテキストやAPI Managementサービス公正へのアクセス機能を使って実行時にAPIの振る舞いを変更することができるため、ポリシー式は強力な手段です。

<https://docs.microsoft.com/azure/api-management/api-management-policies>

先ほど、CORS ポリシーとキャッシングの設定について簡単に説明しました。もう少し深く掘り下げてみましょう。

### 構成

- APIを選択（例：Color API）
- フロントエンド、インバウンド処理、アウトバウンド処理、バックエンドの設定ができることに着目してください。
  - 鉛筆アイコンを選択するだけで編集できます
- また、設定をAPIや個々のオペレーションに範囲を絞ることができる点にも着目してください。

![](Images/APIMPolicyEditor.png)

- フロントエンドを編集する
  - Operationを編集する場合、'Code View' エディターもしくはフォームベースのエディターを選択できます。
  - If editing an APIを編集する場合、'Code View' エディターのみが選択できます。
  - 'Code View' エディターではSwagger (OpenAPI) 定義の修正が可能です。

![](Images/APIMFrontendCodeEditor.png)

![](Images/APIMFrontendFormEditor.png)

- インバウンド処理 / アウトバウンド処理 / バックエンド
  - 'Code View'エディターもしくは [Add Policy] フォームを選択できます。

![](Images/APIMInboundProcessing.png)

![](Images/APIMInboundCodeEditor.png)

![](Images/APIMInboundFormEditor.png)

### 例

#### HTTPレスポンスのキャッシュ

- RandomColor API
- 'Code View' に切り替え
  - （設定済みの）キャッシングポリシーを参照してください。

```xml
<!-- Inbound -->
  <cache-lookup vary-by-developer="false"
      vary-by-developer-groups="false"
      downstream-caching-type="none" />

<!-- Outbound -->
<cache-store duration="15" />
```

#### XMLからJSONへの変換

よくある要件としてコンテンツの変換があります。

- Calc API を覚えていますか... これは XML を返しました。
- Calculator APIの'Code View'を開いて
- レスポンスボディをJSONに変換するためのアウトバウンドポリシーを追加します
- APIを呼び出しレスポンスをチェックし、レスポンスがJSONになっていることを確認しましょう。

```xml
<!-- Outbound -->
<xml-to-json kind="direct" apply="always" consider-accept-header="false" />
```

![](Images/APIMResponseXMLtoJSON.png)

#### プロパティ・コレクション

プロパティは、サービス インスタンスに対してグローバルなKey-Vauleコレクションです。これらのプロパティを使って全てのAPI構成やポリシー間にわたる固定文字列を管理できます。値は式やシークレット（表示されません）を取り得ます。

- `TimeNow`というプロパティを設定
  - 例：`@(DateTime.Now.ToString())`

![](Images/APIMNamedValues.png)

- Calculator APIの'Code View'を開く
- インバウンドポリシーをHeaderに追加
- Azure管理ポータル内でAPIをテスト
  - [Ocp-Apim-Trace]というヘッダーを追加し、値としてtrueを設定
  - レスポンスや[Trace]タブをチェック

```xml
<!-- Inbound -->
<set-header name="timeheader" exists-action="override">
    <value>{{TimeNow}}</value>
</set-header>
```

![](Images/APIMTraceNV.png)

![](Images/APIMTraceNV2.png)

- HTTPレスポンスで指定されたURL [ocp-apim-trace-location] に移動します。
  - [timeheader] フィールドがバックエンドAPIに送信されていることに注目してください。

![](Images/APIMTraceNV3.png)

#### レスポンス・ヘッダーの削除

ヘッダーの削除もよくある要件です。例えば、潜在的なセキュリティ情報が漏洩する可能性のあるヘッダの削除などです。

- Calculator APIの'Code View'を開く
- レスポンス・ヘッダーを削除するアウトバウンド・ポリシーを追加する
- APIを呼び出しレスポンスを確認する

```xml
<!-- Outbound -->
<set-header name="x-aspnet-version" exists-action="delete" />
<set-header name="x-powered-by" exists-action="delete" />
```

ポリシー設定前:
![](Images/APIMResponseDeleteHeaders.png)

ポリシー設定後:
![](Images/APIMResponseDeleteHeaders2.png)

#### バックエンドに渡された内容を変更する

ポリシー式にはC#のコードを含めることができｌ数多くの.NET Frameworkの型やそのメンバーにアクセスできます。`context`という名前の変数を暗黙のうちに利用できます。そのメンバーはAPIリクエストに関連する情報を提供します。

詳細は <https://docs.microsoft.com/en-us/azure/api-management/api-management-policy-expressions> を参照ください。

- Calculator APIの'Code View'を開く
- クエリ文字列とヘッダーを変更するインバウンド・ポリシーを追加する
- APIを呼び出し、Trace関数を使って何がバックエンドに渡されたかを確認する

```xml
<!-- Inbound -->
<set-query-parameter name="x-product-name" exists-action="override">
    <value>@(context.Product.Name)</value>
</set-query-parameter>
<set-header name="x-request-context-data" exists-action="override">
    <value>@(context.User.Id)</value>
    <value>@(context.Deployment.Region)</value>
</set-header>
```

注意：以下のトレースは開発者ポータルから取得したものです。Azure管理ポータルからテストした場合には[User Id]を評価できないためにエラーが発生しました。

![](Images/APIMTraceAmendBackend.png)

#### 条件に依存した変換

製品によってレスポンス・ボディを操作する別のC#の例は、この式を使ってStarter製品のサブスクライバーの場合は情報のサブセットのみを取得し、他の製品の場合は、完全な情報を取得します。

- Star Wars APIのGetPeopleById オペレーションで 'Code View' エディターを開く
- 条件付きでレスポンス・ボディを変更するアウトバウンド・ポリシーを追加する
- Starter製品のキーを使ってAPIを呼び出し、レスポンスを確認する
- Unlimited製品のキーを使ってAPIを呼び出し、レスポンスを確認する

注意：レスポンス・ボディがエンコードされずJSONのパースに失敗しないよう、インバウンドヘッダを設定してください。

```xml
<!-- Inbound -->
        <set-header name="Accept-Encoding" exists-action="override">
            <value>deflate</value>
        </set-header>
<!-- Outbound -->
<choose>
    <when condition="@(context.Response.StatusCode == 200 && context.Product.Name.Equals("Starter"))">
        <set-body>@{
                var response = context.Response.Body.As<JObject>();
                foreach (var key in new [] {"hair_color", "skin_color", "eye_color", "gender"}) {
                    response.Property(key).Remove();
                }
                return response.ToString();
            }
        </set-body>
    </when>
</choose>

```

Starterの場合

![](Images/APIMResponseCondStarter.png)

Unlimitedの場合

![](Images/APIMResponseCondUnlimited.png)


#### Aborting the processing

- Open the Calculator API 'Code View'
- Add the inbound policy to test for a condition and return error
- Invoke the API - with Authorization header as above ... should get a 599 error
- Replace the condition with some more meaningful code

```xml
<!-- Inbound -->
<choose>
    <when condition="@(true)">
        <return-response response-variable-name="existing response variable">
            <set-status code="599" reason="failure" />
            <set-header name="failure" exists-action="override">
                <value>failure</value>
            </set-header>
            <set-body>blah</set-body>
        </return-response>
    </when>
</choose>
```

![](Images/APIMResponseAbort.png)

#### Send message to Microsoft Teams channel

Alternatively use Slack

This example shows 'Send one way request' ... sends a request to the specified URL without waiting for a response.  Another option is to Send request and Wait.  Complex in flight processing logic is better handled by using Logic Apps.

For Microsoft Teams

- First need to go into Teams and enable a Web hook connector.
  - Get the URL of the webhook.

![](Images/APIMTeamsWebHook1.png)

![](Images/APIMTeamsWebHook2.png)

![](Images/APIMTeamsWebHook3.png)

![](Images/APIMTeamsWebHook4.png)

- Format the required payload ... the payload sent to a Teams channel is of the MessageCard JSON schema format
  - <https://docs.microsoft.com/en-us/microsoftteams/platform/task-modules-and-cards/cards/cards-reference>
  - <https://messagecardplayground.azurewebsites.net/>

- Open the Calculator API 'Code View'
- Add the outbound policy
  - replace the webhook and payload as required

```xml
<!-- Outbound -->
<choose>
    <when condition="@(context.Response.StatusCode >= 299)">
        <send-one-way-request mode="new">
            <set-url>https://outlook.office.com/webhook/78f54a63-f217-451a-b263-f1f5c0e866f0@72f988bf-86f1-41af-91ab-2d7cd011db47/IncomingWebh00k/34228a8ccbe94e368d3ac4782adda9b2/4e01c743-d419-49b7-88c6-245e5e31664a</set-url>
            <set-method>POST</set-method>
            <set-body>@{
                    return new JObject(
                        new JProperty("@type","MessageCard"),
                        new JProperty("@context", "http://schema.org/extensions"),
                        new JProperty("summary","Summary"),
                        new JProperty("themeColor", "0075FF"),
                        new JProperty("sections",
                            new JArray (
                                new JObject (
                                    new JProperty("text","Error - details: [link](http://azure1.org)")
                                    )
                                )
                            )
                        ).ToString();
            }</set-body>
        </send-one-way-request>
    </when>
</choose>

```

- for demo purposes, amend the condition so it always fires i.e. `StatusCode = 200`
- Invoke the API ... should get a 200 success
- Look for a message in the Teams channel

```xml
    <when condition="@(context.Response.StatusCode == 200)">
```

Received notification in the Teams channel:

![](Images/APIMTeamsMessage.png)

#### Send to Azure Event Hub

Using the log-to-eventhub policy enables sending any details from the request and response to an Azure Event Hub. Usage examples include audit trail of updates, usage analytics, logging, monitoring, billing, exception alerting and 3rd party integrations.

The Azure Event Hubs is designed to ingress huge volumes of data, with capacity for dealing with a far higher number of events. This ensures that your API performance will not suffer due to the logging infrastructure.

<https://docs.microsoft.com/en-us/azure/api-management/api-management-log-to-eventhub-sample>

#### Cross-origin resource sharing (CORS)

The cors policy adds cross-origin resource sharing (CORS) support to an operation or an API to allow cross-domain calls from browser-based clients.

<https://docs.microsoft.com/en-us/azure/api-management/api-management-cross-domain-policies#CORS>

```xml
<!-- Inbound -->
    <cors>
        <allowed-origins>
            <origin>*</origin>
        </allowed-origins>
        <allowed-methods>
            <method>GET</method>
            <method>POST</method>
            <method>PUT</method>
            <method>DELETE</method>
            <method>HEAD</method>
            <method>OPTIONS</method>
            <method>PATCH</method>
            <method>TRACE</method>
        </allowed-methods>
        <allowed-headers>
            <header>*</header>
        </allowed-headers>
        <expose-headers>
            <header>*</header>
        </expose-headers>
    </cors>
```

#### Mock responses

Mocking provides a way to return sample responses even when the backend is not available. This enables app developers to not be help up if the backend is under development.

- Open the Star Wars API and select [Add Operation]
- Create a new operation called GetFilm
- In the Response configuration tab, set Sample data as below

![](Images/APIMMockingFrontend.png)

![](Images/APIMMockingFrontend2.png)

![](Images/APIMMockingFrontend3.png)

```json
{
  "count": 1,
  "films": [   { "title": "A New Hope",  "blah": "xxx"    }   ]
}
```

- Open the Inbound processing 'Code View'
- Enable mocking and specify a 200 OK response status code

![](Images/APIMMockingInbound.png)

- Select the 200 OK response ... Save

![](Images/APIMMockingInbound2.png)

- Mocking is now enabled

![](Images/APIMMockingInbound3.png)

- Invoke the API ... should get a 200 success with the mocked data

![](Images/APIMMockingResponse.png)

---
[Home](README.md) | [Prev](apimanagement-3.md) | [Next](apimanagement-5.md)

