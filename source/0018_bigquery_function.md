# BigQueryで変数定義 ＆ 関数定義

```SQL
# timestamp, date, _TABLE_SUFFIXの定義
CREATE TEMP FUNCTION StartTime1() RETURNS timestamp AS ('2023-01-01 00:00:00');
CREATE TEMP FUNCTION StartDate1() RETURNS DATE AS (DATE(StartTime1()));
CREATE TEMP FUNCTION StartSuffix1() RETURNS STRING AS (format_date('%Y%m%d',StartDate1()));
CREATE TEMP FUNCTION EndTime1() RETURNS timestamp AS ('2023-01-31 23:59:59');
CREATE TEMP FUNCTION EndDate1() RETURNS DATE AS (DATE(EndTime1()));
CREATE TEMP FUNCTION EndSuffix1() RETURNS STRING AS (format_date('%Y%m%d',EndDate1()));
# 例：UTC→JSTに変換する関数
CREATE TEMP FUNCTION utc2jst(date TIMESTAMP) AS (TIMESTAMP_ADD(date, INTERVAL 9 HOUR));
```