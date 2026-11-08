# Adwize - Data Collection Monitoring (GTM Template)

Monitor data quality at the source. This Google Tag Manager template captures dataLayer events — event name, optional e-commerce/items, optional full dataLayer snapshot, and container metadata — and forwards them to the [Adwize](https://getadwize.com) API for quality monitoring, anomaly detection, and alerting.

Optionally, it also reports whether the GTM tags you care about (GA4, Ads, etc.) fired and how they ran — see [Tag monitoring](#tag-monitoring-optional). The two answer different questions: dataLayer capture checks that the site sends the right data into GTM; tag monitoring checks that GTM fires your key tags without errors.

## What it does

- Captures dataLayer events (e.g. `page_view`, `purchase`) as they occur
- Collects e-commerce data, transaction details, and container metadata
- Sends a lightweight pixel request (`GET`) to the Adwize API
- Supports filtering by all events, e-commerce only, or a custom event list
- Records consent state (`ad_storage`, `analytics_storage`, `ad_user_data`, `ad_personalization`) with each payload
- Optionally reports tag status (`success`, `failure`, `exception`, `timeout`): totals for every tag, full detail for tags you mark with tag metadata

## Setup

1. **Import the template** into your GTM container: **Templates** > **Tag Templates** > **New** > three-dot menu > **Import** > select `template.tpl`
2. **Create a tag**: **Tags** > **New** > choose **Adwize - Data collection monitoring**
3. **Paste your API key** from the Adwize dashboard (**Settings > API Keys**)
4. **Choose a trigger mode**: All Events (recommended), E-commerce Only, or Custom Event List
5. **Add triggers** for the events you want to monitor, for example a Custom Event trigger on your conversion events (`^(purchase|add_to_cart|begin_checkout)$`, regex matching). The tag only sees the events its triggers fire on.
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
| **Report status of monitored tags** | After each event this tag fires on, send status totals for every tag plus full detail for tags with metadata `monitor` = `true` (Tag Monitoring, default: off) |

## Tag monitoring (optional)

Uses GTM's [`addEventCallback`](https://developers.google.com/tag-platform/tag-manager/templates/monitoring) API. At the end of each event this tag fires on, it reads the tags that fired for that event and sends an `adwize.tag_status` event.

1. On each tag to monitor: **Advanced Settings** > **Additional Tag Metadata** > **Add Metadata**, key `monitor`, value `true`. Check **Include Tag Name** with key `name` to report tag names.
2. In the Adwize tag: **Tag Monitoring** > check **Report status of monitored tags**.
3. Make the Adwize tag's trigger cover **every event a monitored tag fires on**. A monitored tag that fires on an event the Adwize tag does not fire on is not reported — it is missing, not failed. Watch for tags triggered by Page View (`gtm.js`), DOM Ready (`gtm.dom`), clicks (`gtm.click`), form submits (`gtm.formSubmit`) or Initialization (`gtm.init`), not only custom events. A Custom Event trigger with regex matching, e.g. `purchase|gtm\.js|gtm\.formSubmit`, covers several at once.
4. Set the Adwize tag to fire **Once per event** (Advanced Settings > Tag firing options).

Tag monitoring ignores **When to Send Data**: the trigger alone decides which events are monitored. Both payloads carry `metadata.page_load_id` (random, shared by every event of a page load) and `metadata.event_id` (GTM's `gtm.uniqueEventId`, which restarts on each page load). Together they join a dataLayer event with its tag statuses.

Each `adwize.tag_status` event carries:

```json
{
  "event": "purchase",
  "part": {"index": 1, "count": 1},
  "summary": {"total": 5, "success": 2, "failure": 1, "exception": 1, "timeout": 1},
  "tags": [
    {"id": "12", "name": "GA4 - purchase", "status": "failure", "execution_time": 120, "platform": "ga4"}
  ],
  "unmonitored_issues": [
    {"id": "31", "status": "exception", "name": "Meta - Purchase"}
  ]
}
```

- `summary` counts every tag that fired for the event, monitored or not.
- `tags` lists the monitored tags, with every metadata key except `monitor` (for example `platform`, `priority`).
- `unmonitored_issues` lists tags without `monitor = true` whose status is not `success` (id, status, and name when Include Tag Name is on).
- A large report is split across several requests that share `page_load_id` and `event_id`. `part` numbers them, and only part 1 carries `summary`. Failed monitored tags are sent first, then unmonitored issues, then successful monitored tags. A single tag too large for one request is sent with `id`, `status` and `execution_time` only.

### What a status means

GTM records the status the tag itself reports.

- **Built-in Google tags and most vendor tags**: `success` means GTM ran the tag without an error, not that the vendor received or accepted the hit.
- **Custom HTML tags**: report the real result by referencing the built-in **HTML ID** variable and calling GTM back once the request has finished. GTM then waits for the call instead of marking the tag a success when it is injected; with no call before the event timeout, the status is `timeout`. Each tag has to be edited, so do it for the critical ones:

  ```html
  <script>
    (function() {
      var gtm = google_tag_manager['{{Container ID}}'];
      var img = new Image();
      img.onload = function() { gtm.onHtmlSuccess({{HTML ID}}); };
      img.onerror = function() { gtm.onHtmlFailure({{HTML ID}}); };
      img.src = 'https://vendor.example/pixel?...';
    })();
  </script>
  ```

- **Custom template tags**: the template must call `data.gtmOnSuccess()` / `data.gtmOnFailure()` from its `sendPixel` or `injectScript` callbacks, not right after sending.

### Consent

Both payload types carry `metadata.consent` with `ad_storage`, `analytics_storage`, `ad_user_data` and `ad_personalization` as read by `isConsentGranted` when the payload is built. A consent type that was never set reads as granted.

## Payload size (GET URL limits)

Events are sent as a `GET` pixel with the JSON payload in the query string. Large payloads — especially with **Capture full dataLayer** enabled, or large e-commerce `items` arrays — can exceed typical browser/proxy URL length limits (~2k–8k characters). When the collect URL exceeds ~1800 characters, the tag skips that `sendPixel` and always logs to the console (`[Adwize] Payload too large…`), even when debug logging is off. Prefer e-commerce-only or a custom event list when possible, and leave full dataLayer capture off unless you need it for debugging. Tag status reports are split across several requests instead of skipped (see [Tag monitoring](#tag-monitoring-optional)).

## How it works

```
User visits page
  → GTM fires event (e.g. page_view, purchase)
    → Adwize tag reads event from dataLayer
      → Collects e-commerce data, transaction details, container metadata
        → Builds JSON payload (event, data, metadata)
          → sendPixel GET request to Adwize API
            → Adwize processes and monitors data quality

With tag monitoring on, at the end of the same event:
  → GTM calls the event callback with the tags that fired
    → Adwize tag counts every tag and keeps full detail for tags with metadata monitor = true
      → one or more sendPixel GET requests with an adwize.tag_status event
```

## Server-side template

`template-server-side.tpl` is the same monitor for a **server** container. Import it there, not into the web container.

What matches the web template:

- The same trigger modes (all events, e-commerce only, custom list)
- Optional tag monitoring: metadata `monitor` = `true`, one `adwize.tag_status` event per event, including events **When to Send Data** skips
- `metadata.page_load_id` and `metadata.event_id` on both payloads, so a captured event joins its tag statuses

What differs, because a server container is not a page:

- The collect call is a `POST` with a JSON body. The API key is the `X-Adwize-Api-Key` header, not a query parameter, so there is no URL-length skip.
- `page_load_id` and `event_id` are taken from the incoming event when the web container forwarded them. Otherwise the tag mints one id for that request. It does not keep the id in server template storage — that storage is shared by every visitor on the process, so one stored id would join unrelated requests.
- The data-event request settles the tag (`gtmOnSuccess` / `gtmOnFailure`). The tag-status request only logs a failure.
- The page URL comes from the event's `page_location`. The full snapshot is the parsed event, not the browser dataLayer.

Set this tag to fire **once per event**, and make its trigger cover every event a monitored server tag fires on. A monitored tag that fires on an event this tag does not fire on is missing, not failed.

## Tests

The web template has 19 unit tests. The server template has 14. After importing, open the template editor > **Tests** tab > click **Run Tests**.

## Support

- Documentation: [docs.getadwize.com](https://docs.getadwize.com/integrations/google-tag-manager)
- Email: support@getadwize.com
- Website: [getadwize.com](https://getadwize.com)

## License

Apache 2.0 - see [LICENSE](LICENSE)
