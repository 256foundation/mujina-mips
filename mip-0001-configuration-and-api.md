# MIP-0001: Configuration and API

- Author: Ryan Kuester (@rkuester)
- Created: 2026-05-30

This document states the requirements for mujina-minerd's
configuration system, which unifies five inputs (defaults, config
files, environment variables, command-line arguments, and the REST
API) into a single tree of named nodes. Every value in the tree is
readable and subscribable for live changes, and the settings a
client writes persist across restarts.

## Starting point

mujina-minerd has no central configuration today. It reads each value
directly from environment variables, and those reads are scattered
across the daemon: the pool URL and worker name (`MUJINA_POOL_URL`,
`MUJINA_POOL_USER`), and so on. A `Config` struct exists, but
its loader is unimplemented, and nothing parses a config file at
runtime. The REST API reports telemetry but neither reads nor changes
configuration.

This system starts from scratch. No prior file format or layer
ordering constrains it.

## User requirements

<a id="ur1"></a>
### UR1. Configuration is a tree

Configuration is a tree of named nodes. A path from the root names any
node, the way a filesystem path names a file:

```
/                            the miner itself
  daemon                     the daemon process
    log_level                set by a client
  api                        the REST API
    listen_addr              set; takes effect on restart
  boards                     hashboards, discovered or declared
    e100-e2f56f9b            a discovered USB board, keyed by model and serial
      target_power_w         set: the requested power
      power_w                reported: the actual power
      temperature_c          reported
    cpu-0                    a virtual board for development
      target_duty_percent    set by a client
      hash_rate              reported
  sources                    job sources
    256fdn                   the 256 Foundation donation pool
      url                    set by a client
      accepted_shares        reported
      rejected_shares        reported
```

So `/boards/e100-e2f56f9b/power_w` names the actual power of one
board, and `/daemon` names the whole daemon node.

The root is the miner itself, so it carries no prefix: one daemon
drives one miner. Each top-level node names a concern: `daemon`, `api`,
and the `boards` and `sources` collections. There is no catch-all
"miner config" node: each value lives under the node that owns it, and
a miner-wide node appears only when some value genuinely belongs to the
miner as a whole.

Collections are plural and address their members by key:
`/boards/e100-e2f56f9b`, `/sources/256fdn`.

<a id="ur2"></a>
### UR2. Settings, measurements, and setpoints

Each node is either one a client sets or one the daemon reports. A
client sets the configuration nodes: a board's requested power, the
daemon's log level, a pool URL. The daemon reports the rest, which it
measures or computes (a temperature, a hash rate, a share count) and
a client can only read. The two kinds sit in the same tree at
adjacent paths, as the diagram shows, so a board reports its
temperature right beside the power a client asks of it.

One physical quantity often appears as two nodes: the value a
client commands and the value the hardware actually reaches. The
two differ while the hardware catches up, and differ for good if it
cannot meet the request, so the tree keeps them apart. The commanded
value, the setpoint, takes a `target_` prefix; the measured value
takes the bare name. A board carries `target_power_w`, the power a
client asks for, beside `power_w`, the power it actually draws.
Writing the setpoint asks for a change; reading the bare value tells
you what happened.

Setpoints within one subsystem can overlap: several sometimes drive
the same outcome from different angles. A board's `target_power_w`,
`target_voltage_v`, and `target_frequency_mhz`, for instance, all
decide how hard it runs. The client owns whichever setpoints it
writes, and the daemon resolves whatever it leaves free. Setting
`target_power_w` alone lets the daemon pick voltage and frequency to
hit it; writing `target_voltage_v` and `target_frequency_mhz` by hand
takes both over, and `target_power_w` no longer drives anything;
writing just one of them fixes that knob and leaves the daemon the
other. The client constrains, the daemon fills in the rest.

Leaving a setpoint unset lets the daemon decide the value. An unset
fan `target_percent`, for example, puts the fan under automatic
thermal control. To return a setpoint that currently holds a number
to automatic, write `null` to it: `null` is a real setpoint value,
meaning "you decide," saved like any other. The daemon can set a
setpoint to `null` too: in the example above, once a client pins
`target_voltage_v` and `target_frequency_mhz`, the daemon returns
`target_power_w` to `null`, since power now follows from the two and
is no longer a target.

<a id="ur3"></a>
### UR3. Values resolve through a layered cascade

A configuration value can come from several inputs at once: a
compiled-in default, a file shipped by a package, a file dropped in at
boot by a system service, a file written by an administrator, a value
saved through the API, an environment variable, or a command-line
argument. The file paths below are the system layout, used when the
daemon runs as a packaged service; running it any other way
remaps them ([UR11](#ur11)). When more than one input supplies the same path, a
fixed precedence decides which takes effect, lowest to highest:

1. Code defaults, compiled in.
2. Package config files (`/usr/lib/mujina/config.d/*.yaml`), dropped
   in by a site-config or board-support package; Mujina's own defaults
   are compiled in (layer 1), not shipped here.
3. Runtime drop-in config files (`/run/mujina/config.d/*.yaml`),
   written by system services at boot and cleared on reboot. See below.
4. Admin config files (`/etc/mujina/config.yaml` and
   `/etc/mujina/config.d/*.yaml`), the administrator's space. Install
   packages put their files in layer 2 and leave this directory
   alone, so an upgrade never overwrites an administrator's edit.
   Conventional for a Unix daemon and kept for completeness, though
   seldom needed.
5. Saved config (`/var/lib/mujina/saved-config.yaml`), where API
   writes persist.
6. Environment variables, read once at startup.
7. Command-line arguments such as `--set`, read once at startup.

Where a layer has a main config file (`config.yaml`), it is loaded
first; drop-in files in the layer's `config.d/` directory are loaded
next, in lexical order by the `NN-name.yaml` convention. Each later
file overrides keys set by earlier ones, matching systemd's drop-in
convention.

Layer 3 is volatile: `/run` is on tmpfs, so anything dropped there is
gone after a reboot. The layer is meant for configuration the system
generates at boot, written by a system service into
`/run/mujina/config.d/`. A boot service might name each miner from
its hardware serial number, IP address, or rack position, read off a
plugged-in USB drive, etc., and write it into the directory, so
the name is set fresh each boot instead of baked into the image.

Config files and saved config are both YAML; [DR4](#dr4) explains the format
choice.

The config-file layers are read-only input. A package, a system
service, or an administrator lays them down; the daemon never edits
them. Saved config is the only layer the daemon writes, and so the
only one that persists a runtime change ([UR8](#ur8)).

Env vars and `--set` feed the cascade only at load. Once the daemon is
running, saved config is the highest layer in play, so a runtime API
write updates the effective value directly and persists, even on a path
an env var or `--set` set at startup. The launch-time flag has done its
job, and the running client's command takes over.

That win lasts only while the daemon runs. The write persists to saved
config, but the next start resolves the cascade afresh; if the same env
var or `--set` is set again, it again outranks saved config, and the
effective value returns to the launch-time setting. An env var or
`--set` value is itself never written to saved config; the daemon
reads it at load as an input layer, leaving the runtime write in
place beneath it.

How an environment variable name or a `--set` argument maps to a
tree path is a matter of formatting, deferred.

<a id="ur4"></a>
### UR4. The API addresses the tree

The REST API and the configuration system are one tree. A path into
the tree is a path into the API: below the `/api/v0` prefix, a URL is
the JSON Pointer into the tree ([DR3](#dr3)). So

```
GET /api/v0/boards/e100-e2f56f9b/power_w
```

reads the same node that a config file sets and a subsystem consumes.

The same addressing works at every depth: one mechanism serves a
single leaf, a whole subtree, or an entire collection. Because every
node has an address, a write names the exact node it means to change
and replaces that node's value with PUT:

```
PUT /api/v0/boards/e100-e2f56f9b/target_power_w
```

Setting one value is therefore a single PUT to that leaf. To change
several fields of a node in one atomic request, PATCH the node with a
partial document, which updates only the fields it names and leaves
the rest alone:

```
PATCH /api/v0/boards/e100-e2f56f9b
{ "target_frequency_mhz": 525, "target_voltage_v": 1.15 }
```

Deleting a saved entry, so the layer below in the cascade ([UR3](#ur3)) takes
over, uses DELETE:

```
DELETE /api/v0/boards/e100-e2f56f9b/target_power_w
```

The API also exposes nodes the configuration tree does not store. A
read-only `/chips` collection, for example, gathers every chip across
the boards with its temperature and hash rate, computed at read time.
The tree stores those values nested, under each board's hash threads;
the collection flattens them into one view.

<a id="ur5"></a>
### UR5. Every node is reactive

Every node supports four operations:

- **Read** its current value.
- **Subscribe** to its changes.
- **Write** a new value. A setting a client writes propagates to
  subscribers and persists ([UR3](#ur3)); a reported value the daemon writes
  propagates the same way but is live, not saved.
- **Delete** the saved entry at this path. For a
  leaf, the layer below then takes over. For a collection member that
  exists only in saved config, the member is gone.

Which actor performs each operation follows from [UR2](#ur2)'s split:
the client sets, the daemon reports, and either kind can be read and
subscribed.

A subscription can be exposed to a remote client over a streaming
transport such as a WebSocket, server-sent events, or streaming HTTP,
not only to in-process code; the transport choice is deferred. A
client watches any path or subtree, named exactly as a read names it,
and receives a notification on each change, with no polling. Each
notification is a delta: the path that changed paired with its new
value. A change to one board's power arrives as, say, `{"path":
"/boards/e100-e2f56f9b/power_w", "value": 1485}`, using the same JSON
Pointer paths as the API ([DR3](#dr3)). Keying each change by its path
is what lets a single subtree subscription report many leaves: a
watcher on `/boards/e100-e2f56f9b` receives one such delta for every
leaf that changes beneath it.

<a id="ur6"></a>
### UR6. Every value is validated as it enters

Each leaf validates the values that enter it, and that validation
applies to every value whatever its source: a file reload, an API
write, an environment variable, or a `--set` argument at startup. An
invalid value is rejected where it tries to enter the tree. (At
startup, a rejected value is fatal; see [UR10](#ur10).)

An unrecognized key is a different case from an invalid value. A key
that the daemon does not recognize names no leaf, so there is no
validator to fail. From a config file or an environment variable, the
daemon ignores it with a warning that points at it, so a typo, a stale
entry, or a file written for a newer daemon does not stop a load. A
`--set` argument that names an unknown path is the strict case: the
user just typed it explicitly, so a misspelling there is an
error rather than a warning.

The same validators that run inside the daemon are also available to
a command-line tool that checks a config file offline, without
starting the daemon. The tool reads a file the way the daemon would,
runs each leaf's validator, and reports problems with their path and
source line. A strict mode promotes unknown-key warnings to errors,
so a user can catch typos and stale entries before shipping a
file, instead of finding them in the daemon's log after a restart.

<a id="ur7"></a>
### UR7. The CLI and API report the effective configuration

The authoritative answer to "what is the configuration now" comes
from the CLI or the API, not from reading a file. A read returns the
effective merged value for any path; config files are one input layer
among several ([UR3](#ur3)), not the source of truth.

Where a value came from, which layer supplied it, is available on
request, not attached to every read. Only the daemon sees all the
layers, so only it can answer; it resolves origin during the cascade
and reports it when asked, in the spirit of `git config --show-origin`
or `sysctl`. The CLI is a thin client over the API, so it shows
origin behind an explicit flag, backed by that same API capability
rather than by reading files itself.

<a id="ur8"></a>
### UR8. Saved config is a sparse overlay

Saved config holds exactly the values a client writes through the
API ([UR3](#ur3)), and nothing else. A pristine system saves nothing, and
each write adds only the path it touched, so the file stays sparse by
usage, not because the daemon prunes it. Deleting a value removes it
from saved config, so the config file or default beneath takes effect
again.

Because saved config records explicit intent, changing a default
never disturbs it. A value the client never set picks up the new
default. A value the client did set keeps what was written, even
where that happened to match the old default.

A startup argument, something like `--delete-saved-config`, sets the
saved config aside before the daemon loads anything. Recovery is the
sharpest reason: a saved value that keeps the daemon from starting
leaves no running daemon to accept a delete, so the escape must be
reachable at startup. It also gives a clean reset, dropping every
runtime override to return the daemon to its file-and-default
configuration. Because it clears all overrides at once, it renames the
existing file aside rather than destroying it, leaving the user a copy
to restore what was good. Wholesale delete of saved config while the
daemon runs is a separate convenience and is deferred.

A write can pass validation and still cut the client off, for example a
setting that crashes a board driver or makes the miner unreachable on
the network. The daemon has no guard against such a write. One guard
would be to stage runtime writes and save them only on an explicit
commit, so a hard reset returns the miner to its last committed
configuration. That guard is deferred until there is some experience
with the configuration system.

<a id="ur9"></a>
### UR9. Discovery and declaration populate collections

Two sources populate a collection like `boards`: runtime discovery
(USB hotplug) and static declaration in a config file. A declared
member is a peer of a discovered one, sharing the same `/boards/:id`
shape and place in the tree.

Declaration exists for hardware that cannot be enumerated, such as a
board on an RS-232 serial port. It supplies exactly what enumeration
otherwise would:

- Type: the board model, which selects the driver, protocol, and
  chip count. Discovery infers this from the device; a declaration
  states it, since a dumb serial line cannot be safely probed.
- Transport: the serial device path and parameters (baud rate, and
  any port layout a chain needs). Discovery derives these from the
  enumerated device; a declaration gives them directly.
- Identity: a stable id used as the collection key and the API
  address. Discovery names a board from its model and serial. For a
  declared board, the driver may read identifying information once it
  knows the type; the declaration names the board only when the
  driver cannot.

Those three are all a declaration must supply. A board's tunable
settings, power or duty and the like, are ordinary configuration under
its `/boards/:id` subtree; a config file may set them alongside the
declaration, or leave them to the API or to defaults. Either way they
flow through the normal cascade ([UR3](#ur3)), just as for a discovered
board.

A declared member has a different lifecycle from a discovered one. It
is assumed present, so there is no hotplug add or remove; failure to
reach it is an error, not a disconnect. Removing a declared board
means removing its declaration from the file: API delete ([UR5](#ur5))
operates on saved config and cannot touch a lower-layer declaration.
Declaration is a config-file act today; the layered model would
hypothetically let an API write create or adjust a board in saved
config instead, where API delete could reach it, but that is deferred.

<a id="ur10"></a>
### UR10. Configuration loads at startup and on explicit reload

The daemon reads the config files once at startup and re-reads them
only on an explicit reload: SIGHUP (which the packaged unit wires
`systemctl reload` to send) or a reload request on the API. There is
no continuous file watching. An edited file takes effect when the
user, or the package that delivered it, asks for a reload, not the
instant the file changes.

A value that fails validation while loading at startup is fatal: the
daemon reports the error and exits before any subsystem comes up,
rather than running half-configured. [UR6](#ur6) requires every entering
value to be validated; at startup, a failure stops the launch.

Reload re-reads the config-file layers only. Environment variables
and arguments are captured once at startup and not refreshed ([UR3](#ur3));
the saved config is always current since API writes go straight to
it. The cascade still governs the outcome: if a reloaded file value
is shadowed by a higher layer (saved config, env, or argument), the
effective value does not change and nothing reaches subsystems.
Where the reload does change an effective value, the change reaches
subsystems through the same notification path as an API write ([UR5](#ur5)).
A file reload does not override a transient env or `--set` setting
the way a runtime API write does ([UR3](#ur3)); those still win by layer
precedence.

<a id="ur11"></a>
### UR11. File locations adapt outside of a packaged system install

The file locations in [UR3](#ur3) are the system layout, used when the
daemon runs as a packaged service. Run any other way, the daemon stays
out of the system paths: it uses a per-user layout where one is set
up, and otherwise runs on compiled-in defaults alone, so a developer
who runs the daemon from a checkout neither reads from nor writes to
system paths.

The daemon walks this list top to bottom, stopping at the first
source that resolves:

1. **`--system` flag.** Use the system layout of [UR3](#ur3). This is what
   the packaged systemd unit passes.
2. **`--no-config` flag.** Force the defaults-only mode of step 7
   even when XDG or `.mujina/` directories would otherwise resolve,
   skipping discovery entirely.
3. **`-c` / `--config-file <path>`.** Read configuration from this
   one file only; no `config.d/`, no runtime layer, and no
   `saved-config.yaml`. API writes update the effective value for the
   running daemon but are not persisted, so a restart returns to what
   the file plus defaults supply. The argument `-` reads the file
   from stdin. This mode suits short tests, single-file experiments,
   and CI runs where setting up a directory is overhead.
4. **`-d` / `--mujina-dir <path>`** or **`MUJINA_DIR`.** The given
   directory is `<mujina-dir>`.
5. **Discovered `.mujina/` directory.** The daemon walks up from the
   current working directory; the first `.mujina/` it finds is
   itself the `<mujina-dir>`.
6. **XDG base directories**, triggered when `$XDG_CONFIG_HOME/mujina/`
   exists. That directory is `<mujina-dir>`; saved config is created
   at `$XDG_STATE_HOME/mujina/saved-config.yaml` if it does not
   already exist. Per the XDG spec, unset variables resolve to their
   conventional `$HOME` defaults.
7. **Defaults only**, the terminal fallback when no step above
   resolves. The daemon runs on compiled-in defaults plus whatever
   arrives via env, CLI, and API, with a transient saved config: API
   writes take effect for the running daemon but are never written to
   disk, and the daemon creates no config files or directories of its
   own. This is the same transient behavior as stdin mode (step 3).

Inside `<mujina-dir>`:

```
<mujina-dir>/
  config.yaml        # optional: main config file
  config.d/          # optional: drop-in config files
    *.yaml
  saved-config.yaml  # written by the daemon
```

Both `config.yaml` and `config.d/` are optional. If neither exists,
the daemon runs with compiled-in defaults plus whatever lands via
env, CLI, and API. `saved-config.yaml` is created on first API
write. User mode has no `/run`-analog runtime layer; that layer is
system-mode only.

## Design requirements

<a id="dr1"></a>
### DR1. The configuration tree is one object

Configuration lives in one data structure inside the daemon. Every
reader and writer touches that same tree directly: a config file at
load, an environment variable at startup, an API call at runtime, a
subsystem consumer. There is no separate "API shape" and "internal
shape" of configuration that the daemon has to keep in sync, and no
conversion code between them.

<a id="dr2"></a>
### DR2. Typed access for consumers

Subsystems work with typed handles, not with paths or raw tree data. A
board driver gets a typed config handle, say a `HashboardConfig`, and
calls read, subscribe, and write on it; it never builds a path like
`/boards/e100-e2f56f9b` and never parses JSON or YAML. The accessor
layer does the parsing and path-walking once on behalf of all
subsystems. Because paths and storage formats stay confined to that
layer, the tree's shape and format can also change without touching
any subsystem.

<a id="dr3"></a>
### DR3. JSON Pointer and JSONPath

[RFC 6901](https://datatracker.ietf.org/doc/html/rfc6901) JSON
Pointers (`/sources/256fdn/url`,
`/boards/e100-e2f56f9b/target_power_w`) are the path syntax wherever
a single node is identified externally: API URLs, CLI overrides for
one leaf, persistence policy, change-notification deltas.

[JSONPath](https://datatracker.ietf.org/doc/html/rfc9535) (RFC 9535)
is the syntax wherever a pattern selects a set of nodes rather than
identifying one: applying a setting to every board of a given model,
querying telemetry across a collection, or any other multi-node
operation. A query like `$.boards.*.target_power_w` selects the
`target_power_w` leaf of every board; filter selectors narrow the
match when needed. The implementation supports the subset of
JSONPath the system actually needs; queries outside that subset are
rejected with a clear error. The two share the same JSON tree model
and cover complementary cases: one identifies a node, the other
selects a set.

<a id="dr4"></a>
### DR4. Configuration file format

Config files and saved config are both YAML. Using one format
throughout means a user who tuned a value through the API and
wants to lock it in as a default can open saved config and copy the
value straight into a config file, without translating between
formats. The daemon writes saved config as a strict block-style YAML
subset (no anchors, no implicit type ambiguity), so it round-trips
cleanly even though YAML in general can be sloppy.
