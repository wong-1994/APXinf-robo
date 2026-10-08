# Qwen-Drive planning

Install APXinf-robo with its `drive` dependencies and the pinned ApxInf CUDA
binding. Keep `planner-sft` under the released model directory. The optional
real-scene example accepts a scene fixture directory containing `scenes.json`,
referenced frame arrays, and `initial-noise.npy`. Robo loads the policy through
`load_policy` and returns a `(50, 3)` trajectory.
For the Thor performance configuration, build the binding with the SM110 AOT
operator bundle as described in the [ApxInf build instructions](../apxinf/doc/qwen-drive-benchmark.md#build-and-load).

## Performance

Direct planning, batch 1, ten flow steps, twelve input frames. Request P50
includes decoded-image preprocessing and the host trajectory.

| Hardware | Precision | Latency | NAVSIM PDM (242 scenes) |
|---|---|---:|---:|
| Jetson AGX Thor | BF16 | 482.60 ms | 85.6786 |

The shared benchmark constructs three cameras with four frames each, 16 history
states, and deterministic noise. It measures resident decoded RGB through the
policy and the returned `[50, 3]` host trajectory. No recorded input files are
required. The historical benchmark image sizes are explicit in the workload;
these are not an accuracy dataset or the official default-resolution protocol.

```sh
python scripts/bench_qwen_drive.py \
  --model-dir /models/Qwen-Drive-1.0-4B --precision bf16 \
  --warmup 10 --samples 30 --seed 0 \
  --out devlocal/qwen-drive-eval/latency.json
```

The same command works in ApxInf. Robo delegates input construction and timing
to the pinned engine while loading the policy through `apxinf_robo.load_policy`.
Without `--tactics`, the engine selects a compatible hardware/toolkit database
under `configs/tuning` when available; otherwise it uses provider defaults.
For controlled comparisons, supply `--tactics` and retain the database identity
and hash in the run evidence.
Lock CPU/GPU/EMC clocks and fan, exclude other compute jobs, and repeat the run
with a second output path. Report the median of all retained samples.

The table retains the previously published recorded-input result. Use the
command above for new measurements.

## Accuracy evaluation

The fixed NAVSIM subset has 242 scenes and a PDM score of 85.6786. Its
trajectories match the accepted padded implementation for 242/242 scenes.
Compare a scene's Robo trajectory with the reference array:

```sh
python examples/qwen_drive_infer.py \
  --model-dir /models/Qwen-Drive-1.0-4B \
  --inputs /data/qwen-drive/public-inputs \
  --scene-index 0 \
  --reference /data/qwen-drive/reference-direct/scene-0-repeat-0.npy \
  --save-actions devlocal/qwen-drive-eval/scene-0-actions.npy \
  --out devlocal/qwen-drive-eval/scene-0.json
```

The output includes `reference_max_abs` and `reference_relative_l2`. Use the
same command for each available scene and score the resulting trajectories
with NAVSIM to reproduce the aggregate PDM result. For the WebSocket entry
point, see the [service example](../examples/README.md#qwen-drive-planning).
