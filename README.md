# Adwize - Data Collection Monitoring (GTM Template)

Monitor data quality at the source. This Google Tag Manager template captures dataLayer events — event name, optional e-commerce/items, optional full dataLayer snapshot, and container metadata — and forwards them to the [Adwize](https://getadwize.com) API for quality monitoring, anomaly detection, and alerting.

It does **not** report whether other GTM tags (GA4, Ads, etc.) fired successfully. The template only observes and forwards dataLayer events; it has no visibility into tag firing success or failure for other tags in the container.

## What it does

- Captures dataLayer events (e.g. `page_view`, `purchase`) as they occur
- Collects e-commerce data, transaction details, and container metadata
- Sends a lightweight pixel request (`GET`) to the Adwize API
- Supports filtering by all events, e-commerce only, or a custom event list
- Does **not** observe or report whether other GTM tags fired successfully

## Setup

1. **Import the template** into your GTM container: **Templates** > **Tag Templates** > **New** > three-dot menu > **Import** > select `template.tpl`
2. **Create a tag**: **Tags** > **New** > choose **Adwize - Data collection monitoring**
3. **Paste your API key** from the Adwize dashboard (**Settings > API Keys**)
4. **Choose a trigger mode**: All Events (recommended), E-commerce Only, or Custom Event List
5. **Add a trigger**: select **All Pages**
6. **Preview** to verify events are sent, then **Publish**

> **Security note:** The collect request is a GET pixel, so the API key travels in the query string (`?k=`). That can appear in browser history, CDN/proxy/WAF logs, and some error-page referrers. Prefer network controls, rotate the key if logs are shared, and treat this as write-ingest credentials — not a dashboard login.

## Configuration Fields

| Field | Description |
|---|---|
| **API Key** | Your Adwize tenant API key (required). Sent in the collect URL query string — see security note above. |
| **When to Send Data** | `All Events` (default), `E-commerce Events Only`, or `Custom Event List` |
| **Custom Events** | Comma-separated event names (only shown when Custom is selected) |
| **Capture full dataLayer** | Include the entire dataLayer snapshot in each event (Advanced). Large snapshots can exceed GET URL limits — see below. |
| **Capture e-commerce items** | Include product/item arrays for e-commerce events (default: on) |
| **Enable debug logging** | Log events to browser console (Advanced, disable in production) |

## Payload size (GET URL limits)

Events are sent as a `GET` pixel with the JSON payload in the query string. Large payloads — especially with **Capture full dataLayer** enabled, or large e-commerce `items` arrays — can exceed typical browser/proxy URL length limits (~2k–8k characters). When that happens, the request may fail silently unless debug logging is enabled. Prefer e-commerce-only or a custom event list when possible, and leave full dataLayer capture off unless you need it for debugging.

## How it works

```
User visits page
  → GTM fires event (e.g. page_view, purchase)
    → Adwize tag reads event from dataLayer
      → Collects e-commerce data, transaction details, container metadata
        → Builds JSON payload (event, data, metadata)
          → sendPixel GET request to Adwize API
            → Adwize processes and monitors data quality
```

## Tests

The template includes 8 unit tests covering all trigger modes, edge cases (null events, filtered events), and URL construction. After importing, open the template editor > **Tests** tab > click **Run Tests** to validate.

## Support

- Documentation: [docs.getadwize.com](https://docs.getadwize.com/integrations/google-tag-manager)
- Email: support@getadwize.com
- Website: [getadwize.com](https://getadwize.com)

## License

Apache 2.0 - see [LICENSE](LICENSE)
