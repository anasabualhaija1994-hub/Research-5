# Intent-based control of a Fischertechnik factory with LLMs and a low-code workflow

Code, data and logs for the paper *Towards Industry 5.0: Intent-Based Cognitive Control in Manufacturing
Systems Utilizing LLMs and Composable Low-Code Architectures*.

An LLM in an n8n workflow translates an Arabic or English chat command into a small JSON intent
(`run`, `home`, `stop`, `status`, `clarify`, `reject`). Three deterministic layers then decide:
the Layer 1 validator in the workflow, the Layer 2 orchestrator service, and the Layer 3 firmware on
four ESP32 stations (high-bay warehouse, grabber, oven with saw, colour sorter) connected over MQTT.

## What is used in the paper

| Path | Role |
|---|---|
| `factory_intent_workflow_.json` | n8n workflow: chat, LLM call (OpenRouter), Layer 1 validator, reply. Import into n8n and set the OpenRouter credential and `BASE` URL in the first code node. |
| `orchestrator_service.py` | Layer 2 orchestrator (MQTT + HTTP API on port 8090). |
| `factory_cli.py` | Small command-line client for the orchestrator (home / run / status / stop). |
| `HBW/HBW_controller_with_MQTT/`, `Grabber/Grabber_controller_with_MQTT/`, `Oven/Oven_controller_with_MQTT/`, `Color Sorter/ColorSorter_controller_with_MQTT/` | Layer 3 station firmware (Arduino, ESP32). Set `WIFI_SSID`, `WIFI_PASSWORD` and the broker address before flashing. |
| `benchmark/` | LLM benchmark: cases, runner, raw results (`results.csv`) and `analyze_results.py`. |
| `benchmark/rule_baseline/` | Rule-based (keyword) parser used as the non-LLM baseline of Table 3, with its per-command results. |
| `benchmark/ablation/` | Prompt ablation of the count rule (Section 5.1): the two modified workflows, the 30 commands, raw results and `analyze_ablation.py`. |
| `stress/` | Stress tests of Layer 1 (`validator_stress.js`) and Layer 2 (`orchestrator_stress.py`). |
| `experiments/` | Physical experiments: runner, protocol, per-repetition MQTT logs, analysis and figure scripts. |
| `logs/` | Orchestrator logs of the jobs and station commands of the experiments (`jobs.csv`, `commands.csv`, `pieces.csv`). |
| `tests/` | Stop-semantics checks with simulated stations (added after the post-experiment review). |
| `experiments/stop_in_preflight.py`, `experiments/analyze_review_check.py`, `experiments/review_check/` | Physical check of the post-experiment fixes, see `experiments/PROTOCOL.md` section 5. |
| `experiments/session2/`, `experiments/analyze_session2.py`, `experiments/replay_watchdog.py`, `tests/test_watchdog.py` | Heartbeat watchdog trials, end-to-end commands of Table 9 and clock-offset measurement, see `experiments/PROTOCOL.md` section 6. |

Not used in the paper (development history, kept for reference): `orchestrator.py` (earlier
stand-alone orchestrator), the `*_standalone` firmware (stations running without MQTT) and the
`*_controller` firmware (intermediate versions before the MQTT integration).

## Versions

- Benchmark: workflow before the Layer 1 hardening (not archived separately). All 2,517 recorded outputs
  were re-validated with the hardened Layer 1 of the experiment version without any decision changing
  (`stress/summary.md`); with the final Layer 1, which always forwards `stop`, one decision changes
  (Qwen3.8 Max, C057, a correct stop that the benchmark version refused because the status query failed).
- Physical experiments (throughput, fault injection, end-to-end): the hardened workflow and the
  orchestrator in `versions/experiments/`, and the station firmware in this repository.
- Fixes from the post-experiment code review: stop during the preflight cancels the job, a Last Will
  fails home jobs too, Layer 1 always forwards `stop`, the preflight requires `hold == "empty"`, and the
  workflow has the deterministic stop-word path. See `stress/summary.md`. The firmware logic is unchanged
  (only the Wi-Fi credentials were replaced by placeholders).
- Physical check of the post-experiment fixes (`experiments/review_check/`): the orchestrator in
  `versions/physical-check/` and the current workflow. Afterwards, the job is assigned under the same lock
  as the state change and a stop reads it under that lock (closes a microsecond window in which a stop
  could see the preflight state without the job); checked with the simulated stations, the stress test and
  the fuzzer. `factory_cli.py` was also fixed: two clients started in the same second shared an MQTT
  client id, and a reconnect re-sent the request.
- Prompt ablation (`benchmark/ablation/`, 27 October 2026): the current workflow (`original`) and two copies
  that differ only in the count rule of the prompt (`no_count`: rule removed; `explicit_count`: rule applied
  only to counts stated as a digit or number word), run on the physical line while idle. The released
  workflow uses the `explicit_count` rule.
  The released workflow was also run on all 140 commands (`benchmark/ablation/results_released_prompt.csv`,
  27-28 October 2026).
- Heartbeat watchdog (28 October 2026): `orchestrator_service.py` fails a job when a station has sent no
  status for 7 s (`WATCHDOG_S`, 0 = off), counted from its last status or the job start. Threshold chosen
  from 7,809 recorded status intervals (longest outside a network fault 6.56 s). Checked with
  `tests/test_watchdog.py`, replayed on all earlier logs (`experiments/replay_watchdog.py`) and on the
  factory in session 2 together with the end-to-end commands of Table 9 (`experiments/session2/`). The
  end-to-end commands of the first session (stops through the model) remain in `experiments/results/`.
- Current files in the repository root: the final version.

## Reproducing the analyses

```
pip install -r requirements.txt
cd benchmark   && python analyze_results.py
cd ../stress   && node validator_stress.js && python orchestrator_stress.py
cd ../experiments && python analyze_physical.py results ../logs
python make_figures.py results ../benchmark/analysis/table_overall.csv figs
python analyze_review_check.py review_check/raw_mqtt.csv
python analyze_session2.py session2
cd ../tests && bash run_scenario.sh stop_in_preflight_run     # needs mosquitto
python test_watchdog.py                                      # no broker needed
cd ../benchmark/rule_baseline && python rule_baseline.py ../cases.json rule_baseline_results.csv
cd ../ablation && python analyze_ablation.py results_original_rerun.csv results_no_count.csv results_explicit_count.csv
```

The physical protocol is described in `experiments/PROTOCOL.md`. Attempts that did not complete are listed with their logged fault in
`experiments/results/analysis/aborted_attempts.csv` and with the root cause identified by the operator (not
logged) in `experiments/results/analysis/aborted_attempts_root_causes.csv` (Table 8 of the paper).

## Security note

The testbed MQTT broker and the orchestrator HTTP API are unauthenticated, and the stations accept
maintenance commands over MQTT. The safety argument of the paper covers faulty or manipulated LLM
output, not attackers on the plant network. Use broker authentication, TLS and topic ACLs outside a
closed lab network.
