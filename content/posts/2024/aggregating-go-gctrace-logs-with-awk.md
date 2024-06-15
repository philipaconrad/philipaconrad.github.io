---
title: "Aggregating Golang's gctrace Logs with Awk"
date: 2024-06-15
summary: "Intro to Golang gctrace logs + a helpful Awk script for aggregating them."
---

# Introduction

I was working on an Open Policy Agent (OPA) PR ([#6818][opa-pr-6818], to be exact), and was having difficulty explaining why the server would start faster when preallocating a particularly large buffer.

I suspected it was due to the Golang garbage collector not having to work as hard, but I couldn't *quantify* the effect.
Not, that is, until I remembered that the Go runtime supports some kind of GC tracing.


## Observing the GC's behavior with GODEBUG

The Go runtime supports a surprising amount of tracing and debugging tooling, all of which is hidden behind opt-in toggle switches in the `GODEBUG` environment variable ([docs link][godebug-docs]).

In my case, I wanted to see what the GC was doing while the Open Policy Agent was starting up with a huge bundle file.
This was accomplished with the following CLI invocation:

    GODEBUG=gctrace=1 ./opa run -s -b bundle-1gb.tar.gz

This would start OPA in server mode, loading up a 1 GB bundle file from disk before enabling the HTTP server. The GODEBUG invocation turns on the `gctrace` feature, which according to the Go 1.22 docs does the following:

```
gctrace: setting gctrace=1 causes the garbage collector to emit a single line to standard
error at each collection, summarizing the amount of memory collected and the
length of the pause. [...]
```
There's a lot of other details on the docs page that I found useful later, but that sounded helpful!
I immediately gave it a go, and the results were helpful.

Here's an example trace from my 8-core `i7-8650U` ThinkPad, with 32 GB of RAM (so memory pressure was never an issue during testing):
```
gc 1 @0.019s 2%: 0.14+1.3+0.045 ms clock, 1.1+0.50/1.9/1.4+0.36 ms cpu, 3->4->1 MB, 4 MB goal, 0 MB stacks, 0 MB globals, 8 P
gc 2 @0.024s 2%: 0.028+4.3+0.039 ms clock, 0.22+0/1.6/5.8+0.31 ms cpu, 3->3->2 MB, 4 MB goal, 0 MB stacks, 0 MB globals, 8 P
gc 3 @0.034s 2%: 0.058+10+0.016 ms clock, 0.47+0/1.7/12+0.13 ms cpu, 8->8->7 MB, 8 MB goal, 0 MB stacks, 0 MB globals, 8 P
gc 4 @0.046s 2%: 0.026+8.1+0.023 ms clock, 0.21+0/1.0/9.3+0.18 ms cpu, 15->15->13 MB, 15 MB goal, 0 MB stacks, 0 MB globals, 8 P
gc 5 @0.056s 2%: 0.014+10+0.012 ms clock, 0.11+0/0.63/11+0.096 ms cpu, 29->29->25 MB, 29 MB goal, 0 MB stacks, 0 MB globals, 8 P
gc 6 @0.070s 1%: 0.024+22+0.041 ms clock, 0.19+0/0.98/22+0.33 ms cpu, 57->57->49 MB, 57 MB goal, 0 MB stacks, 0 MB globals, 8 P
gc 7 @0.100s 1%: 0.034+38+0.003 ms clock, 0.27+0/0.61/38+0.025 ms cpu, 113->113->97 MB, 113 MB goal, 0 MB stacks, 0 MB globals, 8 P
gc 8 @0.152s 0%: 0.024+70+0.097 ms clock, 0.19+0/0.58/71+0.78 ms cpu, 225->225->193 MB, 225 MB goal, 0 MB stacks, 0 MB globals, 8 P
gc 9 @0.250s 0%: 0.024+134+0.011 ms clock, 0.19+0/0.58/134+0.088 ms cpu, 449->449->385 MB, 449 MB goal, 0 MB stacks, 0 MB globals, 8 P
gc 10 @0.433s 0%: 0.023+268+0.053 ms clock, 0.18+0/0.54/268+0.42 ms cpu, 897->897->769 MB, 897 MB goal, 0 MB stacks, 0 MB globals, 8 P
gc 11 @0.801s 0%: 0.024+514+0.003 ms clock, 0.19+0/0.73/514+0.031 ms cpu, 1793->1793->1537 MB, 1793 MB goal, 0 MB stacks, 0 MB globals, 8 P
gc 12 @3.252s 0%: 0.028+504+0.011 ms clock, 0.22+0/252/0.87+0.090 ms cpu, 3585->3585->2561 MB, 3585 MB goal, 0 MB stacks, 0 MB globals, 8 P
{"addrs":[":8181"],"diagnostic-addrs":[],"level":"info","msg":"Initializing server. OPA is running on a public (0.0.0.0) network interface. Unless you intend to expose OPA outside of the host, binding to the localhost interface (--addr localhost:8181) is recommended. See https://www.openpolicyagent.org/docs/latest/security/#interface-binding","time":"2024-06-12T18:14:21-04:00"}
```

The last line is the usual "everything's healthy!" message OPA spits out when it's done loading up bundles and data, and is ready to serve authorization requests.

There's a TON of information in those traces! Everything from how long individual GC phases took, to the state of the Golang heap, and more.


## The format of gctrace (as of Go 1.22)

The Golang docs are quite helpful, and explain things better than I could, so I've quoted the relevant bits below:
```
[...]
Currently, [the format] is:
gc # @#s #%: #+#+# ms clock, #+#/#/#+# ms cpu, #->#-># MB, # MB goal, # MB stacks, #MB globals, # P
where the fields are as follows:
	gc #         the GC number, incremented at each GC
	@#s          time in seconds since program start
	#%           percentage of time spent in GC since program start
	#+...+#      wall-clock/CPU times for the phases of the GC
	#->#-># MB   heap size at GC start, at GC end, and live heap, or /gc/scan/heap:bytes
	# MB goal    goal heap size, or /gc/heap/goal:bytes
	# MB stacks  estimated scannable stack size, or /gc/scan/stack:bytes
	# MB globals scannable global size, or /gc/scan/globals:bytes
	# P          number of processors used, or /sched/gomaxprocs:threads
The phases are stop-the-world (STW) sweep termination, concurrent
mark and scan, and STW mark termination. The CPU times
for mark/scan are broken down in to assist time (GC performed in
line with allocation), background GC time, and idle GC time.
If the line ends with "(forced)", this GC was forced by a
runtime.GC() call.
```

This is a huge amount of information, but for my purposes, I really wanted to understand the *aggregate* behavior of the garbage collector, i.e. how much time *total* have we spent collecting garbage for a given trace.

This means the fields I care about are the "clock" and "cpu" groups:
```
gc # @#s #%: #+#+# ms clock, #+#/#/#+# ms cpu, #->#-># MB, # MB goal, # MB stacks, #MB globals, # P
             ^^^^^           ^^^^^^^^^
```

The clock group gives us the "[wall clock time][wikipedia-wall-clock-time]" the GC spent for a given collection.
If we sum those lines up, we get the total time the program has spent doing GC, which is awesome.

I also realized having a detailed breakdown could be handy for seeing if one or more GC phases were particularly affected.
That wound up making the final aggregator script a bit more complex, but I think it's worthwhile.


## Aggregating gctraces with Awk

> Huge thanks to my friend [Charles][charles-blog] for his help putting this script together!

I considered writing a Python or Go program to parse the logs with regexes, but didn't want the heavyweight Python interpreter or Golang compile-then-run workflows.
I realized a much older Unix tool might work very well for the job though-- Awk!

Awk is an underappreciated Unix tool that's been around since the dark ages of computing, and is shockingly useful for one-off log parsing/reporting jobs.
I also happened to have worked with an Awk wizard in the past (Charles), so I knew Awk was capable of doing what I wanted.

My first attempt was purely regex-based, and failed horribly, because my Ubuntu box didn't have GNU Awk on it as the default `awk`.
After some back-and-forth with Charles, I wound up with the much more idiomatic Awk script below that breaks apart the line on a variety of separator characters.
Then, the script simply grabs out the useful fields, and goes on with adding them to running totals and such.

`summarize-gctrace.awk`:
```awk
# AWK script for summarizing GODEBUG=gctrace=1 logs.
# Copyright (c) Philip Conrad and Charles Daniels, 2024.
# Released under the Apache License 2.0.
#
# Known to work for Go 1.22, but the log format is subject to change.
# The format is the following (pulled from the Go `runtime` package docs):
#   gc # @#s #%: #+#+# ms clock, #+#/#/#+# ms cpu, #->#-># MB, # MB goal, # MB stacks, #MB globals, # P
#                ^$5             ^$10                                                               ^$28
# Example:
#   gc 11 @3.169s 1%: 0.025+509+0.017 ms clock, 0.10+0/261/0.81+0.068 ms cpu, 3585->3585->2561 MB, 3585 MB goal, 0 MB stacks, 0 MB globals, 4 P

BEGIN {
    FS = "[\t +/]+"
}

# Some simple assertions to prevent matching non-gctrace logs.
$1 != "gc" { non_matching_rows++; next; }
$9 == "cpu" { non_matching_rows++; next; }

# The main pattern-action block:
{
    num_threads = $28

    clock_stw_sweep_term += $5
    clock_conc_mark_scan += $6
    clock_stw_mark_term += $7

    cur_clock = $5 + $6 + $7
    total_clock += cur_clock
    max_pause_clock = (max_pause_clock > cur_clock) ? max_pause_clock : cur_clock

    cpu_ms_stw_sweep_term += $10
    cpu_ms_assist_gc += $11
    cpu_ms_background_gc += $12
    cpu_ms_idle_gc += $13
    cpu_ms_stw_mark_term += $14
    total_cpu += $11 + $12 + $13
}

END {
    print  "--- Wall clock time summary -------------------"
    printf "  STW sweep termination ......... %10.3f ms\n", clock_stw_sweep_term
    printf "  Concurrent mark and scan ...... %10.3f ms\n", clock_conc_mark_scan
    printf "  STW mark termination .......... %10.3f ms\n", clock_stw_mark_term
    printf "                          Total = %10.3f ms (%.3f s)\n", total_clock, total_clock / 1000
    printf "               Longest GC Pause = %10.3f ms\n", max_pause_clock
    print  ""
    print  "--- CPU time breakdown ------------------------"
    printf "  STW sweep termination ......... %10.3f ms\n", cpu_ms_stw_sweep_term
    printf "    Mark/Scan assist time ....... %10.3f ms\n", cpu_ms_assist_gc
    printf "    Mark/Scan background GC time  %10.3f ms\n", cpu_ms_background_gc
    printf "    Mark/Scan idle GC time ...... %10.3f ms\n", cpu_ms_idle_gc
    printf "  STW mark termination .......... %10.3f ms\n", cpu_ms_stw_mark_term
    printf "                          Total = %10.3f ms (%.3f s) (Does not include STW phases)\n", total_cpu, total_cpu / 1000
    print  ""
    print  "Num threads:", num_threads
    print  "Non-matching rows:", non_matching_rows
}
```

This generates reports like the following (taken from running `awk -f summarize-gctrace.awk logs.txt` on the earlier logs example):

```
--- Wall clock time summary -------------------
  STW sweep termination .........     0.447 ms
  Concurrent mark and scan ......  1583.700 ms
  STW mark termination ..........     0.354 ms
                           Total = 1584.501 ms (1.585 s)

--- CPU time breakdown ------------------------
  STW sweep termination .........     3.540 ms
  Mark/Scan assist time .......       0.500 ms
  Mark/Scan background GC time      262.850 ms
  Mark/Scan idle GC time ......    1087.370 ms
  STW mark termination ..........     2.840 ms
                           Total = 1350.720 ms (1.351 s) (Does not include STW phases)
```

This is *much* more useful for understanding what's going on.
We can now talk about the total time OPA spends in garbage collection while it's starting up, which makes the performance argument in the PR stronger.

### Running alongside OPA

OPA running in server mode presented a challenge: collecting the `stderr` trace from OPA would either require redirecting to a file and then separately processing the logs, or else careful shell wrangling.

I wound up with the following, somewhat cursed invocation sequence across 2x shells:

Terminal 1:
```
$ GODEBUG=gctrace=1 ./opa run -s -b bundle-1gb.tar.gz 2> >(tee >(awk -f summarize-gctrace.awk >&2) >&2)
```

After observing the `{"addrs":[":8181"], ...` startup log line from OPA, I fired up a second terminal.

Terminal 2:
```
$ pkill -SIGINT opa
```

Which would kill the running OPA server, *without* also killing the Awk script running off of its `stderr` feed.

This gave me the best of both worlds: being able to see the logs, live from the server, while also feeding said logs to the Awk script to get a helpful summary at the end.

Here's what an example final printout looks like on Terminal 1:

```
$ GODEBUG=gctrace=1 ./opa-main run -s -b bundle-1gb.tar.gz --pprof 2> >(tee >(awk -f summarize-gctrace.awk >&2) >&2)
gc 1 @0.141s 2%: 0.053+5.5+2.8 ms clock, 0.42+3.1/3.0/5.1+22 ms cpu, 3->3->1 MB, 4 MB goal, 0 MB stacks, 0 MB globals, 8 P
gc 2 @0.160s 2%: 0.043+6.1+0.063 ms clock, 0.35+0/2.1/8.7+0.50 ms cpu, 3->3->2 MB, 4 MB goal, 0 MB stacks, 0 MB globals, 8 P
gc 3 @0.175s 2%: 0.059+17+0.054 ms clock, 0.47+0/2.9/19+0.43 ms cpu, 8->8->7 MB, 8 MB goal, 0 MB stacks, 0 MB globals, 8 P
gc 4 @0.194s 2%: 0.027+11+0.17 ms clock, 0.21+0/1.2/12+1.4 ms cpu, 15->15->13 MB, 15 MB goal, 0 MB stacks, 0 MB globals, 8 P
gc 5 @0.208s 2%: 0.017+11+0.011 ms clock, 0.13+0/0.80/12+0.089 ms cpu, 29->29->25 MB, 29 MB goal, 0 MB stacks, 0 MB globals, 8 P
gc 6 @0.223s 2%: 0.018+16+0.15 ms clock, 0.14+0/0.59/17+1.2 ms cpu, 57->57->49 MB, 57 MB goal, 0 MB stacks, 0 MB globals, 8 P
gc 7 @0.246s 1%: 0.011+32+0.012 ms clock, 0.088+0/0.56/32+0.10 ms cpu, 113->113->97 MB, 113 MB goal, 0 MB stacks, 0 MB globals, 8 P
gc 8 @0.290s 3%: 0.020+64+0.009 ms clock, 0.16+0/64/0.67+0.075 ms cpu, 225->225->193 MB, 225 MB goal, 0 MB stacks, 0 MB globals, 8 P
gc 9 @0.378s 3%: 0.020+127+1.8 ms clock, 0.16+0/0.52/127+15 ms cpu, 449->449->385 MB, 449 MB goal, 0 MB stacks, 0 MB globals, 8 P
gc 10 @0.554s 5%: 0.022+254+0.011 ms clock, 0.18+0/254/0.68+0.088 ms cpu, 897->897->769 MB, 897 MB goal, 0 MB stacks, 0 MB globals, 8 P
gc 11 @0.901s 3%: 0.023+505+0.003 ms clock, 0.18+0/0.62/505+0.028 ms cpu, 1793->1793->1537 MB, 1793 MB goal, 0 MB stacks, 0 MB globals, 8 P
gc 12 @3.360s 2%: 0.027+496+0.010 ms clock, 0.22+0/253/0.77+0.082 ms cpu, 3585->3585->2561 MB, 3585 MB goal, 0 MB stacks, 0 MB globals, 8 P
{"addrs":[":8181"],"diagnostic-addrs":[],"level":"info","msg":"Initializing server. OPA is running on a public (0.0.0.0) network interface. Unless you intend to expose OPA outside of the host, binding to the localhost interface (--addr localhost:8181) is recommended. See https://www.openpolicyagent.org/docs/latest/security/#interface-binding","time":"2024-06-15T18:53:54-04:00"}
{"level":"info","msg":"Shutting down...","time":"2024-06-15T18:54:03-04:00"}
{"level":"info","msg":"Server shutdown.","time":"2024-06-15T18:54:03-04:00"}
--- Wall clock time summary -------------------
  STW sweep termination .........      0.340 ms
  Concurrent mark and scan ......   1544.600 ms
  STW mark termination ..........      5.093 ms
                          Total =   1550.033 ms (1.550 s)
               Longest GC Pause =    505.026 ms

--- CPU time breakdown ------------------------
  STW sweep termination .........      2.708 ms
    Mark/Scan assist time .......      3.100 ms
    Mark/Scan background GC time     583.290 ms
    Mark/Scan idle GC time ......    739.920 ms
  STW mark termination ..........     40.992 ms
                          Total =   1326.310 ms (1.326 s) (Does not include STW phases)

Num threads: 8
Non-matching rows: 3
```

## End notes

There's a lot of useful info in the reports, and it made connecting the dots much easier on the OPA PR I was working on.
For an example head-to-head comparison between the PR's results and the original behavior of the OPA server, check out the PR notes.

Hope you find the Awk script useful if you find yourself doing low-level Golang debugging!

[charles-blog]: https://cdaniels.net/
[godebug-docs]: https://pkg.go.dev/runtime#hdr-Environment_Variables
[opa-pr-6818]: https://github.com/open-policy-agent/opa/pull/6818
[wikipedia-wall-clock-time]: https://en.wikipedia.org/wiki/Elapsed_real_time
