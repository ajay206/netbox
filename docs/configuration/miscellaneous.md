# Miscellaneous Parameters

## ADMINS

NetBox will email details about critical errors to the administrators listed here. This should be a list of (name, email) tuples. For example:

```python
ADMINS = [
    ['Hank Hill', 'hhill@example.com'],
    ['Dale Gribble', 'dgribble@example.com'],
]
```

---

## BANNER_BOTTOM

!!! tip "Dynamic Configuration Parameter"

Sets content for the bottom banner in the user interface.

---

## BANNER_LOGIN

!!! tip "Dynamic Configuration Parameter"

This defines custom content to be displayed on the login page above the login form. HTML is allowed.

---

## BANNER_MAINTENANCE

!!! tip "Dynamic Configuration Parameter"

This adds a banner to the top of every page when maintenance mode is enabled. HTML is allowed.

---

## BANNER_TOP

!!! tip "Dynamic Configuration Parameter"

Sets content for the top banner in the user interface.

!!! tip
    If you'd like the top and bottom banners to match, set the following:

    ```python
    BANNER_TOP = 'Your banner text'
    BANNER_BOTTOM = BANNER_TOP
    ```

---

## COPILOT_ENABLED

!!! tip "Dynamic Configuration Parameter"

Default: `True`

Enables or disables the [NetBox Copilot](https://netboxlabs.com/docs/copilot/) agent globally. When enabled, users can opt to toggle the agent individually.

---

## CENSUS_REPORTING_ENABLED

Default: `True`

Enables anonymous census reporting. To opt out of census reporting, set this to `False`.

This data enables the project maintainers to estimate how many NetBox deployments exist and track the adoption of new versions over time. Census reporting effects a single HTTP request each time a worker starts. The only data reported by this function are the NetBox version, Python version, and a pseudorandom unique identifier.

---

## CHANGELOG_RETAIN_CREATE_LAST_UPDATE

!!! tip "Dynamic Configuration Parameter"

Default: `False`

When pruning expired changelog entries (per `CHANGELOG_RETENTION`), retain each non-deleted object's original `create`
change record and its most recent `update` change record. If an object has a `delete` change record, its changelog
entries are pruned normally according to `CHANGELOG_RETENTION`.

!!! note
    For objects without a `delete` change record, the original `create` record and most recent `update` record are
    exempt from pruning. All other changelog records (including intermediate `update` records and all `delete` records)
    remain subject to pruning per `CHANGELOG_RETENTION`.

---

## CHANGELOG_RETENTION

!!! tip "Dynamic Configuration Parameter"

Default: `90`

The number of days to retain logged changes (object creations, updates, and deletions). Set this to `0` to retain
changes in the database indefinitely.

!!! warning
    If enabling indefinite changelog retention, it is recommended to periodically delete old entries. Otherwise, the database may eventually exceed capacity.

---

## CHANGELOG_SKIP_EMPTY_CHANGES

Default: `True`

If enabled, a change log record will not be created when an object is updated without any changes to its existing field values.

!!! note
    The object's `last_updated` field will always reflect the time of the most recent update, regardless of this parameter.

---

## DATA_UPLOAD_MAX_MEMORY_SIZE

Default: `2621440` (2.5 MB)

The maximum size (in bytes) of an incoming HTTP request (i.e. `GET` or `POST` data). Requests which exceed this size will raise a `RequestDataTooBig` exception.

---

## STREAMING_EXPORTS

Default: `False`

When set to `True`, CSV bulk exports are returned as a streaming HTTP response, emitting rows to the client as they are rendered rather than buffering the entire dataset in memory first. This can significantly reduce memory usage and time-to-first-byte for very large exports.

Because streaming responses do not have a `Content-Length` header and defer errors until after the response has begun, this behavior is opt-in.

---

## ENFORCE_GLOBAL_UNIQUE

!!! tip "Dynamic Configuration Parameter"

Default: `True`

By default, NetBox will prevent the creation of duplicate prefixes and IP addresses in the global table (that is, those which are not assigned to any VRF). This validation can be disabled by setting `ENFORCE_GLOBAL_UNIQUE` to `False`.

---

## EVENTS_PIPELINE

Default: `['extras.events.process_event_queue',]`

A list of dotted Python paths to callable handlers. After a request or job finishes successfully, NetBox flushes the in-memory event queue and calls each handler in list order, passing the queued events. If `extras.events.process_event_queue` is omitted from this list, NetBox inserts it at the start of the pipeline so [event rules](../features/event-rules.md) still run. Plugin `events_pipeline` entries are appended.

Handlers are invoked only when the request or job completes without raising. Events queued during a failed request or job are discarded.

### Handler signature

Each handler must accept a single argument: the list of queued events.

```python
def handle_events(events):
    for event in events:
        ...
```

Return values are ignored. Handlers run in the same process as the originating request or job, after the wrapped view or job body has returned.

### Event objects

Each item is an `EventContext` (`extras.events.EventContext`), a dictionary-compatible wrapper. Object create/update/delete events include these keys:

| Key | Description |
|-----|-------------|
| `object_type` | `ObjectType` for the changed model |
| `object_id` | Primary key of the instance |
| `object` | The model instance. On coalesced updates this remains the first instance that was queued; later saves refresh only the serialization source used for `data`. |
| `event_type` | Event type (for example create, update, or delete) |
| `snapshots` | Dict with `prechange` and `postchange` snapshots. A later event for the same object updates `postchange`. For a delete, `postchange` is `None`. |
| `object_change_ids` | List of `ObjectChange` IDs recorded for this object during the request. Repeated events for the same object append distinct IDs. |
| `request` | The current request |
| `user` | `request.user` |
| `data` | Serialized object payload (see below) |

Repeated events for the same object (`<app>.<model>:<pk>`) are combined into one queue entry. A later delete promotes a prior create/update to a delete event.

Objects that do not support event rules are not queued.

### Serialization of `data`

`data` is produced by the model's REST API serializer (`request` context is `None`). It is **not** the same as the retained `object` instance.

- For create and update events, `data` is serialized on first access. If the same object is queued again, any cached `data` is dropped and the next access serializes the latest queued instance.
- For delete events, `data` is serialized immediately when the event is queued, before the row (and related data) may become inaccessible.

### Passing data to background workers

The default handler (`process_event_queue`) evaluates event rules and enqueues webhook and script work. Those actions pass `data` and `snapshots` into RQ. When a request is present, they copy it with `copy_safe_request(request, include_files=False)` so uploaded files are not pickled (which can fail for some file types). Custom handlers that enqueue jobs should do the same if they need request context.

Job start/complete event rules are processed directly from job signals and do not go through `EVENTS_PIPELINE`. Those events may omit `request` and use the job's `data` field as the payload.

### Exceptions

If a handler path cannot be imported, NetBox logs an `ImportError` and continues with the next handler. Any other exception raised by a handler is not caught: it propagates and can fail the request or job after the originating database changes have already been saved.

### Example

```python
# configuration.py
EVENTS_PIPELINE = [
    'extras.events.process_event_queue',
    'my_plugin.events.log_object_events',
]
```

```python
# my_plugin/events.py
import logging

logger = logging.getLogger('netbox.events_processor')


def log_object_events(events):
    for event in events:
        logger.info(
            'event %s for %s pk=%s (changes=%s)',
            event['event_type'],
            event['object_type'],
            event['object_id'],
            event['object_change_ids'],
        )
        # Accessing event['data'] serializes the object (already done for deletes).
        payload = event['data']
        snapshots = event['snapshots']
```

---

## FILE_UPLOAD_MAX_MEMORY_SIZE

Default: `2621440` (2.5 MB)

The maximum amount (in bytes) of uploaded data that will be held in memory before being written to the filesystem. Changing this setting can be useful for example to be able to upload files bigger than 2.5MB to custom scripts for processing.

---

## JOB_RETENTION

!!! tip "Dynamic Configuration Parameter"

Default: `90`

The number of days to retain job results (scripts and reports). Set this to `0` to retain job results in the database indefinitely.

!!! warning
    If enabling indefinite job results retention, it is recommended to periodically delete old entries. Otherwise, the database may eventually exceed capacity.

---

## MAINTENANCE_MODE

!!! tip "Dynamic Configuration Parameter"

Default: `False`

Setting this to `True` will display a "maintenance mode" banner at the top of every page. Additionally, NetBox will no longer update a user's "last active" time upon login. This is to allow new logins when the database is in a read-only state. Recording of login times will resume when maintenance mode is disabled.

---

## MAPS_URL

!!! tip "Dynamic Configuration Parameter"

Default: `https://maps.google.com/?q=` (Google Maps)

This specifies the URL to use when presenting a map of a physical location by street address or GPS coordinates. Set this to `None` to disable the "map it" button within the UI.

**For street addresses**, the URL must accept a free-form address string appended directly to it.

**For GPS coordinates**, two formats are supported:

* **Simple prefix** (default behavior): The latitude and longitude are appended as a comma-separated pair. For example, `https://maps.google.com/?q=` produces `https://maps.google.com/?q=48.858,2.294`.
* **Coordinate placeholders**: Include `{lat}` and/or `{lon}` anywhere in the URL. Only these two literal placeholders are supported. For example:

```
MAPS_URL = "https://www.openstreetmap.org/?mlat={lat}&mlon={lon}#map=16/{lat}/{lon}"
```

!!! note
    When `MAPS_URL` contains `{lat}` or `{lon}` placeholders, the "map it" button will only appear on pages with GPS coordinates — address-based map links will be suppressed, since the coordinate-format URL cannot be used with a plain address string.

---

## MAX_PAGE_SIZE

!!! tip "Dynamic Configuration Parameter"

Default: `1000`

Defines the maximum number of objects that may be returned in a single page across the web UI, REST API, and GraphQL API. Setting `MAX_PAGE_SIZE` to `0` or `None` removes the limit.

See the [REST API](../integrations/rest-api.md#pagination) and [GraphQL API](../integrations/graphql-api.md#pagination) pagination documentation for details.

---

## METRICS_ENABLED

Default: `False`

Toggle the availability Prometheus-compatible metrics at `/metrics`. See the [Prometheus Metrics](../integrations/prometheus-metrics.md) documentation for more details.

---

## PREFER_IPV4

!!! tip "Dynamic Configuration Parameter"

Default: `False`

When determining the primary IP address for a device, IPv6 is preferred over IPv4 by default. Set this to `True` to prefer IPv4 instead.

---

## QUEUE_MAPPINGS

Allows changing which queues are used internally for background tasks.

```python
QUEUE_MAPPINGS = {
    'webhook': 'low',
    'report': 'high',
    'script': 'high',
}
```

If no queue is defined the queue named `default` will be used.

---

## RELEASE_CHECK_URL

Default: `None` (disabled)

This parameter defines the URL of the repository that will be checked for new NetBox releases. When a new release is detected, a message will be displayed to administrative users on the home page. This can be set to the official repository (`'https://api.github.com/repos/netbox-community/netbox/releases'`) or a custom fork. Set this to `None` to disable automatic update checks.

!!! note
    The URL provided **must** be compatible with the [GitHub REST API](https://docs.github.com/en/rest).

---

## RQ

Default: `{}` (Empty)

This is a wrapper for passing global configuration parameters to [Django RQ](https://github.com/rq/django-rq) to customize its behavior. It is employed within NetBox primarily to alter conditions during testing.

---

## RQ_DEFAULT_TIMEOUT

Default: `300`

The maximum execution time of a background task (such as running a custom script), in seconds. This may also be expressed as a duration string such as `1h` or `30m`, which NetBox normalizes to seconds when comparing it against webhook timeouts. Set this to `-1` to disable the job timeout entirely.

!!! note
    A value of zero (or `None`) does not disable the timeout: RQ falls back to its own default of 180 seconds, and NetBox validates webhook timeouts against that value accordingly.

---

## RQ_RETRY_INTERVAL

Default: `60`

This parameter controls how frequently a failed job is retried, up to the maximum number of times specified by `RQ_RETRY_MAX`. This must be either an integer specifying the number of seconds to wait between successive attempts, or a list of such values. For example, `[60, 300, 3600]` will retry the task after 1 minute, 5 minutes, and 1 hour.

---

## RQ_RETRY_MAX

Default: `0` (retries disabled)

The maximum number of times a background task will be retried before being marked as failed.

## DISK_BASE_UNIT

Default: `1000`

The base unit for disk sizes. Set this to `1024` to use binary prefixes (MiB, GiB, etc.) instead of decimal prefixes (MB, GB, etc.).

## RAM_BASE_UNIT

Default: `1000`

The base unit for RAM sizes. Set this to `1024` to use binary prefixes (MiB, GiB, etc.) instead of decimal prefixes (MB, GB, etc.).

---

## WEBHOOK_DEFAULT_TIMEOUT

Default: `60`

The default maximum time (in seconds) to wait for a response when sending a webhook. This value is used for any webhook which does not define its own timeout. Keeping this below [`RQ_DEFAULT_TIMEOUT`](#rq_default_timeout) gives an unresponsive receiver a chance to be cut off by the request timeout rather than by termination of the background job.

This value must be an integer between 1 and 3600, and must be less than `RQ_DEFAULT_TIMEOUT`; NetBox will refuse to start otherwise. The same upper bound is enforced on the per-webhook [timeout](../models/extras/webhook.md#timeout) field.

!!! warning "Upgrading"
    If you have lowered `RQ_DEFAULT_TIMEOUT` to 60 seconds or less and have not set `WEBHOOK_DEFAULT_TIMEOUT`, NetBox will not start until you set `WEBHOOK_DEFAULT_TIMEOUT` to a value below your job timeout.

!!! note
    The timeout is applied separately to establishing the connection and to waiting for data, rather than to the request as a whole. A receiver which responds slowly but continuously can therefore keep a request open for longer than the configured value. `RQ_DEFAULT_TIMEOUT` remains the ultimate upper bound on how long a webhook job can occupy a worker.
