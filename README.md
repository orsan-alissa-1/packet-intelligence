# PI — Packet Intelligence

PI is an ongoing network capture analysis project. The aim is to help investigate suspicious traffic by combining packet inspection, network-flow analysis, and machine-learning experiments in one application.

The current work focuses on preparing capture data, checking the features extracted with Zeek, and evaluating models before integrating their predictions into the investigation workflow.

**Status:** Under development. This repository currently contains a project overview. Source code and the technical report are not published yet.

## Dataset and Scale

The current experiments use IoT-23 v2. The dataset inventory and the eligible modelling cohort represent different stages of preparation.

| Measurement | Recorded scale |
| --- | ---: |
| Scenarios inventoried | 28 |
| Publisher flow-log rows, across all protocols | 336,741,519 |
| Indexed packet captures | 59.50 GiB |
| Label-log inventory | 45.02 GiB |
| Eligible flows reported in the modelling cohort | 179,841,465 |
| Captures represented in that cohort | 12 |

The inventory sizes describe the indexed source material, not the amount downloaded or held in memory at once. The full source-row count is also not a training-set count. Protocol filtering, feature checks, label reconciliation, and capture eligibility determine which flows can enter an experiment.

Captures `34-1`, `49-1`, and `52-1` were excluded from the modelling cohort by the label-coverage gate.

## Capture Processing

A central part of the project is checking whether features generated from a packet capture agree with the inputs used during model development.

The preparation process involves:

1. Recording source locations, file sizes, and integrity information.
2. Processing captures through the offline Zeek worker.
3. Matching extracted connections to publisher labels using connection identifiers and timestamps.
4. Checking label coverage and excluding ambiguous or unsupported records.
5. Applying the same versioned feature transformation to batch data and runtime connections.
6. Checking flow counts against earlier preparation results.

Pilot checks included `CTU-Honeypot-Capture-4-1`, `CTU-IoT-Malware-Capture-20-1`, and `CTU-IoT-Malware-Capture-42-1`. Later preparation runs checked additional captures against the earlier flow counts.

## Flow Features

The current flow contract, `iot23-zeek-flow-v1`, defines seven fields.

| Field | Description |
| --- | --- |
| `protocol` | Transport protocol: TCP or UDP |
| `orig_pkts` | Packets sent by the connection originator |
| `resp_pkts` | Packets sent by the responder |
| `conn_state` | Connection state recorded by Zeek |
| `duration` | Connection duration |
| `orig_bytes` | Originator payload bytes |
| `resp_bytes` | Responder payload bytes |

The contract checks packet counts, numeric bounds, connection states, and missing values. Optional quantities remain missing when unavailable.

IP addresses and timestamps are used for label reconciliation rather than as classifier inputs. Feature comparisons start with protocol and packet counts, then add connection state and the optional duration and byte fields.

## Experimental Design

The evaluation separates training, model selection, threshold calibration, and internal testing.

Whole captures are assigned to partitions, with publisher-defined groups kept together. This reduces overlap between development and evaluation data, although it does not establish independent physical-device identities for every capture.

The recorded assignments include:

| Partition | Publisher groups |
| --- | --- |
| Training | Mirai, Okiru, IRCBot, Hakai, Torii, Trojan, and a benign honeypot capture |
| Validation | Hajime and Muhstik |
| Calibration | Hide and Seek and Gafgyt |
| Internal test | Kenjiro and a separate benign honeypot capture |

Unknown-label scenarios are kept outside supervised fitting.

The comparison plan includes an always-benign baseline, logistic regression, LightGBM, and XGBoost. Model selection considers:

- Capture-macro recall, so large captures do not determine the result alone.
- Pooled precision.
- Average precision.
- False alerts per 10,000 benign flows.
- Scored-flow coverage.

The operating budget is 10 false alerts per 10,000 benign flows. Per-capture checks also require enough benign records to make that comparison meaningful. Thresholds are to be frozen using calibration data before internal testing.

## Development Findings

The validation experiments showed why ranking quality and performance at a usable alert threshold need separate checks.

One earlier challenger reached an average precision of 0.9982 but had zero capture-macro recall at the specified false-alert budget. Its ranking score alone was therefore insufficient for selecting an operating point.

A later flow-only comparison reported 98.03% capture-macro recall. Its per-capture results still showed a problem:

| Validation capture | Malicious flows detected | False alerts per 10,000 benign flows |
| --- | ---: | ---: |
| `3-1` | 145,598 / 151,567 | 24.0312 |
| `9-1` | 6,355,745 / 6,355,745 | 3.0170 |

Capture `3-1` exceeded the budget of 10. These are development results, not final test results or evidence of deployment readiness.

The context-feature variant also performed substantially worse in that comparison. Further work is needed to understand the difference and establish whether the selected pipeline can meet the operating criteria consistently.

## Application Plan

The intended application brings capture inspection and experimental scoring into a single investigation workflow.

| Component | Intended role |
| --- | --- |
| Python and FastAPI | Backend API and analysis coordination |
| React and TypeScript | Investigation interface |
| TShark | Packet-level inspection |
| Zeek | Connection logs and flow features |
| Suricata | Signature-based findings |
| Docker workers | Isolated execution of capture-analysis tools |
| SQLite and Alembic | Application records and schema migrations |
| DuckDB and Parquet | Larger analytical tables and feature processing |

This table describes the intended architecture. It is not a claim that every component is complete or integrated.

## Next Work

- Replay the selected feature and scoring pipeline to check reproducibility.
- Calibrate and freeze its decision threshold.
- Apply the locked evaluation criteria before opening the internal test.
- Report per-capture results, exclusions, and coverage alongside pooled metrics.
- Integrate qualified scoring into the capture-investigation workflow.
- Complete the interface, access controls, and reporting.
- Publish the source and technical documentation after review.

## Limitations

The current evidence concerns IoT-23 experiments. No independent external benchmark has qualified the model for use on other traffic sources.

The dataset's published labels and PI's newly extracted flows must be reconciled; matching them is part of the work, not an assumption. Excluded captures, uncertain labels, and unscored flows affect how results should be interpreted.

PI remains under development. Final internal-test performance and the complete application workflow are not established by the results presented here.

## Dataset Reference

[IoT-23 v2 — publisher documentation](https://mcfp.felk.cvut.cz/publicDatasets/IoT-23-Dataset-v2/README.html)

The captures and labels come from the dataset publisher. This repository describes my processing and evaluation work.

## Author

Orsan Al Issa  
BSc Cybersecurity Engineering  
Abu Dhabi University
