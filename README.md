# Tor Circuit Padding Simulator

A minimal simulator for padding machines in Tor's [circuit padding
framework](https://github.com/torproject/tor/blob/master/doc/HACKING/CircuitPaddingDevelopment.md)

## Overview

The circuit padding simulator consists of two repositories. This repository
holds python glue code that extracts traces from Tor logfiles and synthesizes
new traces.

The branch with [the tor patches you are looking
for](https://github.com/mikeperry-tor/tor/commits/circpad-sim-v4) is
elsewhere. That Tor branch adds patches to instrument Tor such that it logs
cell events at the client, guard, middle, exit, or any other position.
Additionally, it provides a unit test that can take input trace files and
apply circuit padding machines to them.

With both pieces together, your Tor client and Tor relays can record
undefended (non-padded) traces and then apply padding machines to these
traces, yielding defended traces, either in simulation, or on the live Tor
network.

## QuickStart and Test Examples

Assuming you have this repository checked out in a directory named
`circpad-sim`, and you're currently in that directory, then do:

```
cd ..      # from this circpad-sim checkout dir, go up one level
git clone https://github.com/mikeperry-tor/tor.git  # Adjust origin and branches as needed
cd tor
git checkout -t origin/circpad-sim-v4
```

Then build tor as normal. The simulator is tested as part of tor's unit testing
framework, you can check for it as follows:

```
./src/test/test circuitpadding_sim/.. 
circuitpadding_sim/circuitpadding_sim_main: [forking] OK
1 tests ok.  (0 skipped)
```

This repository also has some example logs and traces that you can apply the
built-in padding machines to, using the unit test as a simulator.

First, we must convert our undefended Tor client logs into trace files. From
this circpad-sim checkout, do:
```
rm ./data/undefended/client-traces/*.trace   # Remove reference trace data
./torlog2circpadtrace.py -i ./data/undefended/client-logs/ -o ./data/undefended/client-traces/
git diff data                                # No diff for client traces
```

Now, we need to use these client traces to simulate some relay-side traces:
```
rm ./data/undefended/fakerelay-traces/*  # Remove reference trace data
./simrelaytrace.py -i ./data/undefended/client-traces/ -o data/undefended/fakerelay-traces
git diff data                            # Timestamps differ, but not event order
```

Once we have both client-side and relay-side trace files, we can simulate
applying a padding machine defense to them, using the previously compiled Tor
test binary:

```
../tor/src/test/test --info circuitpadding_sim/.. --circpadsim ./data/undefended/client-traces/eff.org.trace ./data/undefended/fakerelay-traces/eff.org.trace 1 > ./data/defended/combined-logs/eff.org.log
```

This gives Tor log output of the following format in `./data/defended/combined-logs/eff.org.log`:

```
Dec 10 10:13:50.240 [info] circpad_trace_event(): timestamp=11339844396 source=relay client_circ_id=1 event=circpad_cell_event_nonpadding_sent
Dec 10 10:13:50.240 [info] circpad_trace_event(): timestamp=11339850638 source=relay client_circ_id=1 event=circpad_cell_event_nonpadding_sent
Dec 10 10:13:50.240 [info] circpad_trace_event(): timestamp=11375969198 source=client client_circ_id=1 event=circpad_cell_event_nonpadding_received
Dec 10 10:13:50.241 [info] circpad_trace_event(): timestamp=11376008271 source=client client_circ_id=1 event=circpad_cell_event_nonpadding_received
```

Note that this log file contains *both* relay and client traces!

To convert that log output into a trace file that can then again be used as
input to classifiers or other code, do:
```
rm ./data/defended/client-traces/*               # Remove any old traces
rm ./data/defended/relay-traces/*                # Remove any old traces
grep "source=client" ./data/defended/combined-logs/eff.org.log > ./data/defended/client-logs/eff.org.log
grep "source=relay" ./data/defended/combined-logs/eff.org.log > ./data/defended/relay-logs/eff.org.log
./torlog2circpadtrace.py --ip -i ./data/defended/relay-logs/ -o ./data/defended/relay-traces/
./torlog2circpadtrace.py -i ./data/defended/client-logs/ -o ./data/defended/client-traces/
git diff ./data/defended/client-traces/          # No diff
git diff ./data/defended/relay-traces/           # Timestamp diffs
```

You should now have defended trace files for the client side and the relay
side.

To verify operation, if you diff your client traces to the ones in this repo,
they should be identical. Note that the simulated relay traces may differ a
bit due to the simulated latency between client and relay.

## Collecting Client-Side Traces

To collect a client side trace using Tor Browser (TB):
- copy `src/app/tor` and replace `tor` at `Browser/TorBrowser/Tor` of TB
- in torrc of TB (`TorBrowser/Data/Tor`), add ``Log [circ]info notice stdout''
- run TB with `/Browser/start-tor-browser --log example.log` 

Additional information on running a custom Tor with Tor Browser can be found
in the [Tor Browser Hacking Guide](https://trac.torproject.org/projects/tor/wiki/doc/TorBrowser/Hacking#RunningMultipleTorBrowsers).

Note that the example.log file created by Tor Browser will have multiple
different circuits recorded in it. Because the circuit padding simulator only
works on one circuit at a time, you must separate each circuit into its own
log and trace files.

A [set of Tor Browser docker-based orchestration
scripts](https://github.com/pylls/padding-machines-for-tor/tree/master/collect-traces)
to generate a set of undefended traces is also available, but be aware that
additional sanity checking and cleanup is needed to ensure that each site only
uses one circuit.

## Collecting Relay-Side Traces

In order for padding machines to work, they need traces for both a relay and a
client (because there are padding machines both at the client, and at a
relay).

### Synthetic Relay Traces

If your experiments are not using timing information, you can create a
synthetic relay trace for input into the simulator using a real client trace:

```
./simrelaytrace.py -i ./data/undefended/client-traces/ -o data/undefended/fakerelay-traces
```

### Real Middle Relay Traces

If you are reproducing your padding machines on the live network, you will
want to run the [circpad simulator Tor
branch](https://github.com/mikeperry-tor/tor/commits/circpad-sim-v4) with your
padding machines applied as a middle relay.

NOTE: If your experiments are sensitive to time, first see the [limitations
section](#limitations) and the [circpad timing section for more
info](https://github.com/torproject/tor/blob/master/doc/HACKING/CircuitPaddingDevelopment.md#72-timing-and-queuing-optimizations))
before just blindly using the timestamps produced from live crawls.

Your Middle Node Torrc should look roughy like:

```
Nickname researchermiddle
ORPort 9001
ExitRelay 0
Log notice stdout
Log [circ]info file relay-circpad.log
```

Then, when Tor starts up and tells you your relay fingerprint, you should go
back to your Tor Browser torrc, and add:

```
MiddleNodes YOUR_FINGERPRINT_HERE
Log [circ]info file client-circpad.log
```

It will [take some time for new
relays](https://blog.torproject.org/lifecycle-new-relay) to obtain the Fast
flag from the authorites (which they must have to get used by your client).

With pinned middle nodes, the simulator branch will send a special logging
command cell only for your client branch circuits, to those middle nodes,
instructing them to log only your Tor circuits. The circuit IDs will also be
sent across in this cell, so numerically they will match on the client and the
relay. 

This special logging negotiation cell event
(`event=circpad_negotiate_logging`) and its following cell event are present
in client-side log files, but are stripped from the trace files by
`torlog2circpadtrace.py`. They are absent from relay log and trace files.

NOTE: Just like the client side trace converion, that script takes only
the longest trace, and makes no effort to make sure that the
client_circuit_id's match. If you have multiple circuits in your relay log,
you should ensure they are matching properly.

### Real Guard Node Traces

You can alternatively (or additionally) log at the entry node by editing the
`log_at_hops` variable of the function
[circpad_negotiate_logging()](https://github.com/mikeperry-tor/tor/blob/circpad-sim-v4/src/core/or/circuitpadding.c#L2204)
in `tor/src/core/or/circuitpadding.c` in the Tor circpad simulator branch.
You can list as many hop positions as you have relays for there. Then, any
entry nodes or bridges you specify (with `EntryNodes` or `Bridge` directives)
will also log your traces, too.

Clients only request logging from any node if the MiddleNodes directive is
set. This means to log from just the Guard node, you must either change the 
`circpad_negotiate_logging()` check, or always pin generic middles, otherwise
the cell will not get sent.

NOTE: If you list any positions that you do not control in that `log_at_hops`
array, or don't properly restrict your client to use only your relays for
those hops, you will get error cells back, which may affect your results.

ALSO NOTE: If you set up logging to multiple hops at once, the earlier
nodes in the path will observe and record these additional logging cells as
`circpad_nonpadding_cell_sent` events. Removing these is tricky in the general
case, but you may be able to do it sifting through the corresponding client
logs. We do not do anything for this yet.

## Usage considerations

It's an embarrassingly parallel problem to sim many traces, so the simulator only
simulates one trace per run. For parallelism, run the simulator many times.
Likely workflow will be dominated by evaluation, including deep learning
traning.

### Running experiments

In `circpad-sim-exp.py` you'll find a brief example with mostly comments of how
one could script the evaluation of padding machines with the circuitpadding
simulator.

While in this `circpad-sim` directory, run it as follows:

```
./circpad-sim-exp.py -c ./data/undefended/client-traces/ -r ./data/undefended/fakerelay-traces/ -t ../tor
```

### Working With Trace Files

The trace files contain full Circuit Padding Framework event logs at
nanosecond precision. They need some processing before they can be used
in a classifier.

In particular, the classifier should only see `circpad_cell_event_*` events
and it obviously should be not be given visibility into if they are padding or
not. It should only see that they were sent or recieved.

Also, the nanosecond timestamps are way higher precision than a network
adversary may see in practice, and may allow the classifier to learn traits
based on fine-grained application timings. You may want to truncate or
eliminate these timestamps when they are used for classifier input.

See also the [timing accuracy issues](#timing-accuracy-issues) section for
more issues on working with timestamps.

### Scope of Trace files

The simulator branch records all padding and non-padding cells sent on a
circuit immediately after the first circuit handshake has completed 
at the hop that is performing the logging, until the circuit is
closed/destroyed at that hop. The DESTROY cell itself is not counted.
Any forwarded `RELAY_COMMAND_TRUNCATED` cells are.

At the client, this means the first `circpad_cell_event_nonpadding_sent`
event is the onionskin that is sent to the middle hop, since logging
starts after the onionskin completes with the guard/bridge.

At the guard/bridge, the first `circpad_cell_event_nonpadding_received`
event is the onionskin that is to be forwarded to the middle hop.

At the middle relay, the first `circpad_cell_event_nonpadding_received`
event is the onionskin that is to be forwarded to the exit/third hop.

If your experiments rely on circuit setup timing for the handshake
before logging begins, please contact us for ways to provide this.
Otherwise you can probably get away with inserting your own synthetic
cell there.

## Limitations

The simulator has some limitations that you need to be aware of.

### Timing Accuracy Issues

The simulator inherits some [timing
issues](https://github.com/mikeperry-tor/tor/blob/circuitpadding-dev-doc/doc/HACKING/CircuitPaddingDevelopment.md#7-future-features-and-optimizations)
from the Circuit Padding Framework and adds some of its own.

Unfortunately, timers for sending padding cells are unreliable, [0-10 ms extra
delay](https://bugs.torproject.org/32670).

Additionally, the padding framework currently has issues sending cells
back-to-back [with 0 delay](https://bugs.torproject.org/31653).

Finally, all cell event and log collection points for the event callbacks
also impose some inaccuracy due to [queuing
overhead](https://bugs.torproject.org/29494).

In this simulator, we also do not model or factor in the varying latency
of running this attack on the network near or at the Guard node. By default,
we also only simulate middle node traces. This also introduces error.

Ideally, the community will provide carefully collected traces in the future
with accurate timestamps at both clients and relays.

However, until these timing issues are resolved, it is wise to omit timestamps
from classifier input, or at least truncate their resolution considerably.

### Multiplexing Multiple Circuits

The padding simulator only works on one circuit at a time. It also only
extracts the longest circuit id trace from a log file, if multiple circuits
are present.

It also resets time to 0 for the start of each circuit. This means there is
more work needed to model the multiplexing effects of eg Guard node TLS.

To study the effects of multiplexing, you will need to write some scripts to
sepeate logfiles by circuit ID, but additionally store the circuit start time
separately, and use that start time to merge individual defended traces back
into a single properly aligned input into your classifier (which should be 
blind to the circuit separation).

NOTE: Just like the client side log converion, the relay side log conversion
takes only the longest trace, and makes no effort to make sure that the
`client_circ_id` matches. If you have multiple circuits in your relay log,
you should ensure they are matching properly.

### Other TODOs

TODOs:
- complete `circpad-sim-evaluator.py` as an example of how to use this thing
- consider writing some tests for the simulator


