# 关键代码节选

以下为现有实现的局部节选，分别展示 Swift 端的目录搜索和 Java 端的报价入库。节选依赖项目内的模型或数据库上下文。

## Swift：离线目录搜索

从卡名、编号和卡组名称构建搜索范围。名称排序以卡牌 ID 作为同名项的次排序键。

```swift
public func search(_ request: SearchRequest) async throws -> [ProductSummary] {
    guard request.category == nil || request.category == .tradingCards else { return [] }
    let words = request.query.split(whereSeparator: \.isWhitespace).map(String.init)
    guard !words.isEmpty else { return [] }
    guard request.sort == .popularity || request.sort == .name else {
        throw CatalogFailure.http(status: 422, code: "price_sort_requires_variant")
    }
    var matches = fixture.cards.filter { card in
        let text = [card.name, card.number, sets[card.setId]?.name ?? ""].joined(separator: " ")
        return words.allSatisfy { text.localizedCaseInsensitiveContains($0) }
    }
    if request.sort == .name { matches.sort { ($0.name, $0.cardId) < ($1.name, $1.cardId) } }
    return matches.compactMap(summary)
}
```

## Java：按卡牌版本和日期保存报价

先确认报价属于当前卡牌的某个版本，再按版本、来源、日期和价格类型检查已有记录。同日金额不一致时要求复核；重复导入一致的记录时不会新增快照。金额以最小货币单位保存。

```java
for (Map<String, Object> snapshot : objects(card, "price_snapshots")) {
    String variantID = string(snapshot, "variant_id");
    if (!variants.stream().anyMatch(v -> variantID.equals(v.get("id"))) ||
            !"tcgcsv-tcgplayer".equals(string(snapshot, "source_id")) ||
            !"marketPrice".equals(string(snapshot, "price_type")) ||
            !"USD".equals(string(snapshot, "currency"))) throw invalid("Invalid price identity");
    long amount = number(snapshot, "amount_minor_units");
    if (amount < 0) throw invalid("Negative price");
    LocalDate observed = LocalDate.parse(string(snapshot, "date"));
    String prior = jdbc.sql("SELECT amount_minor_units::text FROM price_snapshots WHERE " +
                    "variant_id=:variant AND source_id='tcgcsv-tcgplayer' AND observed_on=:date " +
                    "AND price_type='marketPrice'")
            .param("variant", variantID).param("date", observed)
            .query(String.class).optional().orElse(null);
    if (prior != null && Long.parseLong(prior) != amount)
        throw conflict("Immutable daily price changed; review source correction");
    jdbc.sql("INSERT INTO price_snapshots(variant_id,source_id,observed_on,price_type,currency_code," +
            "amount_minor_units,source_product_id,subtype_name,retrieved_at) VALUES " +
            "(:variant,'tcgcsv-tcgplayer',:date,'marketPrice','USD',:amount,:product,:subtype,:retrieved) " +
            "ON CONFLICT DO NOTHING")
            .param("variant", variantID).param("date", observed).param("amount", amount)
            .param("product", String.valueOf(number(snapshot, "tcgplayer_product_id")))
            .param("subtype", string(snapshot, "subtype_name"))
            .param("retrieved", OffsetDateTime.parse(string(snapshot, "retrieved_at"))).update();
    snapshots++;
}
```
