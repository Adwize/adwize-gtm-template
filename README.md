# Adwize - Data Collection Monitoring (GTM Template)

Monitor data quality at the source. This Google Tag Manager template captures dataLayer events — event name, optional e-commerce/items, optional full dataLayer snapshot, and container metadata — and forwards them to the [Adwize](https://getadwize.com) API for quality monitoring, anomaly detection, and alerting.

Optionally, it also reports whether the GTM tags you care about (GA4, Ads, etc.) fired and how they ran — see [Tag monitoring](#tag-monitoring-optional). The two answer different questions: dataLayer capture checks that the site sends the right data into GTM; tag monitoring checks that GTM fires your key tags without errors.

## What it does

- Captures dataLayer events (e.g. `page_view`, `purchase`) as they occur
- Collects e-commerce data, transaction details, and container metadata
- Sends a lightweight pixel request (`GET`) to the Adwize API
- Supports filtering by all events, e-commerce only, or a custom event list
- Optionally reports the status (`success`, `failure`, `exception`, `timeout`) and execution time of tags you mark with tag metadata

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
| **Report status of monitored tags** | Send the status of tags with metadata `monitor` = `true` after each event this tag fires on (Tag Monitoring, default: off) |

## Tag monitoring (optional)

Uses GTM's [`addEventCallback`](https://developers.google.com/tag-platform/tag-manager/templates/monitoring) API. At the end of each event this tag fires on, it reads the tags that fired for that event and sends one `adwize.tag_status` event listing the monitored ones.

1. On each tag to monitor: **Advanced Settings** > **Additional Tag Metadata** > **Add Metadata**, key `monitor`, value `true`. Check **Include Tag Name** with key `name` to report tag names.
2. In the Adwize tag: **Tag Monitoring** > check **Report status of monitored tags**.
3. Make the Adwize tag's trigger cover **every event a monitored tag fires on**. A monitored tag that fires on an event the Adwize tag does not fire on is not reported — it is missing, not failed. Watch for tags triggered by Page View (`gtm.js`), DOM Ready (`gtm.dom`), clicks (`gtm.click`), form submits (`gtm.formSubmit`) or Initialization (`gtm.init`), not only custom events. A Custom Event trigger with regex matching, e.g. `purchase|gtm\.js|gtm\.formSubmit`, covers several at once.
4. Set the Adwize tag to fire **Once per event** (Advanced Settings > Tag firing options).

Tag monitoring ignores **When to Send Data**: the trigger alone decides which events are monitored. Both payloads carry `metadata.page_load_id` (random, shared by every event of a page load) and `metadata.event_id` (GTM's `gtm.uniqueEventId`, which restarts on each page load). Together they join a dataLayer event with its tag statuses.

Each monitored tag is reported as:

```json
{"id": "12", "name": "GA4 - purchase", "status": "success", "execution_time": 120}
```

`success` means the tag reported completion, not that the vendor received or accepted the hit.

## Payload size (GET URL limits)

Events are sent as a `GET` pixel with the JSON payload in the query string. Large payloads — especially with **Capture full dataLayer** enabled, or large e-commerce `items` arrays — can exceed typical browser/proxy URL length limits (~2k–8k characters). When the collect URL exceeds ~1800 characters, the tag skips that `sendPixel` and always logs to the console (`[Adwize] Payload too large…`), even when debug logging is off. Prefer e-commerce-only or a custom event list when possible, and leave full dataLayer capture off unless you need it for debugging. With tag monitoring on, monitor only the tags that matter: each one adds about 100 characters.

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
    → Adwize tag keeps the tags with metadata monitor = true
      → sendPixel GET request with an adwize.tag_status event
```

## Server-side template

`template-server-side.tpl` is the same monitor for a **server** container. Import it there, not into the web container.

What matches the web template:

- The same trigger modes (all events, e-commerce only, custom list)
- Optional tag monitoring: metadata `monitor` = `true`, one `adwize.tag_status` event per event, including events **When to Send Data** skips
- Tag status carries `summary` (every tag that fired), `tags` (monitored tags with all their metadata, failures first) and `unmonitored_issues` (unmarked tags that did not succeed)
- `metadata.page_load_id` and `metadata.event_id` on both payloads, so a captured event joins its tag statuses

What differs, because a server container is not a page:

- The collect call is a `POST` with a JSON body. The API key is the `X-Adwize-Api-Key` header, not a query parameter, so there is no URL-length limit and tag status is never split.
- `page_load_id` and `event_id` are read from the incoming event, under the keys set in **Join IDs** (default `page_load_id` and `event_id`). Otherwise the tag mints one id for that request. It does not keep the id in server template storage — that storage is shared by every visitor on the process, so one stored id would join unrelated requests.
- **Check `event_id` in your setup.** Many server setups (Stape Data Tag, Meta CAPI deduplication) use `event_id` for the Meta deduplication id, which is not GTM's `gtm.uniqueEventId` and will not join with web payloads. Forward `gtm.uniqueEventId` as an event parameter under another key (for example `adwize_event_id`) and enter that key in **Join IDs**.
- `metadata.consent` comes from the GA4 hit: `gcs` (`x-ga-gcs`, decoded into `ad_storage` and `analytics_storage`) and `gcd` (`x-ga-gcd`, passed raw). It is empty when the hit carries neither, for example a Data Tag hit that does not forward consent.
- `metadata.client_name` names the client that claimed the request (for example `GA4` or `Data Client`).
- The full snapshot is the parsed event, not the browser dataLayer. It never includes `user_data`, `ip_override`, `user_agent` or `x-fb-ud-*` keys; add more under **Also exclude from snapshot**.
- The data-event request settles the tag (`gtmOnSuccess` / `gtmOnFailure`). The tag-status request only logs a failure.
- The page URL comes from the event's `page_location`.
- Server tags usually report `success` or `failure` from the vendor's HTTP response (Meta CAPI, GA4, Google Ads), so a server status is closer to "the platform accepted it" than a web status. It still depends on each tag's template.

Set this tag to fire **once per event**, and make its trigger cover every event a monitored server tag fires on. A monitored tag that fires on an event this tag does not fire on is missing, not failed.

## Tests

The web template has 13 unit tests. The server template has 20. After importing, open the template editor > **Tests** tab > click **Run Tests**.

## Support

- Documentation: [docs.getadwize.com](https://docs.getadwize.com/integrations/google-tag-manager)
- Email: support@getadwize.com
- Website: [getadwize.com](https://getadwize.com)

## License

Apache 2.0 - see [LICENSE](LICENSE)
