# Standard reporting API service

## 设计目标

* 所有`维度`支持筛选。
* 所有`指标`支持（聚合后）过滤。
* 所有`维度`支持分组。
* 所有`指标`支持支持可选查询（提升查询响应）。
* 所有`指标`和`维度`支持排序。

## 对象说明

* Dimension：定义了维度的标准特征。
* Metrics：定义了指标的标准特征。
* StandardApiRequest：定义了标准报表请求的属性。
* StandardApiResponse：定义了标准报表响应的属性。
* SqlGenerator：因为都是标准的，所以可使用SqlGenerator直接生成SQL查询语句，减少开发和维护的工作量。


## 示例：开发一个报表接口

1. 定义纬度

```java
public enum AdDailyDimension implements Dimension {
    stat_date,
    account_id("dim_ads_account", "id", "email", "account");
}
  ```

2. 定义指标

```java
public enum AdDailyMetrics implements Metrics {
  impression,
  click,
  ctr(true, "click / impression", click, impression);
}
```

3. 创建请求对象

```java
public class AdDailyRequest extends AbstractStandardRequest<AdDailyMetrics, AdDailyDimension> {
}
```

4. 创建Controller

```java

@RestController
@RequestMapping(path = "/v1.0/reports")
public class ReportController {

  @Resource
  private ParallelQuery parallelQuery;
  @Resource
  private JdbcTemplate jdbcTemplate;

  @PostMapping(path = "/ad-dailies/get")
  public DefaultStandardApiResponse get(@RequestBody AdDailyRequest request)
          throws ExecutionException, InterruptedException {
    SqlGenerator sql = request.toSqlGenerator("dwm_ad_daily", "report");
    return parallelQuery.query(sql, jdbcTemplate)
            .buildPage(request.getPage(), request.getPage_size())
            .build();
  }
}
```

5. 启动程序，开始对接口发送请求。

_请求示例：_

```shell
curl --request POST \
  --url http://localhost:8080/v1.0/reports/ad-dailies/get \
  --header 'content-type: application/json' \
  --data '{
	"groupings": [
		"stat_date"
	],
	"metrics": [
		"cost"
	],
	"conditions": [
		
		{
			"key": "stat_date",
			"op": "GE",
			"value": "2023-05-01"
		}
	]
}'
```

_响应示例：_

```json
{
  "code": 200,
  "msg": "OK",
  "data": {
    "list": [
      {
        "stat_date": "2023-05-01",
        "cost": 3361.957789
      },
      {
        "stat_date": "2023-05-02",
        "cost": 55426.267316
      }
    ],
    "sum": {
      "cost": 39631843.472675
    },
    "page": 1,
    "page_size": 10,
    "total_number": 101,
    "total_page": 11
  },
  "traceId": "1692843711099jKK07Z"
}
```

## 数据导出

在请求头中加上`Accept: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`。

### 导出美化
增加请求参数 `export_format`

_示例：_
```json
{
	"groupings": [
		"material_id"
	],
	"export_format": [
		{
			"column": "stat_date",
			"title": "日期"
		},
		{
			"column": "platform_name",
			"title": "媒体"
		},
		{
			"column": "cost",
			"title": "消耗",
			"metrics": true
		},
		{
			"column": "cpi",
			"title": "CPI",
			"metrics": true
		},
		{
			"column": "click",
			"title": "点击",
			"metrics": true
		}
	]
}
```

> column: 列名；title: 标题；metrics: 是否示指标；
