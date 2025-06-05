# Couchbase LiteによるiOS端末間のデータ同期テスト

[オリジナル](https://github.com/waynecarter/simple-sync)を少し修正し2025年3月13日の [モバチキ 〜Mobile Tips 共有会〜 #7](https://lycorptech-jp.connpass.com/event/345858/) の発表でデモ動画を流したアプリです

発表タイトル「オフラインファーストで自動同期もするモバイルDBあれこれ」

修正点は以下のとおりです (詳細は https://github.com/ec22s/couchbase-simple-sync/pull/1 を参照)

- iOS 15.6以上で動くようにした (オリジナルは17.0以上)
- SPMを使い、パッケージの手動インストールを不要にした
- `Couchbase Lite Vector Search` を使う箇所がエラーで動かないため一時的に回避

<br>

当日のスライドとデモ動画です
- [スライド (PDF)](https://ec22s.github.io/couchbase-simple-sync/20250313_Fukuoka.pdf)
  - 誤記などを修正しています
  - 文字を濃くしました (2025/3/19)

- デモ動画その1

  https://github.com/user-attachments/assets/ccf01161-7a74-4b6b-a253-035403815ae4
  - スライドp.17-18にある、当日流したものです
  - この時点ではオリジナルとほぼ同じソースでビルドしました
  - 画面左側に映るルータの設定で、通信がインターネットへ出ない状態にしています
  - マップでもインターネットと通信できないことを確認しています (キャッシュのみ表示)
  - このLAN内だけの通信で、シミュレータ上のアプリ (画面左) と実機のiPad (右) で、アプリ内の「円の色」が同期します

- デモ動画その2    

  https://github.com/user-attachments/assets/5c8645bc-11be-4462-988b-8516f2e0bfde
  - 本リポジトリをビルドし動作を確認しました
  - 最初に映る設定画面でクラウドとの同期設定がない、つまり同期するなら端末同士の通信になることが確認できます
  - アプリを開き、実機のiPhone7 (画面左、iOS15) とシミュレータ上のアプリ (右) がいろいろ同期します

<br>

- 追記 (2025.6.6)

  上のデモを再度試そうとビルドし直したら、端末間でデータ同期されなくなっていました. Issue https://github.com/ec22s/couchbase-simple-sync/issues/7 で調査中です


<br>

以下、オリジナルの README です. パッケージの手動インストール ([Run the Project](#run-the-project) 2.) は、本ソースを使う場合不要です

---

# Simple Sync

Simple Sync is a demonstration of how to read, write, search, and sync data using [Couchbase Lite](https://docs.couchbase.com/couchbase-lite/current/). This repository provides a comprehensive guide to handling different types of data and demonstrates how to synchronize this data across devices, with and without the Internet.

The [Simple Sync](https://apps.apple.com/us/app/simple-color-sync/id6449199482) app is available for download from the App Store. You can download and run the app on your device without any additional setup to see these features in action.

[<img alt="Download on the App Store" src="images/download.svg" width="120" height="40" />](https://apps.apple.com/us/app/simple-color-sync/id6449199482)

## Introduction

The code is divided into four major areas, each demonstrating different aspects of data handling and synchronization:

### Color Sync

The `ColorViewController` class manages the color sync feature. It demonstrates how to read, write, and sync simple scalar data, and listen for database changes.

```swift
// Get the profile document from the database collection.
let profile = collection["profile1"].document
// Get the color from the profile.
let color = profile["color"].string

// Update the color.
profile["color"].string = "green"

// Save the document.
try collection.save(document: profile)
```

#### Listen for Changes

```swift
// Listen for changes to the profile document.
collection.addDocumentChangeListener(id: "profile1") { change in
    // React to the change.
}
```

### Photo Sync

The `PhotoViewController` class manages the photo sync feature. It demonstrates how to read, write, and sync binary data.

```swift
// Get the profile document from the database collection.
let profile = collection["profile1"].document
// Get the photo from the profile.
let photo = profile["photo"].blob

// Update the photo.
let newPhoto = UIImage(named: "newPhoto")
if let pngData = newPhoto.pngData() {
    profile["photo"].blob = Blob(contentType: "image/png", data: pngData)
}
    
// Save the document.
try collection.save(document: profile)
```

### Count Sync

The `CountViewController` class manages the count sync feature. It demonstrates how to read, write, and sync complex data using a CRDT [PN-Counter (Positive-Negative Counter)](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type#PN-Counter_(Positive-Negative_Counter)).

```swift
// Get the item document from the database collection.
let item = collection["item1"].document
// Get the counter from the item.
var count = item.counter(forKey: "count")

// Increment the count.
count.increment(by: 1)
// Decrement the count.
count.decrement(by: 1)

// Save the document with concurrency control.
try collection.save(document: item, concurrencyControl: .failOnConflict)
```

### Search

The `SearchViewController` class manages the search feature. It demonstrates how to use Couchbase Lite’s SQL, full-text search, vector search, and indexing capabilities.

#### SQL

```sql
SELECT name, image
FROM products
WHERE category = $category
  AND VECTOR_MATCH(ImageVectorIndex, $embedding, 10)
  AND VECTOR_DISTANCE(ImageVectorIndex) < 0.35
  AND MATCH(NameColorAndCategoryIndex, $search)
ORDER BY VECTOR_DISTANCE(ImageVectorIndex), RANK(NameColorAndCategoryIndex), name
```

#### Query

```swift
// Create the query.
let query = try database.createQuery(sql)

// Set the query parameters.
query.parameters = Parameters()
    .setString(category, forName: "category")
    .setString(embedding, forName: "embedding")
    .setString(search, forName: "search")

// Execute the query.
let results = try query.execute()
```

#### Indexing

```swift
// For fast sorting, create an index on the "name" field.
let nameIndex = ValueIndexConfiguration(["name"])
try collection.createIndex(withName: "NameIndex", config: nameIndex)

// For fast predicates, create an index on the "category" field.
let categoryIndex = ValueIndexConfiguration(["category"])
try collection.createIndex(withName: "CategoryIndex", config: categoryIndex)
        
// Initialize the vector index on the "embedding" field for image search.
var vectorIndex = VectorIndexConfiguration(expression: "embedding", dimensions: 768, centroids: 1000)
try collection.createIndex(withName: "ImageVectorIndex", config: vectorIndex)

// For fast searches, create a full-text search index on the "name", "color", and "category" fields.
let ftsIndex = FullTextIndexConfiguration(["name", "color", "category"])
try collection.createIndex(withName: "NameColorAndCategoryIndex", config: ftsIndex)
```

## App

The `App` class manages the overall synchronization of the application using Couchbase Lite's sync capabilities. It demonstrates how to sync peer-to-peer and with an endpoint over the internet. The class handles network management, monitoring, and trust verification. 

```swift
// Create the app passing the database to sync, the sync endpoint, and the identity
// and certificate authority for peer-to-peer trust verification.
let app = App(
    database: database,
    endpoint: endpoint,
    identity: identity,
    ca: ca
)

// Start syncing.
app.start()
```

**NOTE:** The included `gen-credentials.sh` script was used to generate the credentials included with the project. If you want to generate new credentials, run that script again and replace the files in the project's credentials folder with the newly generated files.

## Configure an Endpoint

An endpoint can be hosted using [Couchbase Capella](https://cloud.couchbase.com) or Couchbase Sync Gateway, and connected in the Simple Sync app settings.

### Capella Setup

Start with an existing App Service or create a new one. In the App Service, create and configure the following endpoints:

#### Color

1. Create an App Endpoint named “color”
2. In the App Endpoint, create a user with the Admin Channel “color”
3. Define the Access Control and Data Validation function as:
```javascript
function(doc, oldDoc, meta) {
    if (doc._id !== "profile") {
        throw new Error();
    }
    channel("color");
}
```

#### Photo

1. Create an App Endpoint named “photo”
2. In the App Endpoint, create a user with the Admin Channel “photo”. The user should have the same username/password as the user created for the "color" App Endpoint.
3. Define the Access Control and Data Validation function as:
```javascript
function(doc, oldDoc, meta) {
    if (doc._id !== "profile") {
        throw new Error();
    }
    channel("photo");
}
```

#### Count

1. Create an App Endpoint named “count”
2. In the App Endpoint, create a user with the Admin Channel “count”. The user should have the same username/password as the user created for the "color" App Endpoint.
3. Define the Access Control and Data Validation function as:
```javascript
function(doc, oldDoc, meta) {
    if (doc._id !== "item") {
        throw new Error();
    }
    channel("count");
}
```

## Run the Project

1. Clone or download this repository
2. [Download](https://www.couchbase.com/downloads/?family=couchbase-lite) the latest `CouchbaseLiteSwift.xcframework` and `CouchbaseLiteVectorSearch.xcframework`, and copy them to the project's `Frameworks` directory.
3. Open the project in Xcode.
4. Run the app on two or more simulators, phones, or tablets.
   1. Tap on the `Color` view, then tap the screen. The color will change and sync with other devices.
   2. Tap on the `Photo` view, then tap the screen. The photo will change and sync with other devices.
   3. Tap on the `Count` view, then tap the buttons. The count will change and sync with other devices.
   3. Tap on the `Search` view, then search using name, color, category, image, and more.

<img alt="Watch the Demo Video" src="images/demo-screenshot.png" />

### Source Files

To explore the code, start with the following source files:

* `ColorViewController.swift`: Manages the color sync feature.
* `PhotoViewController.swift`: Manages the photo sync feature.
* `CountViewController.swift`: Manages the count sync feature.
* `SearchViewController.swift`: Manages the search feature.
* `Counter.swift`: Contains the `Counter` and `MutableCounter` classes for managing the CRDT pn-counter, and the `CRDTConflictResolver` class for resolving conflicts.
* `App.swift`: Manages the peer-to-peer and internet endpoint sync features, and handles network monitoring and trust verification.
