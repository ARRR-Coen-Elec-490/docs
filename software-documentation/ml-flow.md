# Machine learning layer flowcharts

Proposed supervised-learning design for team review. Green = ML, orange = MCU deployment, light blue = phone, red = unavailable results or safety actions.

## Offline development and model comparison

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontSize": "16px", "primaryColor": "#D9EAD3", "primaryBorderColor": "#477A3C", "primaryTextColor": "#203D1C", "lineColor": "#525252"}, "flowchart": {"wrappingWidth": 300, "rankSpacing": 30}}}%%
flowchart TD
    accTitle: Develop and deploy a supervised model
    accDescr: Define reference labels, split participants, compare suitable models, evaluate the frozen candidate on held-out data, then validate exported behavior and MCU resource use.
    goal["Define prediction target<br/>and correct reference labels"]
    goal --> data["Collect labelled recordings;<br/>split participants into development and test sets"]
    data --> features["Calculate features consistently:<br/>breathing pace, TA ratio,<br/>available HRR, and supported context"]
    features --> task{"What type of output?"}
    task -->|Category| classify["Compare baseline, linear SVM,<br/>logistic regression, and Random Forest"]
    task -->|Number| regress["If a numeric target is defined:<br/>compare baseline, linear regression,<br/>SVR, and Random Forest regressor"]
    classify --> validation["Train and tune on development data;<br/>keep participants separate across folds"]
    regress --> validation
    validation --> select["Select using validation results;<br/>freeze model and test on held-out participants"]
    select --> pass{"Test requirements met?"}
    pass -->|No| revise["Revise using development data;<br/>reserve fresh test evidence for a new iteration"]
    pass -->|Yes| export["Export model and preprocessing to C/C++;<br/>check outputs match Python"]
    export --> device{"Outputs match and MCU memory,<br/>timing, and power limits met?"}
    device -->|No| revise
    device -->|Yes| deploy["Deploy a versioned, validated model<br/>for MCU inference"]
    classDef mcu fill:#FFE5C2,stroke:#A66012,color:#30200F
    classDef caution fill:#F8D4D4,stroke:#B43D3D,color:#641D1D
    class export,device,deploy mcu
    class revise caution
```

Training happens on a development computer or optional cloud compute. The validated model runs on the MCU. A separate reinforcement-learning/Azure feature would need its own agreed design.

## Runtime inference and decision handling

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontSize": "16px", "primaryColor": "#D9EAD3", "primaryBorderColor": "#477A3C", "primaryTextColor": "#203D1C", "lineColor": "#525252"}, "flowchart": {"wrappingWidth": 300, "rankSpacing": 30}}}%%
flowchart TD
    accTitle: Use the embedded model during a session
    accDescr: Use compatible valid features, keep safety independent, abstain when necessary, and recheck freshness before app feedback.
    input["Receive calculated features<br/>and current safety status from MCU"]
    input --> safe{"Safe and fresh enough<br/>for coaching?"}
    safe -->|No| suppress["Suppress coaching;<br/>report safety or freshness issue"]
    safe -->|Yes| valid{"Required features and<br/>model configuration valid?"}
    valid -->|No| unavailable["Report unavailable;<br/>never invent missing values"]
    valid -->|Yes| predict["Apply saved preprocessing<br/>and run embedded model"]
    predict --> result["Return prediction or abstention,<br/>with model version and window time"]
    result --> app["Phone rechecks safety and freshness;<br/>shows status or reviewed feedback"]
    unavailable --> app
    suppress --> app
    classDef mcu fill:#FFE5C2,stroke:#A66012,color:#30200F
    classDef phone fill:#E2EBFA,stroke:#456A9E,color:#172C4D
    classDef danger fill:#F8D4D4,stroke:#B43D3D,color:#641D1D
    class input mcu
    class app phone
    class safe,suppress,unavailable danger
```

When HRR is missing early in a session, use a separately validated reduced-feature model or report unavailable. Safety checks operate independently of ML and cannot be overridden by a prediction.

## Features, labels, and outputs

| Term | Meaning for this design |
| --- | --- |
| Features | Calculated pace, TA ratio, available HRR, and justified profile context |
| Labels | Correct training answers; breathing-pattern categories need a reviewed definition and annotation method |
| Prediction | Model estimate with timing, model version, and validity status |
| Safety inputs | HR, SpO2, and temperature monitored through separate rules |

Possible labels for discussion are thoracic-dominant, abdominal-dominant, and mixed breathing. They are not approved categories or diagnoses. Linear regression requires a separately defined numeric target; it is not compared directly with classifiers as though they solve the same task.

Detailed data fields, model comparisons, evaluation rules, and deployment checks are in the [ML implementation notes](implementation-notes/ml-flow.md). See the [shared open decisions](README.md#decisions-needed-before-implementation) for team review.
