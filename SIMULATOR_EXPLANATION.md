# What the Latency-Aware RDM Simulator Does

## In brief

The Latency-aware Remote Data Mirroring (RDM) Simulator models how a data package is copied through a network of mirror servers over time. You choose the number of mirrors, how they are connected, the package size, and timing and bandwidth ranges. The simulator then advances the system in discrete time steps and records how the mirrors and links change as the package spreads.

It is a model for exploring timing, topology, bandwidth, and scaling decisions. It does not send files over a real network or run actual mirror servers.

## The model

The network consists of three main pieces:

- **Mirrors** are simulated servers that can host a copy of the package. A mirror moves through startup and readiness stages, can receive data, and can be shut down or crash.
- **Links** connect pairs of mirrors. A link takes time to activate and has a bandwidth selected from the configured range. Once active, it can carry package data between a mirror that has the complete package and a mirror that does not.
- **The data package** has a configured size. The first mirror is the root and starts with the complete package; the other mirrors acquire it through links.

The package is divided conceptually by the model's received-data counter rather than by simulating individual packets. A receiving mirror accumulates bandwidth from eligible active links until it has the complete package. Link bandwidth is randomized during transfers, so separate runs can produce different timing results.

## What happens during a run

The simulation runs for a configured number of numbered time steps. At each step, it updates the network in this order:

1. Mirrors may crash according to the configured per-step fault probability, then mirror startup, readiness, shutdown, and data-transfer behavior is updated.
2. Links whose endpoints are ready enough can finish activating; links that have been closed or whose endpoints have stopped are removed.
3. Any scheduled changes to the topology, mirror count, or target links per mirror are applied.
4. Probes update their observations, and the simulator records network metrics.

Startup, data-readiness, shutdown, and link-activation delays are randomized within their configured ranges. A crashed non-root mirror loses any partial or complete copy it held and starts again. The root mirror is excluded from random crashes.

Simulation time is separate from wall-clock time. The standard step methods pause briefly between steps to make the visualization viewable; they do not represent real network latency in seconds.

## Network shapes and changes

The topology strategy determines which mirrors can exchange data:

- **Balanced tree** connects mirrors in a tree rooted at the initial mirror. Each mirror has a bounded number of children, based on the configured links-per-mirror setting.
- **Fully connected** creates a link between every pair of mirrors. This offers many possible transfer paths but increases the number of links quickly as the network grows.
- **N-connected** is another supported strategy for networks with a bounded connection pattern.

An `Effector` can schedule changes for particular time steps: set a new target mirror count, switch topology, or change the target number of links per mirror. Scaling up creates mirrors that must still start and obtain the package; scaling down shuts selected mirrors down over time. These changes are not instantaneous.

The simulator also contains example optimizer and MAPEK-loop code. That is an additional layer around the simulation: the core simulator does not automatically optimize the network. In particular, target active-link values used by the examples are recorded for the optimizer workflow and visualization; they are not by themselves a command that forces the network to that active-link count.

## Observations and metrics

The built-in mirror and link probes report counts and ratios, and can print them to the console. The graphical view displays the network and charts over time. The network also keeps histories for:

- **Bandwidth use:** received data compared with the modeled maximum capacity.
- **Active links:** the active-link count expressed relative to the possible connections in the network.
- **Time to write:** a topology-based estimate of how many transfer hops are needed to distribute data, expressed as a score.

These are model measurements, not measurements from a real network. The time-to-write value is an estimate based on topology; it is not a wall-clock duration.

## Configuration

The simulator reads Java properties from a configuration file. The supplied `resources/sim.conf` is a useful starting point:

| Setting | Meaning |
| --- | --- |
| `sim_time` | Number of simulation steps to run. |
| `num_mirrors` | Initial number of mirrors. |
| `num_links_per_mirror` | Target number of links per mirror; topology strategies interpret this according to their structure. |
| `fileSize` | Size of the package in the model's data units (the source describes these as GB). |
| `startup_time_min`, `startup_time_max` | Range for mirror startup delay. |
| `ready_time_min`, `ready_time_max` | Additional delay before a mirror is ready to receive data. |
| `stop_time_min`, `stop_time_max` | Range for mirror shutdown delay. |
| `link_activation_time_min`, `link_activation_time_max` | Range for link activation delay. |
| `min_bandwidth`, `max_bandwidth` | Range for the bandwidth available to a transfer over a link. |
| `fault_probability` | Per-step probability that an eligible mirror crashes. This is a fraction from `0` to `1`; for example, `0.005` means a 0.5% chance per mirror per step, not 0.005 percent. |
| `debug` | Enables probe output in the standard `run()` loop. |
| `target_bandwidth`, `target_active_links`, `target_time_to_write` | Target values used by the example visualization and optimizer setup. |

The random-range maximum follows Java's usual exclusive upper-bound convention. For example, a delay configured from 5 to 10 can be 5, 6, 7, 8, or 9. Use sensible positive ranges and ensure each maximum is greater than its minimum.

When using the no-argument `TimedRDMSim` constructor, the simulator looks for `sim.conf` in the current working directory and copies the bundled default there if it is missing. When using `TimedRDMSim("path/to/file.conf")`, it reads the path you supply.

## Running and extending it

The project targets Java 17 and is built with Maven. A normal library-style run follows this pattern:

```java
TimedRDMSim sim = new TimedRDMSim("resources/sim.conf");
sim.initialize(new BalancedTreeTopologyStrategy());

for (int time = 0; time < sim.getSimTime(); time++) {
    sim.runStep(time);
    for (Probe probe : sim.getProbes()) {
        probe.print(time);
    }
}
```

Initialize the simulator before stepping it, and call the step method once per time step in sequence, starting at step `0`. You can select another topology, inspect the network and its probes, or schedule changes through `sim.getEffector()`. Set the simulator to headless mode when you do not want the Swing visualization.

## What it is useful for

Use the simulator to compare how network structure, mirror startup delays, link activation delays, bandwidth ranges, failures, and scaling actions affect package distribution and the recorded metrics. Because relevant delays and bandwidth are randomized, repeat runs when comparing strategies and compare their results rather than relying on one run.

It is best understood as a controllable timing and topology model. It does not model real network protocols, packet loss, congestion, geographic distance, or precise real-world transfer times unless those behaviors are added to the model.

## RL-agent integration: is there already a bridge?

There is no Python-Java integration layer in this project at the moment. The repository does not contain a socket, HTTP server, subprocess bridge, or Python wrapper for the simulator. The built-in optimization example is still Java-side logic, not a communication layer for an external RL agent.

That means a Python agent cannot currently talk directly to the simulator without an adapter. However, building one would still be useful. The simulator already exposes the right control points:

- `TimedRDMSim` manages the environment lifecycle and time stepping.
- `Network` exposes the current mirror and link state, plus metric histories.
- `Effector` can schedule changes to the number of mirrors, topology, and links-per-mirror.
- Probes can expose the metrics the agent needs for observation.

A practical bridge would wrap the simulator in a Java API that exposes a `reset()` and `step(action)` contract. Python would send an action such as a target mirror count, target links per mirror, or a topology choice, then Java would apply the change, advance one simulation step, and return a structured observation as JSON or another lightweight format.

This is not useless work: it is the standard design for RL integration in a Java simulator. The main caveat is that the simulator currently inserts a 250 ms pause in each `runStep` call, which makes pure RL training much slower than a stripped-down headless version. For training, a bridge is still worthwhile, but the environment should ideally be run in a fast headless mode with the UI and delay removed or made configurable.

So the short answer is: no, there is no existing bridge, and yes, building one is a valid and useful step if the goal is to train or evaluate a Python RL agent against the simulated environment.