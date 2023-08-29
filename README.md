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
### Prometheus Fundamentals
- Use Case
    - does have builtin alerting
- Basics
    - collects metrics via an http endpoint
    - scraped metrics are stored in a time series database to be queried by `promql`
    - designed to monitor data that's `numeric`
        - not:
            - events
            - traces
            - system logs
- Architecture
    - three main parts
        - data retrieval worker 
            - collects via http requests
        - time series database
        - http server 
    - Additionally:
        - exporters run on targets that serve up metrics to be pulled
        - short lived jobs are push-based, and are used for situations where data doesn't live long enough for Prom to pull it
        - Prometheus expects you to hardcode the targets, but you can set up service discovery to dynamically update the list of targets
        - can generate alerts with alertmanager, but doesn't send sms itself. It acts as a trigger
        - prometheus webui / grafana
    - requests are sent to /metrics by default
    - several native exporterss
        - linux
        - windows
        - mysql
        - apache
        - haproxy
    - can track custom app metrics
        - you use client libraries for this
        - supports:
            - go
            - java
            - python
            - ruby
            - rust
    - follows a pull based model
    - push based is more ideal for:
        - event based systems (prom isn't meant for this)
        - short lived jobs
- Installation
    - wget binary
    - untar
    - cd into directory
        - prometheus.yml is the config file
        - prometheus is the executable
        - promtool is a cli tool
    - start with ./prometheus
    - navigate to localhost:9090
- Create systemd unit file
    - `sudo useradd --no-create-home --shell /bin/false prometheus`
    - `sudo mkdir /etc/prometheus`
    - `sudo mkdir /var/lib/prometheus`
    - `sudo chown prometheus:prometheus group /var/lib/prometheus`
    - `sudo chown prometheus:prometheus group /etc/prometheus`
    - download, untar prometheus
    - `sudo cp prometheus /usr/local/bin`
    - `sudo cp promtool /usr/local/bin`
    - `sudo chown prometheus:prometheus /usr/local/bin/promtool`
    - `sudo chown prometheus:prometheus /usr/local/bin/prometheus`
    - `sudo cp -r consoles /etc/prometheus`
    - `sudo cp -r console_libraries /etc/prometheus`
    - update permissions for both of the above, don't forget `-R` to do recursively
    - `sudo cp prometheus.yml /etc/prometheus.yml`
    - update permissions for the above
    - start command is now:
         ``` 
        sudo -u prometheus /usr/local/bin/prometheus \
        --config.file /etc/prometheus/prometheus.yml \
        --storage.tsdb.path /var/lib/prometheus/ \
        --web.console.templates=/etc/prometheus/consoles \
        --web.console.libraries=/etc/prometheus/console_libraries
        ```
    - `sudo vim /etc/systemd/system/prometheus.service`
        ```
        [Unit]
        Description=Prometheus
        Wants=network-online.target
        After=network-online.target
        
        [Service]
        User=prometheus
        Group=prometheus
        Type=simple
        ExecStart=/usr/local/bin/prometheus \
            --config.file /etc/prometheus/prometheus.yml \
            --storage.tsdb.path /var/lib/prometheus/ \
            --web.console.templates=/etc/prometheus/consoles \
            --web.console.libraries=/etc/prometheus/console_libraries

        [Install]
        WantedBy=multi-user.target
        ```
    - `sudo systemctl deamon-reload`
    - `systemctl start prometheus`
    - `systemctl enable prometheus`
- Node Exporter
    - responsible for collecting metrics on a linux host
    - download binary from download site
    - untar
    - cd into directory, run `./node_exporter`
    - default port is `9100`
    - if you curl this, you will get all the metrics presented
- Node exporter systemd service file
    - `sudo cp node_exporter /usr/local/bin`
    - `sudo useradd --no-create-home --shell /bin/false node_exporter`
    - `sudo chown node_exporter:node_exporter /usr/local/bin`
    - `sudo vim /etc/systemd/system/node_exporter.service`
        ```
        [Unit]
        Description=Node Exporter
        Wants=network-online.target
        After=network-online.target
        
        [Service]
        User=node_exporter
        Group=node_exporter
        Type=simple
        ExecStart=/usr/local/bin/node_exporter

        [Install]
        WantedBy=multi-user.target
        ```
    - `sudo systemctl deamon-reload`
    - `systemctl start node_exporter`
    - `systemctl enable node_exporter`
- Configuration
    - configure targets in `/etc/prometheus/prometheus.yml`
    - `global` section applies to all config sections
        - can be overridden in a specific section
    - `scrape_configs` defines targets and configs for metric collections
    - `alerting` is related to `AlertManager`
    - `rule_files` specifies a list of files rules are read from
    - there are additional settings for `remote_read`, `remote_write`, and `storage`
    - default config is http, not https
    - after you make changes to the config settings, you have to restart prometheus
- Authentication / encryption
    - by default, no authentication is required
    - by default, everything is in plaintext
    - node_exporter config: 
        - create certificates
        - define a `config.yml` file
        ```
        tls_server_config:
            cert_file: node_exporter.crt
            key_file: node_exporter.key
        ```
        - update node_exporter process to use that file
            - add `--web.config=config.yml`
        - `sudo mkdir /etc/node_exporter`
        - `mv node_exporter.* /etc/node_exporter`
        - `sudo cp config.yml /etc/node_exporter`
        - `chown -R node_exporter:node_exporter /etc/node_exporter`
        - `sudo systemctl deamon-reload`
        - `sudo systemctl restart node_exporter.service`
    - tls config on prometheus server:
        - scp .crt over
        - change ownership to prometheus user / group
        - edit prometheus.yml to note `tls_config`
    - setting up authentication
        - create a hash for a password
            - use some tool or a programming language
        - go to node exporter config and specify `basic_auth_users`
        - restart the node exporter service
        - go and update the prometheus.yml file and set up `basic_auth`
- Metrics
    - metrics have three properties
        - name
        - labels
        - metric value
    - When a metric is scrapped, it also stores the time the metric was scrapped
        - uses unix timestamps
            - (time since Epoch time, aka Jan 1st 1970)
            - does this to help deal with timezones easily
    - time series
        - stream of timestamped values sharing the same metric and set of labels
    - every metric has two attributes
        - help
            - description of metric
        - type
            - specifies what type of metric (counter, gauge, histogram, summary)
    - metric types
        - counter
            - how many times did something happen
            - can only increase
        - gauge
            - what is the current value of something
            - can go up or down
        - histogram
            - how long or big something is
            - groups observations into configurable bucket sizes
            - useful for:   
                - response time
                - request size
        - summary
            - similar to histograms, but doesn't have to define quantiles ahead of time
    - metric rules
        - metric names specifies a general feature of a system to be measured
        - metric names cannot start with a number
        - colons are reserved only for recording rules
        - may contain ascii letters, numbers, underscores, and colons
    - labels
        - key value pairs
        - allows you to split up a metric by specified criteria
        - metrics can have more than one
        - ascii letters, numbers, and underscores
    - Why labels?
        - make it easier to calculate things as needed if there are too many separate metrics
    - names are technically just labels themselves
    - labels that are surrounded by `__` are considered internal to Prometheus
    - every metric is assigned two labels by default
        - job
        - instance
- Prometheus in Containers
    - pull image
    - configure prometheus.yml
    - expose ports and bind mounts
- Intro to PromTools
    - can be used to
        - check and validate config
            - both rule files and prometheus.yml
        - validate metrics
        - perform queries
        - debugging server
        - performing unit tests against rules
    - can be used to validate before applying changes to a server
- Monitoring Containers
    - can collect things like
        - container metrics via cAdvisor
        - docker engine metrics
            - wont give us metrics specific to a container
    - will need to set up special files for each service
        - i.e. `/etc/docker/daemon.json` for docker
        - use a container to run cAdvisor
### PromQL
- Selectors and Matchers
- Modifiers
- Operators
- Vector Matching
- Aggregation
- Functions
- Subquery
- Histogram / Summary
- Recording Rules
- HTTP API