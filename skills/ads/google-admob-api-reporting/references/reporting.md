# Query the AdMob API

Use the [v1 REST API](https://developers.google.com/admob/api/reference/rest) for publisher settings, inventory, and performance. The examples use Bash, `curl` with header-file support, and `jq`. Run them in a private terminal with an [access token](oauth-setup.md) in the `admob_token` shell variable. Keep shell tracing off. The helper supplies the bearer header through standard input.

```bash
admob_base='https://admob.googleapis.com/v1'

admob_request() {
  if [[ -z ${admob_token:-} ]]; then
    printf '%s\n' 'Get an AdMob access token first.' >&2
    return 1
  fi
  printf 'Authorization: Bearer %s\n' "$admob_token" |
    curl --disable --fail --silent --show-error --max-time 120 --header @- "$@"
}
```

## Resolve the publisher and inventory

`accounts.list` can return the publisher most recently used in the AdMob UI. Confirm the result against the user's intended publisher. Use its `pub-…` ID with `accounts.get` to check the reporting time zone and currency. An app ID contains `~` and an ad unit ID contains `/`. Join ad units to apps by `appId`. [List publishers](https://developers.google.com/admob/api/reference/rest/v1/accounts/list), [Get a publisher](https://developers.google.com/admob/api/reference/rest/v1/accounts/get), [Retrieve inventory](https://developers.google.com/admob/api/v1/app-ad-units)

Follow `nextPageToken` for each listing. This helper emits one JSON record per line and returns a failure if a page request or parse fails. Discard partial output when a listing fails.

```bash
admob_list() {
  local path=$1 collection=$2 response page_token next_token
  response=$(admob_request "$admob_base/$path") || return
  page_token=''
  while true; do
    jq -c --arg key "$collection" '.[$key] // [] | .[]' <<<"$response" || return
    next_token=$(jq -r '.nextPageToken // empty' <<<"$response") || return
    [[ -n $next_token ]] || return 0
    if [[ $next_token == "$page_token" ]]; then
      printf '%s\n' 'AdMob repeated a page token.' >&2
      return 1
    fi
    page_token=$next_token
    response=$(admob_request --get --data-urlencode "pageToken=$page_token" \
      "$admob_base/$path") || return
  done
}

admob_list accounts account
publisher_id='pub-1234567890123456' # Replace with the verified publisher ID.
admob_request "$admob_base/accounts/$publisher_id"

# These two inventory requests require admob.readonly.
admob_list "accounts/$publisher_id/apps" apps
admob_list "accounts/$publisher_id/adUnits" adUnits
```

## Generate a network or mediation report

Use the network endpoint for AdMob Network performance. Use mediation to cover mediated sources or break results down by `AD_SOURCE`. Choose one report for the requested coverage because their data can overlap. Both endpoints accept `admob.report` or `admob.readonly`. [Network report](https://developers.google.com/admob/api/reference/rest/v1/accounts.networkReport/generate), [Mediation report](https://developers.google.com/admob/api/reference/rest/v1/accounts.mediationReport/generate)

Replace the inclusive dates in the sample. Use a completed period and the account's reporting time zone for relative dates unless the user specifies another. Omit `timeZone` for the account default. The v1 API documents `America/Los_Angeles` as its sole explicit override. If a requested day falls across report buckets in another time zone, disclose that difference. Specify at most one of `DATE`, `WEEK`, or `MONTH`. Check each endpoint's field compatibility before changing the request.

```bash
network_spec=$(cat <<'JSON'
{
  "reportSpec": {
    "dateRange": {
      "startDate": {"year": 2026, "month": 1, "day": 1},
      "endDate": {"year": 2026, "month": 1, "day": 7}
    },
    "dimensions": ["DATE", "APP"],
    "metrics": ["IMPRESSIONS", "ESTIMATED_EARNINGS"]
  }
}
JSON
)

admob_generate() {
  local report_type=$1 spec=$2
  admob_request --header 'Content-Type: application/json' --data-binary "$spec" \
    "$admob_base/accounts/$publisher_id/${report_type}Report:generate"
}
```

For the network report, run this command.

```bash
report=$(admob_generate network "$network_spec")
```

For a mediation report grouped by ad source, run these commands.

```bash
mediation_spec=$(jq -c '.reportSpec.dimensions += ["AD_SOURCE"]' <<<"$network_spec")
report=$(admob_generate mediation "$mediation_spec")
```

## Validate and interpret the report

REST returns the streamed messages as an array containing a header, any data rows, and a footer. Check the envelope before using the rows. Inspect the effective `reportingTimeZone`, currency, and inclusive dates in the header. Check `footer.warnings`, the number of returned rows, and `matchingRowCount`, which can differ from the number of rows sent. [Report header](https://developers.google.com/admob/api/reference/rest/v1/ReportHeader), [Report footer](https://developers.google.com/admob/api/reference/rest/v1/ReportFooter)

```bash
jq -e '
  if type == "array" and length >= 2
    and (.[0].header | type == "object")
    and (.[-1].footer | type == "object")
    and all(.[1:-1][]; (.row | type == "object"))
  then {header: .[0].header, rowsReturned: (length - 2), footer: .[-1].footer}
  else error("Incomplete or unexpected AdMob report")
  end
' <<<"$report"
```

- If the report reaches its configured row limit or the API maximum of 100,000, check coverage before using totals. Shorten the date range or reduce the number of dimensions, then combine additive totals across non-overlapping slices. [Report limits](https://developers.google.com/admob/api/v1/report-overview#reduce-report-limits)
- Group rows by raw dimension `value` and use `displayLabel` for presentation when available. Parse `integerValue` and `microsValue` as exact integers. Divide monetary micros by 1,000,000 and use the currency in the report header. Format ratio `doubleValue` fields as percentages when appropriate. Label `ESTIMATED_EARNINGS` as estimated and distinguish missing fields or dates from observed zeros. [Report row types](https://developers.google.com/admob/api/reference/rest/v1/ReportRow)
- Recompute grouped rates using totals. CTR is clicks divided by impressions, match rate is matched requests divided by ad requests, and show rate is impressions divided by matched requests. Earnings-based eCPM is earnings in currency units divided by impressions and multiplied by 1,000. Use an unavailable value for a zero or missing denominator. `OBSERVED_ECPM` is a separate mediation metric. [Network metrics](https://developers.google.com/admob/api/reference/rest/v1/accounts.networkReport/generate#metric), [Mediation metrics](https://developers.google.com/admob/api/reference/rest/v1/accounts.mediationReport/generate#metric)
- For unique app-level ad requests, query without `AD_SOURCE` or `AD_SOURCE_INSTANCE`. One request can go to multiple sources, so adding source-level request counts can overstate app-level volume. [AdMob request counting](https://support.google.com/admob/answer/10979428)

For HTTP `400`, verify the report type, compatible dimensions and metrics, dates, and time zone. For `401` or `403`, verify token freshness, scope, API enablement, and publisher access. For `429` or a transient server error, retry with bounded backoff. If results differ from the UI, compare publisher, report coverage, filters, dates, time zone, and currency. [AdMob API errors](https://developers.google.com/admob/api/v1/errors)

When done, run `unset admob_token report` and keep any exported report in the user's private workspace.
