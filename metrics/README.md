# Kata Containers metrics

* [Kata Containers metrics](#kata-containers-metrics)
    * [Goals](#goals)
        * [PR regression checks](#pr-regression-checks)
        * [Developer pre-checking](#developer-pre-checking)
    * [Stability or Performance?](#stability-or-performance)
    * [Requirements](#requirements)
        * [For PR checks](#for-pr-checks)
    * [Categories](#categories)
        * [Time (Speed)](#time-speed)
        * [Density](#density)
        * [Networking](#networking)
        * [Storage](#storage)
    * [Saving Results](#saving-results)
        * [JSON API](#json-api)
            * [`metrics_json_init()`](#metrics_json_init)
            * [`metrics_json_save()`](#metrics_json_save)
            * [`metrics_json_add_fragment(json)`](#metrics_json_add_fragmentjson)
            * [`metrics_json_start_array()`](#metrics_json_start_array)
            * [`metrics_json_add_array_element(json)`](#metrics_json_add_array_elementjson)
            * [`metrics_json_add_array_fragment(json)`](#metrics_json_add_array_fragmentjson)
            * [`metrics_json_close_array_element()`](#metrics_json_close_array_element)
            * [`metrics_json_end_array(name)`](#metrics_json_end_arrayname)
    * [Preserving results](#preserving-results)
    * [Report generator](#report-generator)

This directory contains the metrics tests for Kata Containers.

The tests within this directory have a number of potential use cases:
- CI checks for regressions on PRs
- CI data gathering for main branch merges
- Developer use for pre-checking code changes before raising a PR
- As part of report generation

## Goals

This section details some of the goals for the potential use cases.

### PR regression checks

The goal for the PR CI regression checking is to provide a relatively quick
CI metrics check and feedback directly back to the GitHub PR.

Due to the relatively fast feedback requirement, there is generally a compromise
that has to be made with the metrics - precision vs time.

Therefore, there is a separate [script](../.ci/run_metrics_PR_ci.sh) which invokes a
subset of the metrics tests, configured for speed over accuracy.

Having said that, accuracy is still important. If we have very noisy tests, then the
CI will either not spot regressions that are below that noise factor, or will cause
false failures, which are very undesirable in a CI.

### Developer pre-checking

The PR regression check scripts can be executed "by hand", and thus are available
for developers to use as a "pre-check" before submitting a PR. It might be prudent for
developers to follow this procedure particularly for large architectural or version changes
of components.

## Stability or Performance?

When forming, or configuring, metrics tests, often we have to make a choice or compromise
on if we want the test to take a more repeatable measurement (less noise and variance), or if
we want to measure the "best performance".

Generally for CI regression checking, we prefer stability over performance, as that allows
us a more accurate (narrow bound) check for regressions.

> **NOTE** It should thus be noted that if you are gathering data to discuss performance,
> then you may *not* want to use the CI metrics data, as this may be favoring stable results
> over best performance.

## Requirements

To try and maintain the quality of the metrics data gathered and the accuracy of the CI
regression checking, we try to define and stick to some "quality measures" for our metrics.

### For PR checks

The PR CI is generally required to execute within a "reasonable time" to provide timely
feedback to the developers (and not stall the review and development process). To that
end, we relax the quality requirements of the PR CI. The quality requirements are:
- <= 5% run to run variance
- <= 5 minutes runtime per test

## Categories

Kata Container metrics tend to fall into a set of categories, and we organise the tests
within this folder as such.

Each sub-folder contains its own `README` detailing its own tests.

### Time (Speed)

Generally tests that measure the "speed" of the runtime itself, such as time to
boot into a workload or kill a container.

This directory does *not* contain "speed" tests that measure network or storage
for instance.

### Density

Tests that measure the size and overheads of the runtime. Generally this is looking at
memory footprint sizes, but could also cover disk space or even CPU consumption.

For further details see the [density tests documentation](density).

### Networking

Tests relating to networking. General items could include:
- bandwidth
- jitter
- latency

For further details see the [network tests documentation](network).

### Storage

Tests relating to the storage (graph, volume) drivers. Measures may include:
- bandwidth
- latency
- jitter
- conformance (to any relevant standards)

For further details see the [storage tests documentation](storage).

## Saving Results


In order to ensure continuity, and thus testing and historical tracking of results,
we provide a bash API to aid storing results in a uniform manner.

### JSON API

The preferred API to store results is through the provided JSON API.

The API provides the following groups of functions:
- A set of functions to init/save the data and add "top level" JSON fragments
- A set of functions to construct arrays of JSON fragments, which are then added as a top level fragment when complete
- A set of functions to construct elements of an array from sub-fragments, and then finalize that element when all fragments are added.

Construction of JSON data under bash could be relatively complex. This API does not pretend
to support all possible data constructs or features, and individual tests may find they need
to do some JSON handling themselves before injecting their JSON into the API.

> If you find a common use case that many tests are implementing themselves, then please
> factor out that functionality and consider extending this API.

#### `metrics_json_init()`

Initialise the API. Must be called before all other JSON API calls.
Should be matched by a final call to `metrics_json_save`.

Relies upon the `TEST_NAME` variable to derive the file name the final JSON
data is stored in (under the `metrics/results` directory). If your test generates
multiple `TEST_NAME` sets of data then:
- Ensure you have a matching JSON init/save call pair for each of those sets.
- These sets could be a hangover from a previous CSV based test - consider using a single JSON file if possible to store all the results.

This function may add system level information to the results file as a top level
fragment, for example:
- `env` - A fragment containing system level environment information
- "time" - A fragment containing a nanosecond timestamp of when the test was executed


Consider these top level JSON section names to be reserved by the API.

#### `metrics_json_save()`

This function saves all registered JSON fragments out to the JSON results file.

> Note: this function will not save any part-registered array fragments. They will
> be lost.

#### `metrics_json_add_fragment(json)`

Add a JSON formatted fragment at the top level.

| Arg    | Description |
| ------ | ----------- |
| `json` | A fully formed JSON fragment |

#### `metrics_json_start_array()`

Initialise the JSON array API subsystem, ready to accept JSON fragments via
`metrics_json_add_array_element`.

This JSON array API subset allows accumulation of multiple entries into a
JSON `[]` array, to later be added as a top level fragment.

#### `metrics_json_add_array_element(json)`

Add a fully formed JSON fragment to the JSON array store.

| Arg    | Description |
| ------ | ----------- |
| `json` | A fully formed JSON fragment |

#### `metrics_json_add_array_fragment(json)`

Add a fully formed JSON fragment to the current array element.

| Arg    | Description |
| ------ | ----------- |
| `json` | A fully formed JSON fragment |

#### `metrics_json_close_array_element()`

Finalize (close) the current array element. This incorporates
any array_fragment parts into the current array element, closes that
array element, and reset the in-flight array_fragment store.

#### `metrics_json_end_array(name)`

Save the stored JSON array store as a top level fragment, with the
name `name`.

| Arg    | Description |
| ------ | ----------- |
| `name` | The name to be given to the generated top level fragment array |

## Preserving results

The JSON library contains a hook that enables results to be injected to a
data store at the same time they are saved to the results files.

The hook supports transmission via [`curl`](https://curl.haxx.se/) or
[`socat`](http://www.dest-unreach.org/socat/). Configuration is via environment
variables.

| Variable         | Description |
| --------         | ----------- |
| JSON_HOST        | Destination host path for use with `socat` |
| JSON_SOCKET      | Destination socket number for use with `socat` |
| JSON_URL         | Destination URL for use with `curl` |
| JSON_TX_ONELINE  | If set, the JSON will be sent as a single line (CR and tabs stripped) |

`socat` transmission will only happen if `JSON_HOST` is set. `curl` transmission will only
happen if `JSON_URL` is set. The settings are not mutually exclusive, and both can be
set if necessary.

`JSON_TX_ONELINE` applies to both types of transmission.

## Report generator

See the [report generator](report) documentation.
