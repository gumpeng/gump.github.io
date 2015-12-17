ES(elasticsearch)的suggester共有四类(term suggester, phrase suggester, completion suggester, context suggester), 其中completion suggester作为搜索框中的自动补齐功能，尤为常用。

本文将用java语言实现一个简单例子来叙述如何使用elasticsearch中的completion suggester功能。例子的主要功能是为股票的名称和编号建立自动补齐功能。

实现一个完整的completion suggester功能，需要三个步骤：建立schema，索引数据，搜索数据。下面分别进行介绍。

##1.建立schema
schema对于ES来说，就如同一个database的表格定义，它用于预先定义各个字段的名称以及类型等。
需要被自动补齐的数据，得用一个类型为completion的字段来存储待补齐数据。具体如下：

```java
{
  "stock_suggest" : {
    "mappings" : {
      "stock" : {
        "_id" : {
          "path" : "id"
        },
        "properties" : {
          "id" : {
            "type" : "string",
            "analyzer" : "keyword"
          },
          "name" : {
            "type" : "string",
            "index" : "not_analyzed"
          },
          "nameSuggest" : {
            "type" : "completion",
            "analyzer" : "standard",
            "payloads" : true,
            "preserve_separators" : false,
            "preserve_position_increments" : false,
            "max_input_length" : 50
          }
        }
      }
    }
  }
}
```
需要说明的是payloads属性被设置为true是为了激活自动补齐时，返回一个payload字段用于承载预定义好（在索引数据时定义）的数据信息。

##2. 索引数据
下面将待补齐数据注入到ES 中。代码如下：

```
String esHosts = "";
        String clusterName = "";
        GsonBuilder gsonBuilder = new GsonBuilder().setDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSSZ");
        Gson gson = gsonBuilder.create();
        Settings settings = ImmutableSettings.settingsBuilder().put("client.transport.sniff", true).put("cluster.name", clusterName).put("node.client", true).build();
        Client client = new TransportClient(settings).addTransportAddress(new InetSocketTransportAddress(esHosts, 9300));
        BulkRequestBuilder bulkRequest = client.prepareBulk();
        List<Stock> stocks = new ArrayList();
        stocks.add(new Stock("601390", "中国中铁"));
        stocks.add(new Stock("601186", "中国铁建"));
        stocks.add(new Stock("601766", "中国中车"));
        stocks.add(new Stock("600115", "东方航空"));
        stocks.add(new Stock("000585", "东北电气"));
        stocks.add(new Stock("000527", "美的电器"));
        for (Stock stock : stocks) {
            List<String> input = new ArrayList<String>();
            input.add(stock.getName());
            input.add(stock.getId());
            Map<Object, Object> payload = new HashMap<Object, Object>();
            payload.put("id", stock.getId());
            payload.put("name", stock.getName());
            payload.put("type", "stock");
            NameSuggest nameSuggest = new NameSuggest(input, payload, stock.getId());
            stock.setNameSuggest(nameSuggest);
            JsonObject jo = (JsonObject) gson.toJsonTree(stock);
            String jsonSource = gson.toJson(jo);
            bulkRequest.add(client.prepareIndex(index, type, stock.getId()).setSource(jsonSource));
        }
        BulkResponse bulkResponse = bulkRequest.execute().actionGet();
```
如上所示，我们插入了六条股票数据做为样例。其中第一行的esHosts和第二行的clusterName需要根据自己的ES集群配置自行设定。

##3.搜索数据　
数据建完索引后，就可以使用自动补齐功能啦。

```java
String input = "60"
String clusterName = "";
String esHosts = "";
Settings settings = ImmutableSettings.settingsBuilder().put("client.transport.sniff", true).put("cluster.name", clusterName).put("node.client", true).build();
Client client = new TransportClient(settings).addTransportAddress(new InetSocketTransportAddress(esHosts, 9300));
String field = "nameSuggest";

SuggestRequestBuilder srb = client.prepareSuggest(index);
CompletionSuggestionBuilder csfb = new CompletionSuggestionBuilder(field).field(field).text(input).size(10);
srb = srb.addSuggestion(csfb);
SuggestResponse response = srb.execute().actionGet();
System.out.println(response);
```
上段代码的输出效果如下所示：

```
{
  "nameSuggest" : [ {
    "text" : "60",
    "offset" : 0,
    "length" : 2,
    "options" : [ {
      "text" : "600115",
      "score" : 1.0,
      "payload":{"name":"东方航空","id":"600115","type":"stock"}
    }, {
      "text" : "601186",
      "score" : 1.0,
      "payload":{"name":"中国铁建","id":"601186","type":"stock"}
    }, {
      "text" : "601390",
      "score" : 1.0,
      "payload":{"name":"中国中铁","id":"601390","type":"stock"}
    }, {
      "text" : "601766",
      "score" : 1.0,
      "payload":{"name":"中国中车","id":"601766","type":"stock"}
    } ]
  } ]
}
```
可以看出，编号以“60”开头的股票都被返回了。
好了，一个简单示例完成了，其中有些细节及相关原理将在后续文章中详细介绍。