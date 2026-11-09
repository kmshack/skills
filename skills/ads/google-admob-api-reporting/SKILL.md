---
name: google-admob-api-reporting
description: >-
  Retrieves Google AdMob publisher account details, app and ad unit inventory,
  and performance reports through the AdMob API. Use when setting up AdMob API
  OAuth, looking up publisher or inventory IDs, or analyzing estimated earnings,
  requests, impressions, match rate, or eCPM across apps, ad units, countries,
  or ad sources.
metadata:
  version: "1.0.0"
  category: GoogleAds
---
# Google AdMob API Reporting

Use this skill for publisher-side reporting and inventory. For SDK integration
or validation, use [Google Mobile Ads SDK - Install](../google-mobile-ads-get-started/SKILL.md)
or [Validate Google Mobile Ads SDK Integration](../google-mobile-ads-validate/SKILL.md).

## Workflow

1. **Resolve the scope.** Identify the publisher account and the requested
   operation. For reporting, resolve the inclusive date range, metrics, filters,
   and breakdown. Resolve relative dates in the effective report time zone,
   which defaults to the account's reporting time zone. Verify whether the API
   supports a requested override and explain any mismatch in day boundaries.
2. **Check authorization.** Use an existing authorized AdMob integration if one
   is available. For direct requests or first-time consent, follow
   [OAuth setup](references/oauth-setup.md). Use `admob.report` for account
   details and reports, or `admob.readonly` if the request needs inventory.
   Keep OAuth credentials and raw account data in the user's private environment.
3. **Discover resources.** Confirm the `accounts/pub-…` resource before querying
   inventory or reports. Use the app and ad unit list methods to resolve IDs for
   filters when the authorized scope permits it. Check for additional pages.
4. **Choose the report.** Use `networkReport:generate` for AdMob Network
   performance and `mediationReport:generate` for performance across mediated ad
   sources. Use the [REST examples](references/reporting.md) to build the report
   and verify that its dimensions and metrics are compatible. Keep these report
   types separate because they can cover overlapping traffic.
5. **Validate and summarize.** Check the report header, footer, warnings, and row
   count. Distinguish a complete report with no rows, a measured zero, an
   incomplete result, and a failed request. State the publisher scope, report
   type, dates, reporting time zone, currency, and any coverage limitations.
   Label earnings as estimated and derive aggregate rates from their component
   totals.

If live access is unavailable, provide the request specification and the
authorization step needed to run it. Base conclusions about performance on
retrieved data or a user-supplied export.
