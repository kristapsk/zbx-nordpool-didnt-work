# Zabbix template for nordpool.didnt.work

Zabbix 7.4 template for collecting Nord Pool day-ahead electricity prices for **Latvia (LV)**, **Lithuania (LT)** and **Estonia (EE)** directly from [nordpool.didnt.work](https://nordpool.didnt.work/).

The template uses only native Zabbix functionality:

- HTTP agent master items
- dependent items
- JavaScript preprocessing
- scheduled checks aligned to 15-minute market intervals

No external scripts, cron jobs, `zabbix_sender` or other middleware are required.

## Requirements

- Zabbix 7.4
- Zabbix server or proxy with outbound HTTPS access to `https://nordpool.didnt.work`

The API schema is published at `https://nordpool.didnt.work/api/openapi.json`.

## Installation

1. In Zabbix, go to **Data collection -> Templates**.
2. Click **Import**.
3. Import `template_nordpool_didnt_work.yaml`.
4. Create a host, for example `Nord Pool`, with no agent interface required.
5. Link the template **Nord Pool prices by didnt.work** to the host.

The template fetches all three Baltic bidding areas on the same host.

## Collected items

For each country (`lv`, `lt`, `ee`) the template creates one HTTP master item and several dependent items.

| Item key | Meaning |
|---|---|
| `nordpool.api.raw[lv]` | Raw 15-minute API response for Latvia |
| `nordpool.price.current[lv]` | Price for the interval containing the current time |
| `nordpool.price.previous[lv]` | Previous 15-minute interval price |
| `nordpool.price.next[lv]` | Next 15-minute interval price when present in today's payload |
| `nordpool.price.today.avg[lv]` | Arithmetic average of today's 15-minute prices |
| `nordpool.price.today.min[lv]` | Minimum price today |
| `nordpool.price.today.max[lv]` | Maximum price today |

The same keys exist with `[lt]` and `[ee]`.

All price items are stored in **EUR/kWh**. The API exposes prices in **EUR/MWh**, so preprocessing divides the value by `1000`.

## Polling and timestamps

The HTTP master items use this Zabbix scheduling interval:

```text
0;m/15s5
```

This means there is no normal polling interval; checks are scheduled at approximately:

```text
00:00:05
00:15:05
00:30:05
00:45:05
...
```

The five-second offset avoids hitting the API exactly on the interval boundary. As a result, the stored `current` price timestamp is a few seconds after the actual market interval start, but each value represents the interval whose `start <= now < end` according to the API timestamps.

## Precision

The template does **not** round prices.

Example API price:

```text
221.18 EUR/MWh
```

is stored as:

```text
0.22118 EUR/kWh
```

Zabbix 7.4 `Numeric (float)` uses a 64-bit floating-point value, so the stored value has substantially more precision than four decimal places. Some Zabbix frontend views may format floating-point values with fewer visible decimals; that is separate from database storage precision.

## Macros

### `{$NORDPOOL.API.URL}`

Default:

```text
https://nordpool.didnt.work
```

Change this only if using a mirror or reverse proxy.

### `{$NORDPOOL.VAT}`

Default:

```text
false
```

Set to `true` if VAT-inclusive prices are desired.

For energy-cost accounting it is usually better to keep the market price and additional fees/taxes as separate components, so the default is VAT-free.

## How `current`, `previous` and `next` are selected

The API returns entries of the form:

```json
{
  "start": "...",
  "end": "...",
  "value": 221.18
}
```

The `current` dependent item finds the entry for which:

```text
start <= current time < end
```

The adjacent array entry is used for `previous` or `next`.

At the first interval of a day there is no previous entry in the current-day payload, and at the last interval there is no next entry. In those cases the value is discarded instead of making the item unsupported.

## Daily statistics

`today.avg`, `today.min` and `today.max` are calculated from all 15-minute entries in the API response.

Because all 15-minute periods have equal duration, the arithmetic mean is also the correct hourly/daily time-weighted average when all intervals are present.

The daily statistic items use `Discard unchanged with heartbeat = 1d` to avoid storing the same daily statistic every 15 minutes.

## Recommended graph

For a simple comparison graph add these three items to the same Graph widget:

```text
nordpool.price.current[lv]
nordpool.price.current[lt]
nordpool.price.current[ee]
```

This gives one 15-minute price history series per Baltic bidding area.

## Important limitation: no historical backfill

This template records the price that is current when Zabbix polls the API. It does not retroactively insert all 96 prices from the daily payload with custom timestamps.

Therefore, if Zabbix is down for two hours, those missing eight 15-minute history points remain missing even though the source API may still know them.

Backfilling historical prices would require a different mechanism such as Zabbix API `history.push` or `zabbix_sender` with explicit timestamps. That is intentionally outside the scope of this template.

## API usage

The template performs three HTTP requests every 15 minutes: one each for LV, LT and EE. That is 288 requests per day in total.

If only one or two countries are needed, the unused `nordpool.api.raw[...]` master items can be disabled; their dependent items will then stop updating as well.

## Validation / test notes

The template was built against the published OpenAPI schema for:

```text
GET /api/{country}/prices
```

with:

```text
country = lv | lt | ee
resolution = 15
vat = false (default)
```

The preprocessing logic expects the documented `prices[]` array with `start`, `end` and numeric `value` fields.

Before publishing a release, it is recommended to verify in a live Zabbix 7.4 instance that:

1. all three master HTTP items become supported;
2. `current` matches the corresponding 15-minute value on nordpool.didnt.work;
3. the stored value has the expected EUR/MWh -> EUR/kWh conversion;
4. successive history entries change on the 00/15/30/45 minute boundaries;
5. `next` and `previous` behave as expected around midnight.

## License

Choose a license appropriate for your repository before publishing (for example MIT).

## Data source

Price data is provided by `nordpool.didnt.work`, which in turn publishes Nord Pool day-ahead price data. Review the upstream site's terms and attribution requirements before redistributing data.
