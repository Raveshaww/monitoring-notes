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
- Introduction
    - queries can return one of four types
        - string
            - currently unused
        - scalar
            - simple numeric floating point value
        - instant vector
            - set of time series containing a single sample for each time series, all sharing the same timestamp
            - includes all labels for that metric
        - range vector
            - set of time series containing a range of data points over time for each time series
- Selectors and Matchers
    - label matchers
        - `=`
            - `node_filesystem_avail_bytes{instance="node1"}`
        - `!=`
            - `node_filesystem_avail_bytes{instance!="node1"}`
        - `=~` is regex
            - return all time series where device starts with "/dev/sda" (i.e. sda2) 
                - `node_filesystem_avail_bytes{instance=~"/dev/sda.*"}`
        - `!~`
    - you can have multiple selectors
        - `node_filesystem_avail_bytes{instance="node1", device!="tmpfs"}`
    - range selector
        - `node_filesystem_avail_bytes{instance="node1"}[2m]`
- Modifiers
    - you can offset the value of a metric with a modifer
    - like when you wanna find out what happened a couple of days ago
    - `node_filesystem_avail_bytes{instance="node1"} offset 5m`
    - `node_filesystem_avail_bytes{instance="node1"} offset 1h5m`
    - to go to a specific point (always unix time):
        - `node_filesystem_avail_bytes{instance="node1"} @1663265188`\
    - these two modifiers can be combined 
    - these also work with range selectors
- Operators
    - arithmetic operators are supported
        - node_memory_Active_bytes / 1024 
    - note that when these are used, the metric name is dropped
    - comparison operators
    - order of precedence, note that most are left-associative 
        - ^
        - *, /, %, atan2
        - +, -
        - ==, !=, <=, <, >=, >
        - and, unless
        - or
    - unless results in a vector consisting of elements on the left side for which there are no elements on the right side
- Vector Matching
    - for example, finding free disk space:
        - node_filesystem_avail_bytes / node_filesystem_size_bytes
    - rules:
        - samples with exactly the same labels get matched together
            - this applies even to things with extra labels
    - there may be certain instances where an operation needs to be performed on 2 vectors with different labels
        - to do this use the `ignoring` keyword
            - `http_errors{code="500} / ignoring(code) http_requests`
    - there is a `on` keyword
        - `http_errors{code="500} / on(method) http_requests`
    - there are two types of vector matching
        - one to one 
            - every element on the vector on the left tries to find a single matching element on the right
        - one to many
            - each vector element on the one side can match with multiple elements on the many side
            - `http_errors{code="500} / on(method) group_left http_requests`
- Aggregation
    - allow you to take an instant vector and aggregate its elements, resulting in a new instant vector with fewer elements
        - kind like functions, i.e. `sum(http_errors)`
    - `by` lets you choose which labels to aggregate along
        - `sum by(path) (http_requests)`
    - `without` is the opposite of `by`
- Functions
    - things like `ciel` (round up to closest integer)
    - `abs`
    - there are also functions that allow you to change types
    - `sort`
    - `rate` can help you figure out how fast something is increasing (useful for graphs)
- Subquery
    - <instant_query> [<range>:<resolution>] [offset <duration>]
        - rate(http_requests_total[1m]) [5m:30s]
- Histogram / Summary
    - histogram
        - bucket sizes can be picked
        - less taxing on client libraries
        - any quantile can be selected
        - prometheus server must calculate quantiles
    - summary
        - quantiles must be defined ahead of time
        - more taxing on client libraries
        - only quantiles predefined in client can be used
        - very minimal server-side cost
- Recording Rules
    - allow Prom to periodically evaluate a promql expression and store the resulting time series generated by them
    - this helps speed up dashboards and provides aggregated results for elsewhere
    - configure in these in a rule file
    - then configure the `prometheus.yml` file to look for that rule file
    - naming rules
        - level:metric_name:operations
            - level indicates the aggregation of the metric by the labels it has
    - all the rules for a specific job should be contained in a single group
- HTTP API
    - basically allows you to use execute queries as well as gather information on alerts and etc
    - sent a post request to `http://<prometheus-ip>/api/v1/query`
### Dashboarding and Visualization
- Expression Browser
    - simple graphs
    - useful for adhoc queries
    - no ability for custom dashboards
- Console Templates
    - allow you to create your own custom html pages using Go templates
    - these are saved to `/etc/prometheus/consoles/`
    - you would navigate to the console like this:
        - `https://9090-port-09e9eca4fca24095.labs.kodekloud.com/consoles/node-stats.html`
    - example file is at 
        - `./example_consoles/node-stats.html`
### Application Instrumentation
- Basics
    - for example, in python code, import the library and create a metric within the code
    - use where you want. For example, create a counter metric and increase it when you'd like
    - the prometheus client has its own http server that will expose a /metric endpoint
        - you can import this into the python code and start it from there
    - you can also do it via a more framework native way, but this requires you to look at the documentation more
- Labels
    - you can add labels in the code without creating a new metric
        - `REQUESTS.labels('/cars', 'get').inc()`
- Best practices   
    - naming
        - metrics should follow `snake_case`
        - the first word of the metric should be used for the app/library it's used for
            - i.e. `postgres_`
        - then the name portion
            - i.e. `postgres_queue_size`
        - then the unit
            - i.e. `postgres_queue_size_bytes`
            - avoids prefixes
    - what to instrument
        - online-serving systems
            - where an immediate response is expected
            - track these:
                - number of queries / requests
                - number of errors
                - latency
                - number of in progress requests
        - offline processing service
            - patch up work
            - track these
                - amount of queued work
                - amount of work in progress
                - rate of processing
                - errors
        - batch jobs
            - will require a pushGateway as it isn't getting continously scraped
            - track these:
                - time processing each stage of job
                - overall runtime
                - last time a job completed
### Service discovery
- file
    - a list of jobs / targets can be imported from a file
    - supports json and yaml files
- re-labeling
    - allows you to classify / filter targets and metrics by rewriting their label set
    - this is specified in `scrape_configs` in `prometheus.yml`
### Push Gateway
- Installation
    - download push gateway
    - untar
    - cd into dir
    - `sudo useradd --no-create-home --shell /bin/false pushgateway`
    - `sudo cp pushgateway /usr/local/bin`
    - `chown pushgateway:pushgateway /usr/local/bin/pushgateway`
    - configure systemd:
        ```
        [Unit]
        Description=Prometheus Pushgateway
        Wants=network-online.target
        After=network-online.target
        
        [Service]
        User=pushgateway
        Group=pushgateway
        Type=simple
        ExecStart=/usr/local/bin/pushgateway

        [Install]
        WantedBy=multi-user.target
        ```
    - restart daemon and service, enable service
    - default is on 9091/metrics
    - configure `prometheus.yml`
        - mostly similar to how you update other targets, but you have to make sure to set `honor_labels: true`
- Pushing Metrics
    - you can send a http request to a pushgateway to get it to send a metric, or you can use the client libs
### Alerting
- Intro
    - conditions are just standard promql expressions
    - config is similar to recording rule
    - the `for` label means that Prom needs to evaluate the expression true for a specified period of time
- Labels and Annotations
    - labels can be added
        - this can provide additional info in the alert's identity
            - like severity
    - annotations are just there for an extra description
    - uses Go's templating language
- Alertmanager architecture
    - alertmanager is responsible for receiving alerts and converting them into notifications
    - flow goes like this:
        - api gets alert and sends it to dispatcher
        - dispatcher puts it into inhibition and can silence certain things, like when you're working on maintenance 
        - alerts are routed to the proper service
- Alertmanager installation
    - download
    - untar
    - cd into dir
    - configure prometheus to use the alert manager under the `alertmanagers` section
- Alertmanager SystemD service
    - add user for alertmanger
    - create `/etc/alertmanager`
    - move `alertmanager.yml` to that dir
    - change permissions on dir
    - make `/var/lib/alertmanager`
    - update perms on dir
    - cp alertmanager to /usr/local/bin
    - cp amtool to /usr/local/bin
    - update perms
    - create systemd service
        ```
        [Unit]
        Description=Alert Manager
        Wants=network-online.target
        After=network-online.target
        
        [Service]
        User=alertmanager
        Group=alertmanager
        Type=simple
        ExecStart=/usr/local/bin/alertmanager.yml \
            --config.file=/etc/alertmanager/alertmanager.yml \
            --storage.path=/var/lib/alertmanager
        Restart=always

        [Install]
        WantedBy=multi-user.target
        ```
    - restart daemon, enable, and start
- Config
    - how the alert manager works is configured in `alertmanager.yml`
        - this includes smtp, receivers, and routes
    - there is a fallback route
    - you can have a route nested in another route
        - this is useful for notifying several things for one issue
    - by default, an alert will operate on the first route it matches unless the `continue: true` flag is set
- Receivers and Notifiers
    - receivers are responsible for taking grouped alerts and producing notifications
    - uses Go templating
### Monitoring k8s
- Intro
    - every host should run a node_exporter
        - yay daemonsets
    - the k8s api can help with service discovery
    - `kube-prometheus-stack` helm chart is the easiest way to install into cluster
- Prom Chart Overview
    - creates the following: 
        - creates 2 statefulsets
            - prometheus server
            - alertmanager
        - creates 3 deployments
            - grafana
            - prometheus-operator
                - manage the lifecycle of the prom instance
            - kube-state-metrics
                - gets data on k8s objects and exposes it
        - creates 1 daemonset
            - node exporter
        - creates 9 services
    - when you wanna access grafana or prometheus, you'll have to set something up as the services are ClusterIP by default
- Additional Scrape Configs
    - two different ways
        - less ideal
            - configure helm values file
                - search for `additional scrape configs`
        - more ideal
            - service monitors
- Service Monitors
    - defines a set of targets for prometheus to monitor and scrape
    - this lets you avoid touching the prometheus configs
    - uses CRD, so it's just another k8s resource
    - kinda feels like an ingress file
- Adding Rules
    - uses a CRD called `prometheusrules`
- Alertmanager rules
    - yet another CRD!