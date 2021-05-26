# OpenAVT-Docs
Documentation repository for the OpenAVT project.

## 1. Introduction

The Open Audio-Video Telemetry is a multiplatform set of tools for performance monitoring in multimedia applications. The objectives are similar to those of the OpenTelemetry project, but specifically for sensing data from audio and video players.

## 2. Platforms

Currently OpenAVT supports the following platforms:

- [Android](https://github.com/asllop/OpenAVT-Android)
- [iOS/tvOS](https://github.com/asllop/OpenAVT-iOS)

Visit the repos and checkout the README for specific installation and usage instructions.

## 3. Data Model

The Data Model describes all the data an instrument could generate and the meaning of each piece of information.

#### 3.1 The telemetry dilemma: Events or Metrics?

First let's define what are Events and Metrics in the context of OpenAVT. Both are time series data, but there are some differences:

An **Event** is an heterogeneous structure of data. It contains a list of key-value pairs, where the key is always a string but the value could be of any type: integer, float, string or boolean. Two events of the same type (or how they are called in OpenAVT, same **Action**), may contain different combinations of key-value pairs (**Attributes**, as they are called in OpenAVT).

A **Metric** on the other side is homogeneous, there is one single value per metric and it's always numeric (integer or float). Two metrics of the same type have always the same kind of data.

Choosing between events and metrics depends on many factors: the kind of calculations we want to do, the amount of data we can store, how often we are going to update our KPIs, where we are going to store our data (the choosen backend), etc.

In general events offer more flexibility to calculate important indicators. We have almost "raw" information, so if we want to make some KPI calculations today, but our needs changes over time, it's possible to update the queries on the recorded data without having to change the instrument code. The main disadvantage of events is that they consume lots of space, so our database will grow rapidly.

Metrics are small and doesn't get too much space on a database. Queries over metrics are also much faster to process. But the information we store is very specific and, in general, with metrics we have to hardcode the KPIs we want to generate in the instrument side. If these needs change, we will face the problem of updating the instrument code. In OpenAVT this jobs is done in the **Metricalc**.

Also, some backends can work better with (or only support) one kind of data. For example, Graphite only offers support for metrics (that's actually not true, it supports events, but they are so limited that doesn't fit the needs of OpenAVT Events).

#### 3.2 Events

Events indicate that something happened in the tracker lifecycle and player workflow. Each event has a type, that in OpenAVT is called Action. The following is an exhaustive list of the available actions. Not all actions are used in all contexts and some players doesn't support certain actions.

| Action | Description |
| ------ | ----------- |
| `TRACKER_INIT` | A tracker has been initialized. |
| `PLAYER_SET` | A player instance has been passed to the tracker. |
| `PLAYER_READY` | The player instance is ready to start generating events. |
| `MEDIA_REQUEST` | An audio/video stream has been requested, usually by the user (tapping a play button, choosing a video from a list, or similar). This actions is meant to be app driven, not fired by the player. |
| `PREPARE_ITEM` | The player is preparing an item to be loaded/played. Not all players support this action. |
| `MANIFEST_LOAD` | The manifest is being loaed. Not all players support this action. |
| `STREAM_LOAD` | An audio/video stream is being loaded. |
| `START` | Stream has started, first frame shown. |
| `BUFFER_BEGIN` | Player started buffering. |
| `BUFFER_FINISH` | Player ended buffering. |
| `SEEK_BEGIN` | Started seeking. |
| `SEEK_FINISH` | Ended seeking. |
| `PAUSE_BEGIN` | Stream paused. |
| `PAUSE_FINISH` | Stream resumed. |
| `FORWARD_BEGIN` | Fast forward begin. Not all players support this action. |
| `FORWARD_FINISH` | Fast forward finish. Not all players support this action. |
| `REWIND_BEGIN` | Fast rewind begin. Not all players support this action. |
| `REWING_FINISH` | Fast rewind finish. Not all players support this action. |
| `QUALITY_CHANGE_UP` | Stream quality (resolution) increased. |
| `QUALITY_CHANGE_DOWN` | Stream quality (resolution) degraded. |
| `STOP` | Playback has been stopped. Usually called when the user closes the player or selects another content before ending the current, and so it's app driven. |
| `END` | Stream reached the end. |
| `NEXT` | Next stream in a playlist is going to be loaded. Not all players support this action. |
| `ERROR` | An error happened. |
| `PING` | Sent every 30 seconds. It's started after a `START` and stopped after an `END`, `STOP`, `NEXT` or unrecoverable `ERROR`. |
| `AD_BREAK_BEGIN` | An ad break (block) has started. An ad break may contain multiple ads. |
| `AD_BREAK_FINISH` | Ad break finished. |
| `AD_BEGIN` | Ad started, first framr shown. |
| `AD_FINISH` | Ad ended. |
| `AD_PAUSE_BEGIN` | Ad paused. |
| `AD_PAUSE_FINISH` | Ad resumed. |
| `AD_BUFFER_BEGIN` | Ad started buffering. |
| `AD_BUFFER_FINISH` | Ad ended buffering. |
| `AD_SKIP` | Ad skipped. |
| `AD_CLICK` | User tapped on the ad. |
| `AD_FIRST_QUARTILE` | Ad reched the first quartile. |
| `AD_SECOND_QUARTILE` | Ad reched the second quartile. |
| `AD_THIRD_QUARTILE` | Ad reched the third quartile. |
| `AD_ERROR` | An error happened during ad playback. |

The common workflow of events for most playbacks is as follows:

1. `TRACKER_INIT` when the tracker is ready.
2. `PLAYER_SET` when the player instance is passed to the tracker.
3. `PLAYER_READY` when all listeners have been set and the player is ready to generate events.
4. `STREAM_LOAD` when a stream starts loading.
5. `START` when the stream ends loading and starts playing.
6. After it, can happen any number of the following blocks: `BUFFER_BEGIN`/`BUFFER_FINISH`, `PAUSE_BEGIN`/`PAUSE_FINISH` or `SEEK_BEGIN`/`SEEK_FINISH`. Also can hapen quality changes (`QUALITY_CHANGE_UP`, `QUALITY_CHANGE_DOWN`).
7. Finally an `END` or `STOP` will happen when the stream ends or is stopped by the user.

An `ERROR` can happen at any time during the player lifecycle. An error usually implies the end of the playback, so use to be followed by an `END`.

An ad block can happen at any time during the content playback. The workflow of ads is as follows:

1. `AD_BREAK_BEGIN` when the block starts.
2. `AD_BEGIN` when an ad starts.
3. `AD_FIRST_QUARTILE`, `AD_SECOND_QUARTILE` and `AD_THIRD_QUARTILE` when the corresponding quartiles are reached.
4. `AD_FINISH` when the ad reaches the end.
5. After it, if there are more ads in the block, the workflow start again on 2.
6. When all ads are played, an `AD_BREAK_FINISH` is sent.

An `AD_SKIP` when the user skips the ad can happen at any time.
An `AD_CLICK` when the user taps the ad can happen at any time.
An `AD_ERROR` can happen at any time and is usually followed by `AD_FINISH` and also commonly by an `AD_BREAK_FINISH`.

#### 3.3 Attributes

As we already said, en event is composed out of an action and a list of attributes. Here we present the list of attributes generated by OpenAVT. Again, like in events, not all attributes are always present in all trackers. Some information may not be available in certain players.

Note: Times are in milliseconds.

| Attribute | Data Type | Description |
| --------- | :-------: | ----------- |
| `TRACKER_TARGET` | String | Which player or event source is the tracker attached to. |
| `STREAM_ID` | String | UUID that identifies a specific stream playback. It's generated when `STREAM_LOAD` happens and is different every time, even if the stream source is the same. |
| `PLAYBACK_ID` | String | Is similar to `STREAM_ID`, but it's generated every time there is an `STREAM_LOAD` or a `MEDIA_REQUEST` and recalculated when there is an `END`, `STOP` or `NEXT`. |
| `SENDER_ID` | String | Identifier of the tracker that generated the event. It identifies a tracker within an instrument. |
| `COUNT_ERRORS` | Integer | Number of `ERROR` events sent. |
| `COUNT_STARTS` | Integer | Number of `START` events sent. |
| `ACCUM_PAUSE_TIME` | Integer | Accumulated time during pause blocks (`PAUSE_BEGIN` / `PAUSE_FINISH`). |
| `ACCUM_BUFFER_TIME` | Integer | Accumulated time during buffering blocks (`BUFFER_BEGIN` / `BUFFER_FINISH`). |
| `ACCUM_SEEK_TIME` | Integer | Accumulated time during seeking blocks (`SEEK_BEGIN` / `SEEK_FINISH`). |
| `ACCUM_PLAY_TIME` | Integer | Accumulated time during playback, not counting ad, pause, buffering or seeking blocks. |
| `DELTA_PLAY_TIME` | Integer | Time playing since last event. |
| `IN_PAUSE_BLOCK` | Boolean | Currently within a pause block. |
| `IN_SEEK_BLOCK` | Boolean | Currently within a seeking block. |
| `IN_BUFFER_BLOCK` | Boolean | Currently within a buffering block. |
| `IN_PLAYBACK_BLOCK` | Boolean | Curently playing. |
| `ERROR_DESCRIPTION` | String | When an `ERROR` happens, description of that error. Usually the error message taken from the error object. |
| `POSITION` | Integer | Current playback position in milliseconds. |
| `DURATION` | Integer | Stream duration in milliseconds. |
| `RESOLUTION_HEIGHT` | Integer | Stream vertical resolution. |
| `RESOLUTION_WIDTH` | Integer | Stream horizontal resolution. |
| `IS_MUTED` | Boolean | Playback muted. |
| `VOLUME` | Integer | Volume, from 0 to 100. |
| `FPS` | Float | Frames per second. |
| `SOURCE` | String | Stream source, usually a URL. |
| `BITRATE` | Integer | Stream bitrate in bits per second. |
| `LANGUAGE` | String | Stream language. |
| `SUBTITLES` | String | Stream subtitles language. |
| `TITLE` | String | Stream title. |
| `IS_ADS_TRACKER` | Boolean | It is an ads tracker or not. |
| `COUNT_ADS` | Integer | MNumber of ads played. |
| `IN_AD_BREAK_BLOCK` | Boolean | Currently within an ad break block. |
| `IN_AD_BLOCK` | Boolean | Currently playing an ad. |
| `AD_POSITION` | Integer | Ad playback position in milliseconds. |
| `AD_DURATION` | Integer | Ad duration in milliseconds. |
| `AD_BUFFERED_TIME` | Integr | Amount of Ad stream buffered time. |
| `AD_VOLUME` | Integer | Ad volume, from 0 to 100. |
| `AD_ROLL` | String | Ad position within the main stream (pre, mid or post). |
| `AD_DESCRIPTION` | String | Ad description. |
| `AD_ID` | String | Ad identifier. |
| `AD_TITLE` | String | Ad title. |
| `AD_ADVERTISER_NAME` | String | Advertiser name. |
| `AD_CREATIVE_ID` | String | Creative identifier. |
| `AD_BITRATE` | Integer | Ad bitrate in bits per second. |
| `AD_RESOLUTION_HEIGHT` | Integer | Ad vertical resolution. |
| `AD_RESOLUTION_WIDTH` | Integer | Ad horizontal resolution. |
| `AD_SYSTEM` | String | Ad system. |

The is also a family of attributes called time-since attributes. They indicate the time elapsed since a certain event was sent. For example `timeSinceTrackerInit` is the time since `TRACKER_INIT` was sent. Every event has a time-since attribute associated.

#### 3.4 Metrics

Metrics represent a numerical value that varies over time. OpenAVT supports two types of metrics: Gauge and Counter.

**Counter** measures the number of occurrences.<br>
**Gauge** represents a value that can increase or decrease.

For OpenAVT these types are purely semantical, they have no implications in how the system behaves. But some backends support this types and it has implications in how metrics are stored and queried.

| Metric | Type | Description |
| ------ | :--: | ----------- |
| `START_TIME` | Gauge | The time that takes to start playing since the stream is requested until the first frame is shown. |
| `NUM_PLAYS` | Counter | Number of plays. |
| `REBUFFER_TIME` | Gauge | Time of rebuffering, that is the time spent in buffering blocks that are not the initial loading. |
| `NUM_REBUFFERS` | Counter | Number of rebuffering events. |
| `PLAY_TIME` | Gauge | Time playing. |
| `NUM_REQUESTS` | Counter | Number of stream requests. |
| `NUM_LOADS` | Counter | Number of stream loads. |
| `NUM_ENDS` | Counter | Number of stream ends. |


<a name="model"></a>
## 4. KPIs

In this section we are going to expose general terms of how to calculate the most common audio-video KPIs using the OpenAVT data model. But not the exact practice of KPI calculation, because this is something that depends on the platform where our data is recorded. Is totally different a query made for InfluxDB than a query for New Relic.

### 4.1 Start Time

Time elapsed since the stream starts loading until it starts playing.

Is the `timeSinceStreamLoad` (or `timeSinceMediaRequest`) value of the `START` event. This one is probably the most used KPI in audio & video telemetry.

### 4.2 Number of Playback

Number of playback started during a certain period of time.

The simple count of `START` events.

### 4.3 Concurrent Playbacks

Number of concurrent playbacks at a certain moment.

The count of `PING` events. To be accurated the time range selected must be of 30 seconds, because is the ping period. We could improve the granularity by sending pings more often, at the cost of increasing the traffic and database size.

### 4.4 Aborted Before Video Start

The proportion (or number) of streams that started loading but never started playing.

Is the difference between the number of `STREAM_LOAD` and the number of `START` events.

### 4.5 Rebuffering Time

Total time spend in buffering blocks that are not the initial stream loading.

Is the `ACCUM_BUFFER_TIME` attribute minus the start time.

### 4.6 Number of Rebufferings

The number of rebuffering blocks.

Number of `BUFFER_BEGIN` events minus one (the initial).

### 4.7 Number of Quality Changes

The number of quality changes during the playback.

It's a simple count of the number of `QUALITY_CHANGE_UP` and `QUALITY_CHANGE_DOWN` events. Normally the quality changes happen at the begining of a playback, when the player is adjusting the quality to the current connection conditions. But this changes use to happen in 1 to 3 steps. If there are a lot of quality changes during a playback and specially if they happen long after the begining, it usually denotes a unstable connection.

### 4.8 Ended Playbacks without errors

The number or proportion of playbacks that ended normally, without errors.

We use the value of `COUNT_ERROR` when `END` or `STOP` happens. For sessions without error this value must be 0.

### 4.9 Initial vs Mid-stream errors

The proportion of initial errors (errors that happen before the `START` or short after it) and mid-stream errors.

We can distinguish initial `ERROR` events because the value of `timeSinceStart` won't be present or will be very small. For mid-stream `ERROR` events this value will be present and bigger. A common threshold is 1 second.
