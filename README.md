# Adwize - Data Collection Monitoring (GTM Template)

Monitor data quality at the source. This Google Tag Manager template captures all dataLayer events and tag firing status, sending them to the [Adwize](https://getadwize.com) API for quality monitoring, anomaly detection, and alerting.

## What it does

- Listens to every GTM event via `addEventCallback`
- Captures which tags fired, their status, and execution time
- Sends a lightweight pixel request (`GET`) to the Adwize API
- Supports filtering by all events, e-commerce only, or a custom event list

## Setup

1. **Import the template** into your GTM container: **Templates** > **Tag Templates** > **New** > three-dot menu > **Import** > select `template.tpl`
2. **Create a tag**: **Tags** > **New** > choose **Adwize - Data collection monitoring**
3. **Paste your API key** from the Adwize dashboard (**Settings > API Keys**)
4. **Choose a trigger mode**: All Events (recommended), E-commerce Only, or Custom Event List
5. **Add a trigger**: select **All Pages**
6. **Preview** to verify events are sent, then **Publish**

## Configuration Fields

| Field | Description |
|---|---|
| **API Key** | Your Adwize tenant API key (required) |
| **When to Send Data** | `All Events` (default), `E-commerce Events Only`, or `Custom Event List` |
| **Custom Events** | Comma-separated event names (only shown when Custom is selected) |
| **Capture full dataLayer** | Include the entire dataLayer snapshot in each event (Advanced) |
| **Capture e-commerce items** | Include product/item arrays for e-commerce events (default: on) |
| **Enable debug logging** | Log events to browser console (Advanced, disable in production) |

## How it works

```
User visits page
  → GTM fires event (e.g. page_view, purchase)
    → Adwize tag calls addEventCallback
      → Callback receives tag firing data
        → Builds JSON payload (event, tags, metadata)
          → sendPixel GET request to Adwize API
            → Adwize processes and monitors data quality
```

## Tests

The template includes 7 unit tests covering all trigger modes, edge cases (no tags, filtered events), and URL construction. After importing, open the template editor > **Tests** tab > click **Run Tests** to validate.

## Support

- Documentation: [docs.getadwize.com](https://docs.getadwize.com/integrations/google-tag-manager)
- Email: support@getadwize.com
- Website: [getadwize.com](https://getadwize.com)

## License

Apache 2.0 - see [LICENSE](LICENSE)
