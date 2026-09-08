This repo contains notes, and script experiments for trying to get better output from agents.

As of 2026, the default output from agents can be dense, verbose, repetitive, and a contain a whole range of issues that makes it fatiguing to read.
There's also an element of personal preference to the aesthetic and format of writing that people prefer. So what's here will be biased by my own preferences.

My current theory is that we need

1. Some solid up front direction to the agent for how to write responses.
2. Harness hooks that run deterministic scripts to add backpressure when the output falls below configured thresholds
3. Harness hooks to inject periodic reminders of the response principles to address lost-in-the middle problems

Also we need more benchmarks and evals, not just recomendations on based on what 'feels right'... but this seems like a hard problem.
