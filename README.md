# Akamai SIEM Integration REST Collector IO
----
## About this Pack

This pack is built as a complete SOURCE + DESTINATION solution (identified by the IO suffix). Data collection and delivery happen entirely within the Pack's context - you can choose how data arrives at a DESTINATION:
*  *Send to Worker Group Routes* (the default): data is sent to the top-level Worker Group Routes.
*  *Default Destination*: data is sent to the Worker Group's [Default Destination](https://docs.cribl.io/stream/destinations-default/). 
*  *In-Pack Destination*: data is sent to one or more Destinations configured within the Pack.

This Collector-based  Pack is designed to collect, process, and output Akamai Security Event data via the [Akamai SIEM Integration REST API](https://techdocs.akamai.com/siem-integration/reference/get-configid). 

The Pack includes OCSF and Splunk output processing:
* Security Event data is mapped to the OCSF [Detection Finding [2004] Class](https://schema.ocsf.io/1.4.0/classes/detection_finding).
* Security Event data is mapped to the Splunk `sourcetype=akamaisiem` for compatibility with the [Akamai SIEM Integration App](https://splunkbase.splunk.com/app/4310)

## Deployment

* Every bundled Source within this pack adds a hidden field: `__packsource`. This field allows for simplified routing based on the Pack source.
* This pack is configured by default to use the Destination *Send to Worker Group Routes*. You *must* add either a Worker Group Route or rely on the Default Destination.
* To explicitly use the Worker Group's *Default Destination*, change the Pack's Routes to *default:default*. The Pack will then route the data to the destination currently set as the Default on the Worker Group.


After installing the Pack, you must perform the following:

### Add Akamai Credentials to Cribl Stream
*Note*: If you use an external secret store, the HMAC function *must* be modified to reference those secrets using the proper names.

* Obtain an Client Token, Client Secret, and Access Token from your Akamai Control Center
* Open the worker group that will contain the Akamai collector.
* Navigate to **Group Settings > Security > Secrets**
* Create 2 Secrets. 
   * The first will have type **API key and secret key**. This secret should be named `akamai_client_credentials`. Populate the Client Token as the `API Key` and the Client Secret as the `Secret Key`.
   * The second will have type **Text**. This secret should be named `akamai_access_token`. Populate the `Value` field with the Akamai Access Token.

### Configure the Collector
* Update the following Pack variables with their correct values for your Akamai environment:

  * `akamai_hostname`: The hostname of your Akamai environment
  * `akamai_config_id`: The unique identifier for each security configuration for which you want to collect data. You can collect from more than one ID simultaneously by separating them via semicolon (`;`)

* Perform a **Run > Preview** of the  `in_akamai_siem_integration` Collector to verify that it works correctly.

* Schedule the Collector and ensure State Tracking is enabled (the correct configuration is already included).

### Configure Output Format
All Security Events travel a **single Route** (`cribl_akamai_siem_events_route` → `cribl_akamai_siem_security_events`). The output format is chosen inside that pipeline, which ends with two mutually exclusive function groups - **Output - OCSF** and **Output - Splunk**. Enable *exactly one* of them and leave the other disabled.

The Pack ships with **OCSF enabled**: `JSON Unroll` and the **Output - OCSF** group are on, and the **Output - Splunk** group is disabled at the group level.

* **OCSF** (Amazon Security Lake) - the shipped default. Needs `JSON Unroll` *and* the `Chain` function in the **Output - OCSF** group. `Chain` hands off to `cribl_akamai_siem_security_events_ocsf`, which reads the pack-native rule names - so do not also enable the Splunk group's `Code` function, which renames them. `Chain` is filtered on `attackData`, so bot-only events bypass the mapping (see below) and reach your Destination as normalized JSON alongside the OCSF findings - add a `Drop` function if you would rather discard them.
* **Splunk**: disable the **Output - OCSF** group, then enable the **Output - Splunk** group and *all three* of its functions - the `Code` function that renames the rule fields to the names the [Akamai SIEM Integration App](https://splunkbase.splunk.com/app/4310) expects (`Rule`→`id`, `ruleAction`→`action`, `ruleMessage`→`message`, and so on), the `Serialize` function, and the `Eval` function that sets `index`, `source` and `sourcetype`. `JSON Unroll` is optional here - see below.
* **Normalized JSON**: disable both Output groups, and `JSON Unroll` too unless you want one event per rule.

#### The shared `JSON Unroll` function
`JSON Unroll` sits *above* the `Serde` extract, and that position is deliberate. Unlike the similarly named `Unroll` function, [`JSON Unroll`](https://docs.cribl.io/stream/json-unroll-function) processes the JSON object string in the `_raw` field - it parses `_raw` internally, replaces the `attackData.rules` array with this event's single rule under the name `ruleData`, and emits one event per rule. Because it reads `_raw` rather than extracted fields, it **must run before the extract**: below `Serde` it would rewrite `_raw` with nothing left to re-extract it, and the OCSF mapping would see no `ruleData` at all.

* **OCSF: required.** Detection Finding treats each matched rule as a separate Finding, and the mapping reads `ruleData.*`, which exists only once the array has been unrolled.
* **Splunk: optional.** Leave it disabled for one event per request, with every rule in `attackData.rules[]` (Splunk extracts those as parallel multivalue fields, so correlating the *n*-th rule across them needs `mvzip`/`mvexpand` at search time). Enable it for one event per rule instead; the Splunk `Code` function renames whichever of the two shapes it finds.

Either way, enabling it multiplies your event count and therefore your licence volume.

#### Bot-only events and OCSF
Not every event Akamai returns is a WAF detection. Bot events arrive with `botData` but no `attackData`, so there is no matched rule to map. Sent through the OCSF mapping they would produce a record claiming `class_uid: 2004` while missing `finding_info`, `severity_id` and `is_alert` - all required by [Detection Finding [2004]](https://schema.ocsf.io/1.4.0/classes/detection_finding) - because 20 of the mapping's expressions dereference `ruleData`, `attackData`, `geo` or `userRiskData`, none of which exist on such an event.

The `Chain` function is therefore filtered on `attackData`. Bot-only events skip the OCSF mapping and continue down the pipeline unchanged, reaching your Destination as normalized JSON. If your Destination is Amazon Security Lake and you want OCSF records *only*, add a `Drop` function with the filter `!attackData` above the Output groups.

#### Decoded rule fields
`cribl_akamai_siem_parse_decode` is deliberately **destination-neutral**: it decodes the base64/URI-encoded `attackData.rule*` fields into an array of objects at `attackData.rules[]`, keyed `Rule`, `ruleAction`, `ruleData`, `ruleMessage`, `ruleSelector`, `ruleTag` and `ruleVersion`. Keys with no value for a given rule are omitted rather than emitted as empty strings.

These are the names `cribl_akamai_siem_security_events_ocsf` reads, so do not rename them in the pre-processing pipeline - rename in the output group instead, the way the Splunk group does. Anything destination-specific (renaming fields, serializing to `_raw`) belongs in `cribl_akamai_siem_security_events`.

### Configure your Destination/Update Pack Routes
To ensure proper data routing, you must make a choice: retain the current setting to use the Default Destination defined by your Worker Group, or define a new Destination directly inside this pack and adjust the pack's route accordingly.

### Commit and Deploy
Once everything is configured, perform a Commit & Deploy to enable data collection.

## Pack Configurable Items 
The following are the in-Pack configurable items - review/update them as needed. 

### Variables

The Pack has the following variables:
* `akamai_siem_default_splunk_index`: Default index for the Splunk output - defaults to `akamai_siem`.

## Advanced Deployments

## Issue: Default Collector is Lagging
If your data collection is lagging, it could be that your collection is too voluminous or a DDOS attack is flooding the logs. This can lead to data loss, inaccurate analysis, and potential security risks. The Akamai API being limited in throughput (approximately 2500-3000 EPS), you can't keep up with the volume during an attack. To solve this you must skip ahead whenever the data collection is lagging.

* Configure (like in the section above) and use the `in_akamai_siem_integration_offset_advanced` REST Collector. This collection is mono-threaded, just like the standard one, and still uses offset (which is the best way to ensure every event is collected exactly once). However, state tracking will detect when the collection is behind and will skip ahead when it lags. Jumping ahead will allow you to skip a burst of logs and avoid lagging. The drawback is that your data collection will be incomplete. Completeness VS Recency : the Akamai API only allows you to pick one. Note: this data collection is *monothreaded* as the offset is sent last by Akamai, so it's limited to 2000-4000eps maximum.

* Monitor your lag to verify that most of the time your lag is OK (below 90s typically). There are multiple ways to achieve this; the easiest is just to check your state in your `input->manage state` : it will show you the latest event time and if it's lagging. You can also gather collection stats in Cribl Lake and monitor them in Cribl search with a custom pipeline (see *Monitor the collection lag and duplicates* section for details).

At this point, if your collection is still lagging, your baseline data volume is too high for a single-threaded collection so you'll need to use the  multithreaded Collector from the next section.

## Issue: Require High Volume Collection
If your Akamai Id sends more than 3000 eps, you might want to use the multithreaded option. This method uses a to/from Collector and splits every minute into 4 jobs of 15 seconds. This collection can ingest a lot more events (8-15000 EPS with 4 threads, in theory). Since it doesn't use offset, it cannot guarantee that you receive every log (for example, if the job fails).

Note that this configuration can only handle *one* Id, which you set directly in the url.

* Ensure the `akamai_config_id` variable contains *only one* ID.
* Disable the othe Collectors and schedule the `in_akamai_siem_integration_multithreaded` REST Collector 

Note that this collection **will** generate duplicate data. Akamai will randomly send you data outside the date boundary, data that you already collected a minute ago. To solve this issue, perform the following:

* Enable the `cribl_akamai_siem_events_monitor_lag` Route and enable the `Drop` function in the `cribl_akamai_siem_monitor_rest_collection_lag` pipeline. This will drop events where the `eventtime` is earlier than the time boundary set.
* Optional but recommened : follow the steps in the *Monitor the collection lag and duplicates* section.

If data collection is still lagging constantly, add more threads. To do so, check the discovery itemList to split every minute : 0,15,30,45 means 4 thread of 15 seconds. Having 6 threads can be achieved by having the discovery itemlist set to : 0,10,20,30,40,50 and changing the to and from request parameters to adjust for the 10-second difference instead of 15. But refrain from changing the current config of 4 threads in a minutes, this is a sweet spot.

## Monitor the collection lag and duplicates
The Pack includes functionality to monitor the data ingestion lag via the `cribl_akamai_siem_monitor_rest_collection_lag` pipeline. It uses `now` as ingestion time, and event time to calculate the lag, effectively changing the _time of the event to visually see the event by the time it came in. It also creates a SHA field from the time and the raw log to detect duplicates without sending sensitive information. 

* You will need a destination configured that allows searching the data e.g. Cribl Lake/Search.
* Enable the `cribl_akamai_siem_events_monitor_lag` Route and add a Destination.  
* Monitor the lag (in Cribl Search or similar) with this search : `dataset="mydataset" input="collection:xxx" |timestats span=5s count(), max(lag), min(lag), percentile(lag,90)`
* Monitor duplicates with this search `dataset="xxx"  | summarize count() by sha | where count_ > 1`

## Upgrades

Upgrading certain Cribl Packs using the same Pack ID can have unintended consequences. See [Upgrading an Existing Pack](https://docs.cribl.io/stream/packs#upgrading) for details.

## Release Notes

### Version 2.1.1

`cribl_akamai_siem_security_events`:
* Added an optional `Code` function to the Splunk group that renames the rule fields to the Splunk TA names.it handles `JSON Unroll` *replaces* the array rather than duplicating it, so there was never a double-ship to prevent; the function now simply renames whichever shape is present.
* The OCSF `Chain` function is now filtered on `attackData` instead of `true`. Bot-only events have no matched WAF rule, so 20 of the mapping's expressions dereferenced a `ruleData` / `attackData` / `geo` / `userRiskData` object that did not exist, and the result was a `class_uid: 2004` record with no `finding_info`, `severity_id` or `is_alert` - all required by the OCSF class. Those events now bypass the mapping and pass through as normalized JSON (see *Bot-only events and OCSF*).
*  The Splunk `index` now falls back to `akamai_siem` if the `akamai_siem_default_splunk_index` variable is missing.


`cribl_akamai_siem_parse_decode`:
* Fixed the `attackData.rule*` decoding. A trailing `;` in an Akamai rule field no longer produces a spurious empty rule; rule slots carrying no values are pruned instead of surfacing as empty objects; and empty per-rule values are omitted instead of being emitted as empty strings, which `Numerify` then turned into `0` (visible as `ruleSelector: 0` in the old decoded sample).
* Request and response headers are now split on `/\r?\n/`. Akamai encodes the separator as `%0d%0a`, so every header element previously carried a trailing `\r`.
* `Numerify` now ignores `httpMessage.requestId`. All-digit Akamai request IDs were being coerced to numbers, losing leading zeros and - past 2^53 - precision.
* The `Code` and `Eval` functions are now scoped with `attackData` / `httpMessage` filters instead of `true`. Events with no `attackData` - bot-only events, or any event whose `_raw` failed to parse in Preview - previously errored with `Cannot set properties of undefined`. They now pass through untouched.
* Rule field matching is case-insensitive and coerces non-string values, so a change in Akamai's casing or types no longer throws.


Samples:
* Added `cribl_akamai_security_event_edge_cases` - collector output as the event breaker delivers it, covering a security event with a trailing `;` in `ruleSelectors`, a bot-only event, and an offset state object.
* Refreshed `cribl_akamai_security_event_decoded` to match the corrected decoding.

### Version 2.0.0
* Updated Route Destinations to "Send to Worker Group Routes". See above for details.

### Version 1.1.0
- Collector configuration is now done via variables

### Version 1.0.0
- Initial release

## Contributing to the Pack

To contribute to the Pack, please connect with us on [Cribl Community Slack](https://cribl-community.slack.com/). You can suggest new features or offer to collaborate.

## License
This Pack uses the following license: [Apache 2.0](https://github.com/criblio/appscope/blob/master/LICENSE).

## Acknowledgments
Thanks to Sidd Shah - sshah@cribl.io and Simon Duchene - sduchene@cribl.io for creating the Collector configurations and initial pipelines!
