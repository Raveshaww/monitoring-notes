# learning_prometheus
Technically, this also counts as prep for an eventual PCA
### Observability Fundamentals
- Intro to Observability
    - Observability is the ability to understand and measure the state of a system
    - `traces` allow you to follow operations as they traverse through various systems and services
        - individual events forming a trace are called `spans` and consist of:
            - start time
            - duration
            - parent ID
    - Prometheus collects and aggregates metrics, not logs or traces