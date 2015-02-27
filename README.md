# HTTP API 設計指南

## 介紹

這份指南描述一系列 HTTP+JSON 的 API 設計經驗，這些最初來自於 [Heroku 平台 API](https://devcenter.heroku.com/articles/platform-api-reference) 的工作經歷。

這份指南補強了原本的 API，也對 Heroku 新的內部
API 起到了引導的作用。我們希望 Heroku 外面的 API 設計者也會對此感興趣。

我們的目標是在保持一致性與關注業務邏輯的同時，也能避免設計歧異。我們在尋找_一個良好、一致並充分文件化的方法_來設計 API，而不需要是_唯一/理想化的方法_。

本文假設你已經熟悉 HTTP+JSON API 的基礎知識，因此不會在這份指南中涵蓋所有的基礎概念。

歡迎對這份指南做出[貢獻](CONTRIBUTING.md)。

## 內容

* [基礎](#基礎)
  *  [關注點分離](#關注點分離)
  *  [要求安全的連接](#要求安全的連接)
  *  [在 Accept 標頭中指定版本](#在-accept-標頭中指定版本)
  *  [支援 ETag 來快取](#支援-etag-來快取)
  *  [提供 Request-Id 用以追蹤檢討](#提供-request-id-用以追蹤檢討)
  *  [用範圍把大型回應拆分成多個請求](#用範圍把大型回應拆分成多個請求)
* [請求](#請求)
  *  [回傳適合的狀態碼](#回傳適合的狀態碼)
  *  [盡可能提供完整的資源](#盡可能提供完整的資源)
  *  [接受用 JSON 編碼的請求本體](#接受用-json-編碼的請求本體)
  *  [使用一致的路徑格式](#使用一致的路徑格式)
  *  [小寫的路徑和屬性](#小寫的路徑和屬性)
  *  [支援非 id 的取用給予方便](#支援非-id-的取用給予方便)
  *  [最小化路徑巢狀](#最小化路徑巢狀)
* [回應](#回應)
  *  [提供 (UU)ID 給資源](#提供-uuid-給資源)
  *  [提供標準的時間戳記](#提供標準的時間戳記)
  *  [使用 ISO8601 中格式化的 UTC 時間](#使用-iso8601-中格式化的-utc-時間)
  *  [巢狀的外鍵關係](#巢狀的外鍵關係)
  *  [產生結構化的錯誤](#產生結構化的錯誤)
  *  [顯示頻率限制狀態](#顯示頻率限制狀態)
  *  [在所有回應中最小化 JSON](#在所有回應中最小化-json)
* [文件](#文件)
  *  [提供機器可讀的 JSON 綱要](#提供機器可讀的-json-綱要)
  *  [提供人可讀的文件](#提供人可讀的文件)
  *  [提供可執行的範例](#提供可執行的範例)
  *  [描述穩定度](#描述穩定度)

### 基礎

#### 關注點分離

保持事情簡單，同時用請求與回應週期的不同部分之間關注點分離的方式來設計。保持簡單的原則讓我們能專注在更大、更難的問題上面。

產生的請求與回應應該指向特定資源或資源集合。使用路徑來表示資源，使用主體來傳輸內容，使用標頭來溝通元資料 (metadata)。在特殊案例中，查詢參數也可以作為一個傳遞標頭資訊的手段，但還是應該優先使用標頭，因為它更靈活並且可以傳達更多樣化的資訊。

#### 要求安全的連接

要求使用 TLS 建立安全的連接來訪問 API，沒有特例。不值得嘗試找出或解釋什麼時候適合使用 TLS，而什麼時候不適合。讓全部都使用 TLS。

理想情况下，為了避免任何不安全的資料交換，應該簡單地拒絕任何 http 或 80 埠的非 TLS 請求，並不進行回應。在不能這樣做的環境下，則回傳 `403 Forbidden`。

不建議使用重導向，因為它允許馬虎和不好的客服端行為，也沒有任何明確的好處。客戶端依賴重導向會使伺服器的流量倍數增長，且會因為已經在第一次呼叫過程中暴露敏感資料而使 TLS 沒有用。

#### 在 Accept 標頭中指定版本

版本化和版本間的轉換是設計與營運 API 中最具有挑戰性的方面之一。因此，最好一開始就使用一些適當的機制來減少麻煩。

為了避免重大改變會讓使用者的程式發生意外，所有請求最好都要指定版本。應該避免使用預設的版本，因為它在未來會非常難以變更。

最好的方式是在標頭中提供版本規格還有其他的元資料 (metadata)，使用 `Accept` 標頭跟客製化的內容類型，例如：

```
Accept: application/vnd.heroku+json; version=3
```

#### 支援 ETag 來快取

在所有回應中加上 `ETag` 來識別回傳資源的特定版本。這讓使用者可以快取資源，並把這個值放在 `If-None-Match` 標頭中來做出請求，依此決定是否應該更新快取。

#### 提供 Request-Id 用以追蹤檢討

在每個 API 回應中加上 `Request-Id` 標頭，附上一個 UUID 的值。藉由在客戶端、伺服器和任何依賴的服務中記錄這些值，可以提供一個機制來對請求追蹤、診斷和除錯。

#### 用範圍把大型回應拆分成多個請求

大型回應應該拆分成多個請求，使用 `Range` 標頭來指定何時有更多資料可以使用，以及要如何去取得。參閱 [Heroku 平台 API 對 Range 的討論](https://devcenter.heroku.com/articles/platform-api-reference#ranges) 來了解請求與回應標頭、狀態碼、限制次數、排序和迭代的細節。

### 請求

#### 回傳適合的狀態碼

Return appropriate HTTP status codes with each response. Successful
responses should be coded according to this guide:

* `200`: Request succeeded for a `GET` call, for a `DELETE` or
  `PATCH` call that completed synchronously, or for a `PUT` call that
  synchronously updated an existing resource
* `201`: Request succeeded for a `POST` call that completed
  synchronously, or for a `PUT` call that synchronously created a new
  resource
* `202`: Request accepted for a `POST`, `PUT`, `DELETE`, or `PATCH` call that
  will be processed asynchronously
* `206`: Request succeeded on `GET`, but only a partial response
  returned: see [above on ranges](#divide-large-responses-across-requests-with-ranges)

Pay attention to the use of authentication and authorization error codes:

* `401 Unauthorized`: Request failed because user is not authenticated
* `403 Forbidden`: Request failed because user does not have authorization to access a specific resource

Return suitable codes to provide additional information when there are errors:

* `422 Unprocessable Entity`: Your request was understood, but contained invalid parameters
* `429 Too Many Requests`: You have been rate-limited, retry later
* `500 Internal Server Error`: Something went wrong on the server, check status site and/or report the issue

Refer to the [HTTP response code spec](https://tools.ietf.org/html/rfc7231#section-6)
for guidance on status codes for user error and server error cases.

#### 盡可能提供完整的資源

Provide the full resource representation (i.e. the object with all
attributes) whenever possible in the response. Always provide the full
resource on 200 and 201 responses, including `PUT`/`PATCH` and `DELETE`
requests, e.g.:

```bash
$ curl -X DELETE \
  https://service.com/apps/1f9b/domains/0fd4

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
...
{
  "created_at": "2012-01-01T12:00:00Z",
  "hostname": "subdomain.example.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "updated_at": "2012-01-01T12:00:00Z"
}
```

202 responses will not include the full resource representation,
e.g.:

```bash
$ curl -X DELETE \
  https://service.com/apps/1f9b/dynos/05bd

HTTP/1.1 202 Accepted
Content-Type: application/json;charset=utf-8
...
{}
```

#### 接受用 JSON 編碼的請求本體

Accept serialized JSON on `PUT`/`PATCH`/`POST` request bodies, either
instead of or in addition to form-encoded data. This creates symmetry
with JSON-serialized response bodies, e.g.:

```bash
$ curl -X POST https://service.com/apps \
    -H "Content-Type: application/json" \
    -d '{"name": "demoapp"}'

{
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "name": "demoapp",
  "owner": {
    "email": "username@example.com",
    "id": "01234567-89ab-cdef-0123-456789abcdef"
  },
  ...
}
```

#### 使用一致的路徑格式

##### Resource names

Use the plural version of a resource name unless the resource in question is a singleton within the system (for example, in most systems a given user would only ever have one account). This keeps it consistent in the way you refer to particular resources.

##### Actions

Prefer endpoint layouts that don’t need any special actions for
individual resources. In cases where special actions are needed, place
them under a standard `actions` prefix, to clearly delineate them:

```
/resources/:resource/actions/:action
```

e.g.

```
/runs/{run_id}/actions/stop
```

#### 小寫的路徑和屬性

Use downcased and dash-separated path names, for alignment with
hostnames, e.g:

```
service-api.com/users
service-api.com/app-setups
```

Downcase attributes as well, but use underscore separators so that
attribute names can be typed without quotes in JavaScript, e.g.:

```
service_class: "first"
```

#### 支援非 id 的取用給予方便

In some cases it may be inconvenient for end-users to provide IDs to
identify a resource. For example, a user may think in terms of a Heroku
app name, but that app may be identified by a UUID. In these cases you
may want to accept both an id or name, e.g.:

```bash
$ curl https://service.com/apps/{app_id_or_name}
$ curl https://service.com/apps/97addcf0-c182
$ curl https://service.com/apps/www-prod
```

Do not accept only names to the exclusion of IDs.

#### 最小化路徑巢狀

In data models with nested parent/child resource relationships, paths
may become deeply nested, e.g.:

```
/orgs/{org_id}/apps/{app_id}/dynos/{dyno_id}
```

Limit nesting depth by preferring to locate resources at the root
path. Use nesting to indicate scoped collections. For example, for the
case above where a dyno belongs to an app belongs to an org:

```
/orgs/{org_id}
/orgs/{org_id}/apps
/apps/{app_id}
/apps/{app_id}/dynos
/dynos/{dyno_id}
```

### 回應

#### 提供 (UU)ID 給資源

Give each resource an `id` attribute by default. Use UUIDs unless you
have a very good reason not to. Don’t use IDs that won’t be globally
unique across instances of the service or other resources in the
service, especially auto-incrementing IDs.

Render UUIDs in downcased `8-4-4-4-12` format, e.g.:

```
"id": "01234567-89ab-cdef-0123-456789abcdef"
```

#### 提供標準的時間戳記

Provide `created_at` and `updated_at` timestamps for resources by default,
e.g:

```javascript
{
  // ...
  "created_at": "2012-01-01T12:00:00Z",
  "updated_at": "2012-01-01T13:00:00Z",
  // ...
}
```

These timestamps may not make sense for some resources, in which case
they can be omitted.

#### 使用 ISO8601 中格式化的 UTC 時間

Accept and return times in UTC only. Render times in ISO8601 format,
e.g.:

```
"finished_at": "2012-01-01T12:00:00Z"
```

#### 巢狀的外鍵關係

Serialize foreign key references with a nested object, e.g.:

```javascript
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0..."
  },
  // ...
}
```

Instead of e.g.:

```javascript
{
  "name": "service-production",
  "owner_id": "5d8201b0...",
  // ...
}
```

This approach makes it possible to inline more information about the
related resource without having to change the structure of the response
or introduce more top-level response fields, e.g.:

```javascript
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0...",
    "name": "Alice",
    "email": "alice@heroku.com"
  },
  // ...
}
```

#### 產生結構化的錯誤

Generate consistent, structured response bodies on errors. Include a
machine-readable error `id`, a human-readable error `message`, and
optionally a `url` pointing the client to further information about the
error and how to resolve it, e.g.:

```
HTTP/1.1 429 Too Many Requests
```

```json
{
  "id":      "rate_limit",
  "message": "Account reached its API rate limit.",
  "url":     "https://docs.service.com/rate-limits"
}
```

Document your error format and the possible error `id`s that clients may
encounter.

#### 顯示頻率限制狀態

Rate limit requests from clients to protect the health of the service
and maintain high service quality for other clients. You can use a
[token bucket algorithm](http://en.wikipedia.org/wiki/Token_bucket) to
quantify request limits.

Return the remaining number of request tokens with each request in the
`RateLimit-Remaining` response header.

#### 在所有回應中最小化 JSON

Extra whitespace adds needless response size to requests, and many
clients for human consumption will automatically "prettify" JSON
output. It is best to keep JSON responses minified e.g.:

```json
{"beta":false,"email":"alice@heroku.com","id":"01234567-89ab-cdef-0123-456789abcdef","last_login":"2012-01-01T12:00:00Z","created_at":"2012-01-01T12:00:00Z","updated_at":"2012-01-01T12:00:00Z"}
```

Instead of e.g.:

```json
{
  "beta": false,
  "email": "alice@heroku.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "last_login": "2012-01-01T12:00:00Z",
  "created_at": "2012-01-01T12:00:00Z",
  "updated_at": "2012-01-01T12:00:00Z"
}
```

You may consider optionally providing a way for clients to retreive
more verbose response, either via a query parameter (e.g. `?pretty=true`)
or via an `Accept` header param (e.g.
`Accept: application/vnd.heroku+json; version=3; indent=4;`).

### 文件

#### 提供機器可讀的 JSON 綱要

Provide a machine-readable schema to exactly specify your API. Use
[prmd](https://github.com/interagent/prmd) to manage your schema, and ensure
it validates with `prmd verify`.

#### 提供人可讀的文件

Provide human-readable documentation that client developers can use to
understand your API.

If you create a schema with prmd as described above, you can easily
generate Markdown docs for all endpoints with with `prmd doc`.

In addition to endpoint details, provide an API overview with
information about:

* Authentication, including acquiring and using authentication tokens.
* API stability and versioning, including how to select the desired API
  version.
* Common request and response headers.
* Error serialization format.
* Examples of using the API with clients in different languages.

#### 提供可執行的範例

Provide executable examples that users can type directly into their
terminals to see working API calls. To the greatest extent possible,
these examples should be usable verbatim, to minimize the amount of
work a user needs to do to try the API, e.g.:

```bash
$ export TOKEN=... # acquire from dashboard
$ curl -is https://$TOKEN@service.com/users
```

If you use [prmd](https://github.com/interagent/prmd) to generate Markdown
docs, you will get examples for each endpoint for free.

#### 描述穩定度

Describe the stability of your API or its various endpoints according to
its maturity and stability, e.g. with prototype/development/production
flags.

See the [Heroku API compatibility policy](https://devcenter.heroku.com/articles/api-compatibility-policy)
for a possible stability and change management approach.

Once your API is declared production-ready and stable, do not make
backwards incompatible changes within that API version. If you need to
make backwards-incompatible changes, create a new API with an
incremented version number.

