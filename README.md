# Voldeno logic blocks

The block library executed by the Voldeno hub. A block is a small program with
named terminals; users wire blocks together on a canvas in the app and the hub
runs the resulting graph.

## Repository layout

```
blocks/<provider>/<version>/<block>/    definition.json, code.vldn, en.json, pl.json
tests/<provider>/<version>/<block>/     *.yaml test scenarios
events/                                 event message translations
```

The folder path is the block identity: `blocks/vldn/1/relay` is provider `vldn`,
version `1`, name `relay`. Renaming a folder renames the block, so don't.

## Anatomy of a block

| File | Contents |
| --- | --- |
| `definition.json` | terminals, configuration, state, block group |
| `code.vldn` | the logic, written in Volang |
| `en.json`, `pl.json` | display names and descriptions for every id above |

### Groups

| Group | Purpose | Terminals |
| --- | --- | --- |
| `input` | brings a hardware register, schedule or operation mode into the graph | outputs only |
| `process` | logic and control | inputs and outputs |
| `output` | writes to a hardware register or operation mode | inputs only |
| `driver` | implements an abstract block (see below) | none |

### definition.json

```json
{
  "name": "Relay",
  "group": "output",
  "ioType": "register",
  "widget": true,
  "inputs":  [ { "id": "state", "abbrev": "S", "type": "BOOLEAN", "default": "false" } ],
  "outputs": [],
  "config":  [ { "id": "invert", "type": "BOOLEAN", "default": "false" } ],
  "state":   [ { "id": "last_ts", "type": "NUMBER", "default": "0" } ]
}
```

- Terminal types are `NUMBER`, `BOOLEAN` or `STRING`. Config adds `ENUM`,
  `SCHEDULE` and `OPERATION_MODE`, state adds `ARRAY`.
- `default` is **always a string**, even for numbers and booleans.
- `abbrev` is the label drawn on the terminal, max 4 characters. Ids are max 32
  characters with no whitespace, unique within their list.
- `ioType` is `register`, `scheduler`, `operation_mode` or `constant`, and which
  ones are allowed depends on the group. `scheduler` also makes a block run
  periodically, driven by an `extern fn std::next_activation_at()` in its code.
- `widget: true` lets the user expose the block in the mobile app, where its
  inputs become controls and its outputs become readouts.
- Config entries can also carry `unit`, `min`, `max`, `values` (for `ENUM`),
  `visibleWhen` and `validation.rules` (`gte`, `lte`, `gt`, `lt`, `eq`, `neq`,
  `required`, compared against a `value` or another `configId`).

## Writing code.vldn

A block script runs whenever something happens to the block:

- a connected input changes — `input::channel()` returns that input's id and
  `input::value()` its value
- its configuration changes — `input::channel()` returns `"config_change"`
- a timer the block set itself fires — `input::channel()` returns `"callback"`
- a `scheduler` block reaches its next activation — `input::channel()` is empty

Everything the script needs is read explicitly: `input::get("id")`,
`config::get("id")`, `state::get("id")`, and results go out through
`output::set("id", value)`. State is the only thing that survives between runs.

Other functions used across the library: `callback::set` / `callback::clear`,
`time::now` / `time::uptime`, `math::*`, `str::*`, `json::get`, `array::*`,
`map::*`, `http::*`, `tcp::*`, `schedule::*`, `operation_mode::*`, `std::print`.

Things that bite:

- A state variable keeps the type of its default. `"0"` is an integer and
  silently truncates decimals - use `"0.0"` when you store fractions. Outputs
  keep whatever number they are given.
- `state::set` on an id that is not declared in `definition.json` aborts the run.
- Only one timer can be pending per block. `callback::set(ms, "fn_name")`
  replaces nothing, so `callback::clear()` first, and the target must be an
  `extern fn` or the compiler will drop it.
- An `ENUM` config reaches the script as the index of the selected value, a
  `SCHEDULE` config as a schedule id.
- Arithmetic on a decimal stays decimal, and `str::fmt` then renders it as
  `22.500000`. `math::int` turns a number (or a boolean) into a whole one, and
  `str::num_fmt(value, decimals)` prints one with the precision a device wants.
- Write the header names you pass to `http::set_header` canonically - `Host`,
  `Content-Type`, not `host` or `content-type`. Some devices accept nothing
  else, and the app normalises to that form anyway. A device that insists on an
  unusual spelling has to be reached with `tcp::*`.

## Abstract blocks and implementations

An **abstract block** holds logic that is the same for every device of a kind;
an **implementation block** (group `driver`) knows how to talk to one particular
device. The user picks the implementation for a block instance in the app, on the
socket drawn under an abstract block.

Abstract side - a normal block, plus `"abstract": true`:

```json
{ "name": "Air conditioner", "group": "process", "abstract": true, ... }
```

```
temp = std::invoke("read_temperature", "indoor")
```

Implementation side - no terminals of its own, pointing at the abstract block it
implements with the abstract block's provider, version and folder name:

```json
{
  "name": "Air conditioner : Daikin",
  "group": "driver",
  "interface": { "provider": "vldn", "version": 1, "name": "air_conditioner" },
  "config": [ { "id": "host", "type": "STRING", "default": "" } ]
}
```

```
extern fn read_temperature(sensor) {
    // talk to the unit
    return 21.5
}
```

What crosses the boundary:

- **Arguments and the return value** of the call. The implementation must declare
  at least as many parameters as are passed, or the call fails.
- **State**, which both sides share: the implementation reads and writes the
  abstract block's state variables. Since a function returns a single value, this
  is how an implementation reports several readings back.
- **Config is not shared.** Each side sees its own, so device settings like an
  address or a protocol variant belong to the implementation.
- Implementations **cannot read inputs or write outputs** - terminals stay with
  the abstract block.

### Implementation state

A block instance has a single state object, and an implementation declares into
that same object rather than getting one of its own. So an implementation can
keep whatever bookkeeping its device needs, but every state id in a `driver`
descriptor has to start with `drv_`, and no other block may use that prefix:

```json
{ "state": [ { "id": "drv_last_frame", "type": "NUMBER", "default": "0" } ] }
```

That way the abstract block's slots and the implementation's can never collide,
and both sides still read and write them with plain `state::get` / `state::set`.

Implementations never run on their own: they execute inside `std::invoke` and
inside their own completions. A call that finds no attached implementation, or a
name the implementation does not declare, aborts the rest of the run - so write
the outputs you care about before invoking, and keep the set of functions small
and stable.

### Devices that answer later

`http::call` and the `tcp::*` functions only send; the answer arrives later, in a
separate run. The completion belongs to the implementation: an
`extern fn http::on_response(status, body, headers)` declared there runs there,
with the same shared state and its own config, and it does not consume the
block's timer. So a function invoked over a network can only report that the
request is on its way, and needs a way to decline while an earlier exchange is
still running.

The return value of the completion decides what happens next:

| Return | Meaning |
| --- | --- |
| `0` | the implementation's business only - a command acknowledged, or one step of an exchange it is about to continue |
| non-zero | the shared state holds fresh readings |

On non-zero the runtime calls `extern fn std::on_driver_update()` on the
abstract block, if it declares one. That hook is the only way an implementation
reaches back into its block, and it carries nothing: the readings are already in
the shared state, so the hook just turns them into outputs.

`blocks/vldn/1/air_conditioner` is a worked example - its header comment is the
contract every air conditioner implementation has to satisfy, and
`blocks/vldn/1/daikin_brp069` satisfies it over the local network.

## Checks

```sh
validator --resources blocks      # definitions and translations
logic_tests_runner --resources .  # scenario tests
```

Both run on every pull request from the `ghcr.io/voldeno/checker` image.

A test is a yaml file with a `config` map and a list of `steps`. Available
actions are `set_inputs`, `assert_outputs`, `wait` (`duration_ms`),
`assert_output_range` (`output` with `gt`, `gte`, `lt`, `lte` or `neq`) and
`capture_output` (`output` and `as`, referenced later as `$name`):

```yaml
name: "input high"

config:
  limit: 11.0

steps:
  - action: "set_inputs"
    inputs:
      input: true
      value: 10.9
  - action: "assert_outputs"
    comment: "Below the limit"
    expected_outputs:
      output: true
```

Abstract blocks and implementations can only be compile checked: the runner
cannot attach an implementation to a block, nor answer its requests, so any
scenario that reaches `std::invoke` fails.
