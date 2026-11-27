# Machine learning implementation notes

Status: proposed supervised-learning design. This expands [white-box section 3.5](../../system-architecture/white-box.md#35-machine-learning-layer) and the supplied meeting/Discord notes. No model or label taxonomy has been selected. [Open decisions](../README.md) must be resolved before deployment.

## Offline development and model comparison

If export or on-device tests cause model selection to change, return to development validation. Repeatedly selecting against the same held-out test results contaminates the test set; a revised release needs fresh untouched evaluation evidence. Test success alone does not authorize a model whose C/C++ conversion changes its behavior.

Training runs on a development computer or optional cloud compute, not on the MCU. This first iteration is supervised learning, not reinforcement learning. No Azure service is required merely to train a small SVM or forest. A later RL proposal needs an explicit state, action space, reward, evaluation environment, and reviewed exploration constraints; it is not obtained by uploading sessions to retrain these classifiers.

## Runtime inference and decision handling

Safety has an independent path in the [MCU flow](../mcu-flow.md), including when inference is warming up. Recheck it at the app output boundary because a fault can arise after a prediction. No numerical warning thresholds or medical reference ranges are specified here.

## Dataset and feature contract

The conceptual training row is one valid analysis window with traceable context and ground truth. It is not one BLE packet: transport batching must not redefine the feature window.

| Field group | Proposed fields | Use |
| --- | --- | --- |
| Traceability | Participant pseudonym, session/device ID, protocol, window start/end, feature-extractor version | Splitting, audit, label alignment; IDs are not model features |
| Calculated features | Breathing pace; calibrated TA ratio; HRR1/2/3 when available | Initial candidates; identical definitions in Python and firmware |
| Feature state | Per-feature validity; HRR observed/interpolated/unavailable; quality and elapsed recovery | Determine eligible model variant and prevent missing values becoming zero |
| Profile context | Age, biological sex as defined by the agreed reference source, height, weight | Optional contextual inputs or versioned reference lookup; retain only justified fields |
| Relevant conditions | Optional supported respiratory-condition context | Proposed only; no free-text diagnosis encoding or unsupported lookup extrapolation |
| Safety channels | Timestamped HR, SpO2, body temperature and validity | Independent deterministic monitoring; not initial learned substitutes for safety rules |
| Development candidates | Separate chest/abdomen amplitude summaries and other justified derived features | Evaluate on development data; avoid assuming redundant features improve the model |
| Target and provenance | Reviewed pattern label or numeric target, annotation method, annotator/reference, label time interval | Training truth; keep separate from inputs and model-produced predictions |

Units, categorical encodings, optional-field presence, window boundaries, and reference-table versions must be explicit in the feature schema. Consent/collection procedures and demographic coverage need team review; no claim is made that a public dataset contains all requested channels, context, and labels.

HRR1/2/3 use 60/120/180-second recovery horizons, as described in [the MCU metric definitions](../mcu-flow.md#metric-definitions-and-validity). Early-session windows require a validated no-HRR variant or abstention. Resting-only sessions may never have an eligible exercise-end HRR baseline. Do not train using late-session HRR and then populate earlier inference rows with future information.

Lookup-table context is not a source of ground-truth labels by itself. If labels are simply thresholded from TA ratio, measure whether an ML model improves anything over that explicit rule. Any learned lookup, normalization, or imputation must be fitted only within training folds. An external reference table must be versioned and applicable to the session protocol and population.

## Labels and model comparison

Proposed pattern vocabulary for discussion: `THORACIC_DOMINANT`, `ABDOMINAL_DOMINANT`, and `MIXED`. These are candidates, not approved labels, diagnoses, or proof of anatomical posture. The team must define a repeatable independent annotation/reference method and establish that the sensors support the distinction. Pace is already a measured feature; predicting the same pace from that feature would be circular unless a different future or independently measured target is explicitly defined.

`WARMUP`, `INVALID_INPUT`, `UNSUPPORTED_CONTEXT`, `ABSTAIN`, and `INHIBITED` are processing statuses, not ground-truth pattern classes. Keep them separate from the label enumeration. An SVM decision score is not automatically a probability; send confidence only if the model supports it and calibration has been evaluated.

| Task | Initial candidates | Comparison and constraints |
| --- | --- | --- |
| Categorical pattern classification | Rule/majority baseline, linear SVM, logistic regression, small Random Forest classifier | Macro F1, balanced accuracy, per-class recall, confusion matrix; also abstention coverage and errors on accepted outputs |
| A separately defined continuous outcome | Mean/constant baseline, linear regression, SVR, Random Forest regressor | MAE/RMSE in target units and residual analysis; requires an actual numeric reference target |
| Embedded feasibility for either task | Same frozen pipeline as evaluated in Python | Flash, peak RAM/stack, inference time, battery impact, parity, and acquisition/BLE interference |

Linear regression should not be ranked against classifiers using classification accuracy; it belongs to a separately defined numeric task. Start SVM comparison with a linear kernel; more complex kernels and larger forests can increase deployment cost. Verify exact estimator/converter support for the planned `m2cgen` path instead of assuming all scikit-learn pipelines export automatically. Preprocessing may need explicit C/C++ implementation even if the estimator exports.

Hold out participants, not random overlapping windows. Use grouped validation within development data; all windows/sessions of a held-out participant stay out of training. If too few participants exist, report that limitation rather than claiming population generalization. See [scikit-learn grouped cross-validation](https://scikit-learn.org/stable/modules/cross_validation.html#cross-validation-iterators-for-grouped-data) and [data-leakage guidance](https://scikit-learn.org/stable/common_pitfalls.html#data-leakage).

## Prediction contract and app handoff

A prediction includes `status`, optional `label` or numeric output, optional validated confidence, model/variant version, feature-schema version, source window bounds, prediction time, and quality/safety reasons. The app owns presentation and cue policy. A predicted breathing pattern alone does not specify a safe breathing pace or justify increasing exercise intensity.

Before release, review labelled fixtures, missing-feature and out-of-domain cases, participant-disjoint metrics, Python/firmware parity, resource measurements, and end-to-end timing evidence. Automatic per-session model updates are outside this design; every changed artifact needs versioning and revalidation.
