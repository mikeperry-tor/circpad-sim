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


## QuickStart and Test

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
$ ./src/test/test circuitpadding_sim/.. 
circuitpadding_sim/circuitpadding_sim_main: [forking] OK
1 tests ok.  (0 skipped)
```

This repository also has some example traces that you can apply the built-in
padding machines to, using the unit test as a simulator.

This process requires a client-side trace file, and a relay-side trace file.
To apply compiled-in circpad machines to a pair of relay and client traces:

```
./src/test/test --info circuitpadding_sim/.. --circpadsim ../circpad-sim/data/circpadtrace-example/eff.org.trace ../circpad-sim/data/sim-relay-circpadtrace-example/eff.org.trace 1 > ../circpad-sim/data/torlog-example/defended-eff.log
```

This gives Tor log output of the following format in `../circpad-sim/data/torlog-example/defended-eff.log`:

```
Nov 07 16:22:59.000 [info] circpad_trace_event(): timestamp=1668564966 source=client client_circ_id=2 event=circpad_cell_event_nonpadding_sent
Nov 07 16:23:00.000 [info] circpad_trace_event(): timestamp=1731585868 source=client client_circ_id=2 event=circpad_cell_event_nonpadding_received
Nov 07 16:23:00.000 [info] circpad_trace_event(): timestamp=1731854076 source=client client_circ_id=2 event=circpad_machine_event_circ_added_hop
Nov 07 16:23:00.000 [info] circpad_trace_event(): timestamp=1731901343 source=client client_circ_id=2 event=circpad_cell_event_nonpadding_sent
Nov 07 16:23:00.000 [info] circpad_trace_event(): timestamp=1952113125 source=client client_circ_id=2 event=circpad_cell_event_nonpadding_received
```

To convert that log output into a trace file that can then again be used as
input to the simulator (or other code), do:
```
rm ./data/circpadtrace-example/*       # Remove any old traces
./torlog2circpadtrace.py -i data/torlog-example/ -o data/circpadtrace-example/
```

## Collecting Real Client Traces

To collect a client side trace using Tor Browser (TB):
- copy `src/app/tor` and replace `tor` at `Browser/TorBrowser/Tor` of TB
- in torrc of TB (`TorBrowser/Data/Tor`), add ``Log [circ]info notice stdout''
- run TB with `/Browser/start-tor-browser --log example.log` 

Additional information on running a custom Tor with Tor Browser can be found
in the [Tor Browser Hacking Guide](https://trac.torproject.org/projects/tor/wiki/doc/TorBrowser/Hacking#RunningMultipleTorBrowsers).

## Creating Fake Relay Traces

In order for padding machines to work, they need traces for both a relay and a
client (because there are padding machines both at the client, and at a
relay).

If your experiments are not using timing information, you can create a
synthetic relay trace for input into the simulator using a real client trace:


```
./simrelaytrace.py -i data/circpadtrace-example/ -o data/sim-relay-circpadtrace-example/
```

```
mkdir example example/log example/client example/relay
./tor/src/test/test circuitpadding_sim/.. --circpadsim data/circpadtrace-example/eff.org.trace data/sim-relay-circpadtrace-example/eff.org.trace --info > example/log/eff.org.log
./torlog2circpadtrace.py -i example/log/ -o example/client/
./simrelaytrace.py -i example/client/ -o example/relay/
```

If you compare `example/client/eff.org.trace` and
`data/circpadtrace-example/eff.org.trace` they should be identical. Note that the
simulated relay traces may differ a bit due to the simulated latency between
client and relay.

## Collecting Real Relay Traces

XXX: Test + writeme
   - Document MiddleNodes scheck for negotiation
   - Document logging mechanisms (guard middle exit rp)

## Running experiments

In `circpad-sim-exp.py` you'll find a brief example with mostly comments of how
one could evaluate padding machines with the circuitpadding simulator. Run it as
follows:

```
./circpad-sim-exp.py -c data/circpadtrace-example/ -r data/sim-relay-circpadtrace-example/ -t tor
```

## Details

Ticket [#31788](https://trac.torproject.org/projects/tor/ticket/31788)

TODOs:
- complete `circpad-sim-evaluator.py` as an example of how to use this thing
- consider writing some tests for the simulator

### Usage considerations

It's an embarrassingly parallel problem to sim many traces, so the simulator only
simulates one trace per run. For parallelism, run the simulator many times.
Likely workflow will be dominated by evaluation, including deep learning
traning.

### Limitations

Unfortunately, timers for sending padding cells are unreliable, 0-10 ms extra
delay [#31653](https://trac.torproject.org/projects/tor/ticket/31653). We
currently only document how to simulate traces from a relay, no collection. This
is a flawed approximation, we encourage researchers to carefully consider the
implications of this. Ideally, the community will provide carefully collected
traces in the future with accurate timestamps at both clients and relays. The
real variability of tor's internal timers will remain a problem though in the
simulation.

XXX:
   - Document simulator timing inaccuracies (middle relays, etc)
   - Document that simulator can use only one circ at a time
     - To study multiplexing effects, must split and merge after


