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
$ ./src/test/test circuitpadding_sim/.. 
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
git diff data                # Verify new data matches old (no diff)
```

Now, we need to use these client traces to simulate some relay-side traces:
```
rm ./data/undefended/fakerelay-traces/*.trace   # Remove reference trace data
./simrelaytrace.py -i ./data/undefended/client-traces/ -o data/undefended/fakerelay-traces
git diff data                # Timestamps will differ but not event order
```

Once we have both client-side and relay-side trace files, we can simulate
applying a padding machine defense to them, using the previously compiled Tor
test binary:

```
../tor/src/test/test --info circuitpadding_sim/.. --circpadsim ./data/undefended/client-traces/eff.org.trace ./data/undefended/fakerelay-traces/eff.org.trace 1 > ./data/defended/combined-logs/eff.org.log
```

This gives Tor log output of the following format in `./data/defended/combined-logs/eff.org.log`:

```
Dec 10 10:13:50.240 [info] circpad_trace_event__real: timestamp=11339844396 source=relay client_circ_id=1 event=circpad_cell_event_nonpadding_sent
Dec 10 10:13:50.240 [info] circpad_trace_event__real: timestamp=11339850638 source=relay client_circ_id=1 event=circpad_cell_event_nonpadding_sent
Dec 10 10:13:50.240 [info] circpad_trace_event__real: timestamp=11375969198 source=client client_circ_id=1 event=circpad_cell_event_nonpadding_received
Dec 10 10:13:50.241 [info] circpad_trace_event__real: timestamp=11376008271 source=client client_circ_id=1 event=circpad_cell_event_nonpadding_received
```

Note that this log file contains *both* relay and client traces!

To convert that log output into a trace file that can then again be used as
input to classifiers or other code, do:
```
rm ./data/defended/client-traces/*                # Remove any old traces
rm ./data/defended/relay-traces/*                # Remove any old traces
grep "source=client" ./data/defended/combined-logs/eff.org.log > ./data/defended/client-logs/eff.org.log
grep "source=relay" ./data/defended/combined-logs/eff.org.log > ./data/defended/relay-logs/eff.org.log

# XXX: This says invalid trace :/
./torlog2circpadtrace.py -i ./data/defended/relay-logs/ -o ./data/defended/relay-traces/
./torlog2circpadtrace.py -i ./data/defended/client-logs/ -o ./data/defended/client-traces/
git diff ./data/defended/client-traces/          # No diff
git diff ./data/defended/relay-traces/          # Timestamp diffs
```

You should now have defended trace files for the client side and the relay
side.

To verify operation, if you diff your client traces to the ones in this repo,
they should be identical. Note that the simulated relay traces may differ a
bit due to the simulated latency between client and relay.

## Collecting Real Client Traces

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

XXX: TODO write helper to split log files into multiple trace files.

## Creating Fake Relay Traces

In order for padding machines to work, they need traces for both a relay and a
client (because there are padding machines both at the client, and at a
relay).

XXX: Multipile circuit problem

If your experiments are not using timing information, you can create a
synthetic relay trace for input into the simulator using a real client trace:

```
./simrelaytrace.py -i data/circpadtrace-example/ -o data/sim-relay-circpadtrace-example/
```

Note once you have these input trace files, the padding simulator will output
log files with *both* relay and client log lines. You do not need to generate
fake trace files for defended traces:

```
mkdir example example/log example/client example/relay
./tor/src/test/test --info circuitpadding_sim/.. --circpadsim data/circpadtrace-example/eff.org.trace data/sim-relay-circpadtrace-example/eff.org.trace 1 > example/log/defended-eff.org.log

grep "source=client" ./example/log/defended-eff.log  > ./data/torlog-example/client-defended-eff.log
grep "source=relay" ./example/log/defended-eff.log  > ./data/torlog-example/relay-defended-eff.log
./torlog2circpadtrace.py -i ./data/torlog-example/client-defended-eff.log -o ./data/circpadtrace-example/client-defended-eff.trace
./torlog2circpadtrace.py -i ./data/torlog-example/relay-defended-eff.log -o ./data/circpadtrace-example/relay-defended-eff.trace
```

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


