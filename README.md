[![Mihael Isaev](https://user-images.githubusercontent.com/1272610/40946272-af6396fa-686d-11e8-82af-192850fe3216.png)](http://mihaelisaev.com)

<p align="center">
    <a href="https://discord.gg/vapor">
        <img src="https://img.shields.io/discord/431917998102675485.svg" alt="Vapor in Discord">
    </a>
    <a href="LICENSE">
        <img src="https://img.shields.io/badge/license-MIT-brightgreen.svg" alt="MIT License">
    </a>
    <a href="https://swift.org">
        <img src="https://img.shields.io/badge/swift-4.1-brightgreen.svg" alt="Swift 4.1">
    </a>
    <a href="https://twitter.com/VaporRussia">
        <img src="https://img.shields.io/badge/twitter-VaporRussia-5AA9E7.svg" alt="Twitter">
    </a>
</p>

<br>

# Quick Intro

```swift
struct PublicUser: Codable {
    var name: String
    var petName: String
    var petType: String
    var petToysQuantity: Int
}
try FluentQuery()
    .select(all: User.self)
    .select(\Pet.name, as: "petName")
    .select(\PetType.name, as: "petType")
    .select(count: \PetToy.id, as: "")
    .from(User.self)
    .join(.left, Pet.self, where: FQWhere(\Pet.id == \User.idPet))
    .join(.left, PetType.self, where: FQWhere(\PetType.id == \Pet.idType))
    .join(.left, PetToy.self, where: FQWhere(\PetToy.idPet == \Pet.id))
    .groupBy(FQGroupBy(\User.id).and(\Pet.id).and(\PetType.id).and(\PetToy.id))
    .execute(on: conn)
    .decode(PublicUser.self) // -> Future<[PublicUser]> 🔥🔥🔥
```

# Intro

It's a swift lib that gives ability to build complex raw SQL-queries in a more easy way using KeyPaths.

Built for Vapor3 and depends on `Fluent` package because it uses `Model.reflectProperty(forKey:)` method to decode KeyPaths.

For now I developing it for Postgres queries so it is with Postgres's SQL-syntax only.

Now it supports: query with most common predicates, building json objects in select, subqueries, subquery into json, joins, aggregate functions, etc.

Note: the project is in active development state and it may cause huge syntax changes before v1.0.0

If you have great ideas of how to improve this package write me (@iMike) in Vapor's discord chat or just send pull request.

Hope it'll be useful for someone :)

### Install through Swift Package Manager

Edit your `Package.swift`

```swift
//add this repo to dependencies
.package(url: "https://github.com/MihaelIsaev/FluentQuery.git", from: "0.2.0")
//and don't forget about targets
//"FluentQuery"
```
### One more little intro

I love to write raw SQL queries because it gives ability to flexibly use all the power of database engine.

And Vapor's Fleunt allows you to do raw queries, but the biggest problem of raw queries is its hard to maintain them.

I faced with that problem and I started developing this lib to write raw SQL queries in swift-way by using KeyPaths.

And let's take a look what we have :)

### How it works

First of all you need to import the lib

```swift
import FluentQuery
```

Then create `FluentQuery` object to do some building and get raw query string

```swift
let query = FluentQuery()
//some building
let rawQuery: String = query.build()
```

Let's take a look how to use it with some example request

Imagine that you have a list of cars

So you have `Car` fluent model

```swift
final class Car: Model {
  var id: UUID?
  var year: String
  var color: String
  var engineCapacity: Double
  var idBrand: UUID
  var idModel: UUID
  var idBodyType: UUID
  var idEngineType: UUID
  var idGearboxType: UUID
}
```
and related models
```swift
final class Brand: Model {
  var id: UUID?
  var value: String
}
final class Model: Model {
  var id: UUID?
  var value: String
}
final class BodyType: Model {
  var id: UUID?
  var value: String
}
final class EngineType: Model {
  var id: UUID?
  var value: String
}
final class GearboxType: Model {
  var id: UUID?
  var value: String
}
```

ok, and you want to get every car as convenient codable model

```swift
struct PublicCar: Content {
  var id: UUID
  var year: String
  var color: String
  var engineCapacity: Double
  var brand: Brand
  var model: Model
  var bodyType: BodyType
  var engineType: EngineType
  var gearboxType: GearboxType
}
```

Here's example request code for that situation

```swift
func getListOfCars(_ req: Request) throws -> Future<[PublicCar]> {
  let query = FluentQuery()
    .select(distinct: \Car.id)
    .select(as: "car", FQJSON(.binary)
      .field("id", \Car.id)
      .field("year", \Car.year)
      .field("color", \Car.color)
      .field("engineCapacity", \Car.engineCapacity)
      .field("brand", func: .rowToJson(Brand.self))
      .field("model", func: .rowToJson(Model.self))
      .field("bodyType", func: .rowToJson(BodyType.self))
      .field("engineType", func: .rowToJson(EngineType.self))
      .field("gearboxType", func: .rowToJson(GearboxType.self))
    )    
    .from(Car.self)
    .join(.left, Brand.self, where: FQWhere(\Brand.id == \Car.idBrand))
    .join(.left, Model.self, where: FQWhere(\Model.id == \Car.idModel))
    .join(.left, BodyType.self, where: FQWhere(\BodyType.id == \Car.idBodyType))
    .join(.left, EngineType.self, where: FQWhere(\EngineType.id == \Car.idEngineType))
    .join(.left, GearboxType.self, where: FQWhere(\GearboxType.id == \Car.idGearboxType))
    .groupBy(FQGroupBy(\Car.id)
      .and(\Brand.id)
      .and(\Model.id)
      .and(\BodyType.id)
      .and(\EngineType.id)
      .and(\GearboxType.id)
    )
    .orderBy(FQOrderBy(\Brand.value, .ascending)
      .and(\Model.value, .ascending)
    )
  let rawQuery: String = query.build()
  return req.requestPooledConnection(to: .psql).flatMap { conn -> EventLoopFuture<[PublicCar]> in
    return conn.query(rawQuery).map { queryResult -> [PublicCar] in
      defer { try? req.releasePooledConnection(conn, to: .psql) }
      return try queryResult.map {
        guard let car = $0.firstValue(forColumn: "car")?.data else {
          throw Abort(.internalServerError, reason: "Can't get car")
        }
        return try JSONDecoder().decode(PublicCar.self, from: car[1...])
      }
    }
  }
}
```

As you can see we've build complex query to get all depended values and decoded postgres raw response to our codable model.

We used `FQJSON` (`jsonb_build_object` equivalent) to build the struct as we have in our codable model to conveniently decode it.

In the future when Fluent will make public function for easy decoding for whole raw postgres response we will be able to simplify this request, but for now we have what we have.

Hahah, but that's cool right? 😃

<details>
    <summary>BTW, this is a raw SQL equivalent</summary>
        
    SELECT
    DISTINCT c.id,
    jsonb_build_object(
      'id', c.id,
      'year', c.year,
      'color', c.color,
      'engineCapacity', c."engineCapacity",
      'brand', (SELECT row_to_json(brand)),
      'model', (SELECT row_to_json("Models")),
      'bodyType', (SELECT row_to_json("BodyTypes")),
      'engineType', (SELECT row_to_json("EngineTypes")),
      'gearboxType', (SELECT row_to_json("GearboxTypes"))
    ) as "car"
    FROM "Cars" as c
    LEFT JOIN "Brands" as brand ON c."idBrand" = brand.id
    LEFT JOIN "Models" as model ON c."idModel" = model.id
    LEFT JOIN "BodyTypes" as bt ON c."idBodyType" = bt.id
    LEFT JOIN "EngineTypes" as et ON c."idEngineType" = et.id
    LEFT JOIN "GearboxTypes" as gt ON c."idGearboxType" = gt.id
    GROUP BY c.id, brand.id, model.id, bt.id, et.id, gt.id
    ORDER BY brand.value ASC, model.value ASC
</details>


So why do you need to use this lib for your complex queries?

#### The reason #1 is KeyPaths!
If you will change your models in the future you'll have to remember where you used links to this model properties and rewrite them manually and if you forgot one you will get headache in production. But with KeyPaths you will be able to compile your project only while all links to the models properties are up to date. Even better, you will be able to use `refactor` functionality of Xcode! 😄
#### The reason #2 is `if/else` statements
With `FluentQuery`'s query builder you can use `if/else` wherever you need. And it's super convenient to compare with using `if/else` while createing raw query string. 😉
#### The reason #3
It is faster than multiple consecutive requests

To tell you the truth `Fluent` can do queries like this and it will look better and will take the same time

But if to speak about real complex queries when you need to join on joined values, count something on the fly and so on - here this lib can be super useful! 🔥

### Methods

The list of the methods which `FluentQuery` provide with

#### Select
These methods will add fields which will be used between `SELECT` and `FROM`

`SELECT _here_some_fields_list_ FROM`

So to add what you want to select call these methods one by one

| Method  | SQL equivalent |
| ------- | -------------- |
| .select("*") | * |
| .select(all: Car.self) | "Cars".* |
| .select(all: someAlias) | "some_alias".* |
| .select(\Car.id) | "Car".id |
| .select(someAlias.k(\.id)) | "some_alias".id |
| .select(distinct: \Car.id) | DISTINCT "Car".id |
| .select(distinct: someAlias.k(\.id)) | DISTINCT "some_alias".id |
| .select(count: \Car.id) | DISTINCT "Car".id |
| .select(count: someAlias.k(\.id)) | DISTINCT "some_alias".id |

_Yeah-yeah in the future I have to add all available aggregate functions(almost done), and also json functions._

_BTW, read about aliases below_

#### From

| Method  | SQL equivalent |
| ------- | -------------- |
| .from("Table") | FROM "Table" |
| .from(raw: "Table") | FROM Table |
| .from(Car.self) | FROM "Cars" as "_cars_" |
| .from(someAlias) | FROM "SomeAlias" as "someAlias" |

#### Join

`.join(FQJoinMode, Table, where: FQWhere)`

```swift
enum FQJoinMode {
    case left, right, inner, outer
}
```

As `Table` you can put `Car.self` or `someAlias`

_About `FQWhere` please read below_


#### Where

`.where(FQWhere)`

`FQWhere(predicate).and(predicate).or(predicate).and(FQWhere).or(FQWhere)`

##### What `predicate` is?
It may be `KeyPath operator KeyPath` or `KeyPath operator Value`

`KeyPath` may be `\Car.id` or `someAlias.k(\.id)`

`Value` may be any value like int, string, uuid, array, or even something optional or nil

List of available operators you saw above in cheatsheet

Some examples

```swift
FQWhere(someAlias.k(\.deletedAt) == nil)
FQWhere(someAlias.k(\.id) == 12).and(\Car.color ~~ ["blue", "red", "white"])
FQWhere(\Car.year == "2018").and(\Brand.value !~ ["Chevrolet", "Toyota"])
FQWhere(\Car.year != "2005").and(someAlias.k(\.engineCapacity) > 1.6)
```

##### Where grouping example

if you need to group predicates like

```sql
"Cars"."engineCapacity" > 1.6 AND ("Brands".value LIKE '%YO%' OR "Brands".value LIKE '%ET')
```

then do it like this

```swift
FQWhere(\Car.engineCapacity > 1.6).and(FQWhere(\Brand.value ~~ "YO").or(\Brand.value ~= "ET"))
```

##### Cheatsheet
| Operator  | SQL equivalent |
| -- | --- |
| == | == / IS |
| != | != / IS NOT|
| > | > |
| < | < |
| >= | >= |
| <= | <= |
| ~~ | IN () |
| !~ | NOT IN () |
| ~= | LIKE '%str' |
| ~~ | LIKE '%str%' |
| =~ | LIKE 'str%' |
| !~= | NOT LIKE '%str' |
| !~~ | NOT LIKE '%str%' |
| !=~ | NOT LIKE 'str%' |

#### Having

`.having(FQWhere)`

About `FQWhere` you already read above, but as having calls after data aggregation you may additionally filter your results using aggreagate functions such as `SUM, COUNT, AVG, MIN, MAX`

```swift
.having(FQWhere(.count(\Car.id) > 0))
//OR
.having(FQWhere(.count(someAlias.k(\.id)) > 0))
//and of course you an use .and().or().groupStart().groupEnd()
```

#### Group by

```swift
.groupBy(FQGroupBy(\Car.id).and(\Brand.id).and(\Model.id))
```

#### Order by
```swift
.orderBy(FQOrderBy(\Car.year, .ascending).and(someAlias.k(\.name), .descending))
```

#### Offset

| Method  | SQL equivalent |
| ------- | -------------- |
| .offset(0) | OFFSET 0 |

#### Limit

| Method  | SQL equivalent |
| ------- | -------------- |
| .limit(30) | LIMIT 30 |

### JSON

You can build `json` on `jsonb` object by creating `FQJSON` instance

| Instance  | SQL equivalent |
| --------- | -------------- |
| FQJSON(.normal) | build_json_object() |
| FQJSON(.binary) | build_jsonb_object() |

After creating instance you should fill it by calling `.field(key, value)` method like

```swift
FQJSON(.binary).field("brand", \Brand.value).field("model", someAlias.k(\.value))
```

as you may see it accepts keyPaths and aliased keypaths

but also it accept function as value, here's the list of available functions

| Function  | SQL equivalent |
| --------- | -------------- |
| rowToJson(Car.self) | SELECT row_to_json("Cars") |
| rowToJson(someAlias) | SELECT row_to_json("some_alias") |
| extractEpochFromTime(\Car.createdAt) | extract(epoch from "Cars"."createdAt") |
| extractEpochFromTime(someAlias.k(\.createdAt)) | extract(epoch from "some_alias"."createdAt") |
| count(\Car.id) | COUNT("Cars".id) |
| count(someAlias.k(\.id)) | COUNT("some_alias".id) |
| countWhere(\Car.id, FQWhere(\Car.year == "2012")) | COUNT("Cars".id) filter (where "Cars".year == '2012') |
| countWhere(someAlias.k(\.id), FQWhere(someAlias.k(\.id) > 12)) | COUNT("some_alias".id) filter (where "some_alias".id > 12) |


### Aliases

`FQAlias<OriginalClass>(aliasKey)`

When you write complex query you can several joins or subqueries to the same table and you need to use aliases for that like `"Cars" as c`

So with FluentQuery you can create aliases like this

```swift
let aliasBrand = FQAlias<Brand>("b")
let aliasModel = FQAlias<Model>("m")
let aliasEngineType = FQAlias<EngineType>("e")
```

and you can use KeyPaths of original tables referenced to these aliases like this

```swift
aliasBrand.k(\.id)
aliasBrand.k(\.value)
aliasModel.k(\.id)
aliasModel.k(\.value)
aliasEngineType.k(\.id)
aliasEngineType.k(\.value)
```

## Known Issues

Looks like predicates doesn't work properly with `Date` properties cause I should format date properly in `FQPredicate` class

`FQSelect` methods should support `FQAggregate`
