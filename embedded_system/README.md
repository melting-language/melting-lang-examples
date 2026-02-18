# Embedded system sample

A minimal embedded-style Melt application: a control loop that reads a "sensor" input, updates state, and writes an "actuator" output. It uses only features available in the **embedded build** (no HTTP, MySQL, or GUI).

## Run

From the project root:

```bash
make embedded
./bin/melt-embedded examples/embedded_system/main.melt
```

You can also run it with the full interpreter:

```bash
./bin/melt examples/embedded_system/main.melt
```

## Files

- **main.melt** — Entry point: loads config, runs the loop for `cycleCount` cycles, exits.
- **config.melt** — Configuration: cycle count, paths for sensor and actuator files.
- **loop.melt** — Core logic: `runCycle(state, sensorPath, actuatorPath)` reads sensor, updates state, writes actuator.
- **sensor.txt** — Optional. If present, its contents are read as the sensor value; otherwise `0` is used.

After running, **actuator.txt** will contain the last cycle’s output.

## Extending

On real hardware, replace file I/O with a loadable extension that provides e.g. `gpioRead`, `gpioWrite`, or `uartWrite`, and call those from `loop.melt` instead of `readFile`/`writeFile`.
