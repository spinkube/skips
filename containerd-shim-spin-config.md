# SKIP 00X - Configuring Runtime Options in the `containerd-shim-spin`

Summary: Configuring Spin runtime options in the `containerd-shim-spin`

Owner: Kate Goldenring <kate.goldenring@fermyon.com> 

Impacted Projects:

- [x] spin-operator
- [ ] `spin kube` plugin
- [x] runtime-class-manager
- [x] containerd-shim-spin
- [ ] Governance
- [ ] Creates a new project

Created: 30/05/2024

## Overview

There is currently no defined way to configure the execution of the
`containerd-shim-spin` SpinKube executor. This proposal aims to address this at
two levels: Spin application execution configuration and Shim execution
configuration. The former is for the lifetime of a Spin application and the
later is for the lifetime of a Spin executor. These configurations will be set
via Custom Resource Definitions (CRDs) in Kubernetes, specifically extending the
SpinApp and SpinAppExecutor CRDs. This approach will allow operators to easily
configure both the application and the platform execution environments.

## Background

Spin exposes various flags and environment variables in order to configure how
an application is executed. Take, for example, the following command:

```sh
OTEL_EXPORTER_OTLP_ENDPOINT=http://127.0.0.1:4318 spin up --listen "127.0.0.1:3000" --disable-pooling --log-dir /var/logs --runtime-config-file /var/spin/config.toml
```

The previous command tells the Spin runtime to do the following:

- Send traces of a Spin app to the exporter at `http://127.0.0.1:4318`
- Listen for requests to the Spin app on IP address and port `127.0.0.1:3000`
- Disable Wasmtime's pooling instance allocator
- Use `/var/logs` as the log directory for the stdout and stderr of components
- Use `/var/spin/config.toml` as the runtime configuration file.

Currently, there is no way for configure any of the runtime flags through the
`containerd-shim-spin` (also referred to as "the shim" in this document). On the
other hand, environment variables can be exposed to the shim as container
environment variables, since `youki` injects the container environment variables
into the shim process before `run_wasi` is executed. For example, the
`OTEL_EXPORTER_OTLP_ENDPOINT` environment variable can be configured in a Spin
application deployment as follows:

```yaml
containers:
- name: foo
  env:
  - name: NODE_IP
    valueFrom:
      fieldRef:
        fieldPath: status.hostIP
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://$(NODE_IP):4318"
```

However, this experience is not SpinKube-native, as there are no fields for
setting environment variables in the `SpinApp` CRD. Furthermore, Kubernetes
users are used to container environment variables being used to configure an
application not the runtime. This points to the need to distinguish between
which environment variables are for the Spin runtime vs the Spin application
components.

The above discussed ways a user may wish to configure the an executor (namely
the shim) for the lifetime of an application's execution. A user may also wish
configuration to last for the lifetime of an executor (the lifetime of the
`SpinAppExecutor` CRD). For example, a user may wish to disable precompilation
for all applications.

In summary, configuration can be broken out into two groups:

- Spin application execution configuration - granular to the `SpinApp` CRD level
  - Spin runtime specific options (oftentimes in the form of CLI flags: `spin up
    --listen "127.0.0.1:3000"`)
  - Spin runtime environment variables (such as
    `OTEL_EXPORTER_OTLP_ENDPOINT=http://127.0.0.1:4318 spin up` )
- Shim execution configuration - granular to the `SpinAppExecutor` level
  - Spin shim execution options (such as disabling pre-compilation)
  - Spin shim execution environment variables (such as `RUST LOG`)

## Terminology

| Term | Definition |
| ---- | ---------- |
| Container runtime | A program that runs and manages the lifecycle of containers, or in this case Spin apps, i.e. Youki. |
| CRD | Custom Resource Definition, a Kubernetes extension mechanism for defining custom resources |
| Environment Variables | Variables that are part of the environment in which a process runs |
| PodSpec | A Kubernetes object that describes the specification of a pod, including its containers, volumes, and other properties |
| Shim | In this document, a shortened reference for the `containerd-shim-spin` containerd shim |
| Spin | A CLI tool for scaffolding, building and running serverless Wasm components |
| SpinApp | A custom resource definition (CRD) in Kubernetes that represents a Spin application |
| SpinAppExecutor | A custom resource definition (CRD) in Kubernetes that represents a Spin executor (such as the Spin containerd shim) |
| SpinKube | A Kubernetes operator for managing Spin applications |
| Spin runtime | A Wasm execution engine that executes Wasm components using Wasmtime |
| Spin Application Variables | Variables defined in a Spin application manifest (in the `[variables]` section) with values that can be dynamically updated by an [application variables provider](https://developer.fermyon.com/spin/v2/dynamic-configuration#application-variables-runtime-configuration). |
| Wasmtime | The WebAssembly runtime that Spin uses to execute WebAssembly components |

## Proposal

As laid out in the [background](#background), this SKIP aims to enable the following categories of configuration:

- Spin application execution configuration
- Executor configuration

The first category pertains to the execution of a single Spin application, while the later discusses how to configure the platform.

### User Interface

How does a user set this configuration? As is standard with Kubernetes,
configuration should be set in Custom Resource Definitions. Rather than creating
a new CRD for configuration, the `SpinAppExecutor` and `SpinApp` CRDs can be
extended to support configuring the platform and application execution,
respectively. Users should be able to configure Spin resources and executions
side-by-side in the `SpinApp` CRD. On the other hand, platform configuration is
not done on a per application basis. It could be configured in the
`SpinAppExecutor` CRD. This would help users easily see how the platform is
configured for the executor their application is using. It would also enable
creating multiple executors with distinct configuration.

### Spin Application Execution Configuration

All Spin application execution configuration will be defined as known
environment variables with an expected range of values. All variables must have
a `SPIN` prefix. Each executor will define which environment variables it
supports. The following is a subset of configuration values that may be
supported:

| Key | Spin CLI |  Example Value|
| ---- | ---- | ---- |
| SPIN_HTTP_LISTEN_ADDR | `spin up --listen` | "0.0.0.0:3000" |
| SPIN_OTEL_EXPORTER_OTLP_ENDPOINT | `OTEL_EXPORTER_OTLP_ENDPOINT=http://123.4.5.6:4318 spin up` | "http://123.4.5.6:4318" |
| SPIN_LOG_DIR | `spin up --log-dir /tmp/log` | "/tmp/log" |
| SPIN_RUNTIME_CONFIG_FILE | `spin up --runtime-config-file /var/config.toml` | "/var/config.toml" |
| SPIN_AWS_DEFAULT_REGION | `AWS_DEFAULT_REGION="us-west-2" spin up` | "us-west-2" |
| SPIN_AWS_ACCESS_KEY_ID | `AWS_ACCESS_KEY_ID="ABC" spin up` | "ABC" |
| SPIN_AWS_SECRET_ACCESS_KEY | `AWS_SECRET_ACCESS_KEY="123" spin up` | "123" |
| SPIN_AWS_SESSION_TOKEN | `AWS_SESSION_TOKEN="token" spin up` | "token" |

These values will be configured as environment variables in the Container spec,
and will be filtered out from all other Container environment variables. All
environment variables that do not have the `SPIN` prefix, will be set in the
environment of the Wasm component instances before execution. The fact that some
container environment variables are configuring the runtime rather than being
set in the application will feel strange for users who are used to configuring
application environment variables in Linux containers. Unfortunately, as
explained in the [alternatives section](#alternatives-considered), there is no
other smooth path to configuring execution at the application level.

> Note: Variables prefixed with `SPIN_VARIABLE` will be set as [Spin application
> variables](https://developer.fermyon.com/spin/v2/variables#application-variables)
> via the Spin environment variable provider as is the experience with the Spin
> CLI.

Each of the Spin application execution environment variables should be
configurable from the SpinApp CRD under `SpinApp.spec.runtimeEnvironment`

```sh
apiVersion: core.spinoperator.dev/v1alpha1
kind: SpinApp
metadata:
  name: myspinapp
spec:
  image: "ghcr.io/myuser:v1"
  replicas: 1
  executor: containerd-shim-spin
  runtimeEnvironment:
  - name: SPIN_HTTP_LISTEN_ADDR
    value: "0.0.0.0:3000"
```

> Note: `runtimeEnvironment` is distinct from `runtimeConfig`

The Spin Operator will then add each key-value pair to the Deployment of the
SpinApp:

```sh
spec
  containers:
  - name: myspinapp
    image: "ghcr.io/myuser:v1"
    env:
    - name: SPIN_HTTP_LISTEN_ADDR
      value: "0.0.0.0:3000"
    ...
```

The `containerd-shim-spin` instance will look for all supported Spin execution
environment variables and configure execution accordingly.

#### Spin Application Environment Variables

As mentioned in the previous section, all environment variables without a `SPIN`
prefix will be forwarded to the environment of each component in the Spin app
upon execution. At first, this may seem unadvisable. While a Linux container
which has one virtualized environment, a Spin app can have N many isolated
components. Setting the environment variables in all of the components seems to
go against the goal of granular sandbox configuration. This is why, in general,
Spin encourages the use of [application
variables](https://developer.fermyon.com/spin/v2/variables#application-variables)
over `spin up --env`, which sets the same set of environment variables in all
components of an app.

However, in order for SpinKube to provide the most Kubernetes native experience,
it should forward these environment variables. First of all, many Kubernetes
libraries expect Kubernetes [cluster information environment
variables](https://kubernetes.io/docs/concepts/containers/container-environment/#cluster-information)
such as `KUBERNETES_SERVICE_HOST` and `KUBERNETES_SERVICE_PORT` to be available
and would be incompatible otherwise. Secondly, the alternative would be to use
[Spin application
variables](https://developer.fermyon.com/spin/v2/variables#application-variables).
However, the variable names must be known and set in the Spin manifest and it
seems strange to mingle the developer  and operator lifecycles by adding a
`SIMPLE_SPINAPP_SERVICE_HOST` variable to a `spin.toml`.

While, by default, these environment variables will be passed to components.
This can be disabled by setting `SPIN_DISABLE_CONTAINER_ENV_VARS`.

#### Spin Shim Executor Configuration

Users may wish to configure settings for the executors at large, beyond the
scope of a single application's execution. Containerd supports [configuring
options for
runtimes](https://github.com/containerd/containerd/blob/main/docs/man/containerd-config.toml.5.md#multiple-runtimes),
such as the Spin shim. These options are set in the containerd configuration
file (`config.toml`), which is where the runtime class manager notifies
containerd of the existence of the shim. The Runtime Class Manager could also
add shim options when setting up the shim. For example, it may make the
following additions to a containerd configuration file:

```toml
  # Tell containerd where to find the shim (it parses this as /containerd/path/containerd-shim-spin-v2)
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes."spin"]
  runtime_type = "io.containerd.spin.v2"
  # Add options to each execution of the shim
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes."spin".options]
  Foo = "bar"
  DISABLE_PRECOMPILATION = "true"
```

> For an example implementation, see [gVisor's containerd runtime options
> configuration
> documentation](https://gvisor.dev/docs/user_guide/containerd/configuration/).

When a container is executed with CRI, these options are sent to Runwasi in a
[`CreateTaskRequest`](https://github.com/containerd/containerd/blob/main/core/runtime/v2/README.md#container-level-shim-configuration).
However, right now Runwasi does not parse them. Runwasi should parse the
options,and pass them to the shim `Engine` through a new `configure_options`
method that should run prior to any other methods. Runwasi may also extract it's
own configuration from these containerd runtime options.

The following are Spin shim and Runwasi options that should be configurable:

| Key | Example Value | Scope | 
| ---- | ---- | ---- |
| DISABLE_PRECOMPILATION | "true" | shim |
| GRPC_MAX_WRITE_CHUNK_SIZE_BYTES | "15728640" | [Runwasi](https://github.com/containerd/runwasi/blob/7cc5e5a8025f1508c264dee1faf1c8c57cf645a1/crates/containerd-shim-wasm/src/sandbox/containerd/client.rs#L38) |

Ideally, these options could be set in the `SpinAppExecutor` CRD. Then, the
RuntimeClassManager can watch the resource and update and restart containerd as
needed.

### Spin Shim Executor Environment Variables

Since containerd shims inherit the environment variables of the calling
containerd process, environment variables that need to be set on the shim
process need to be set on containerd. Configuring containerd environment
variables differs depending on how the service is executed. When directly
invoking containerd, the log level of the shim could be adjusted as follows:

```sh
RUST_LOG=info containerd --address /tmp/containerd.sock
```

If the process is managed by systemd, you can add an environment variable to the
systemd service file. First, stop the service and bring up the service file:

```sh
systemctl stop containerd
systemctl edit containerd
```
> note, for k3s, edit the k3s service (`systemctl edit k3s`)

At the top of the file, add the environment variable to the service:

```sh
[Service]
Environment="RUST_LOG=info"
```

Then the containerd service must be restarted to surface the change.

```sh
systemctl restart containerd
```

For this specific scenario, a log level field could be added to the
`SpinAppExecutor` and the Runtime Class Manager could handle configuring and
restarting containerd.

## References

- Runwasi issue for tracking containerd config options for shims:
  https://github.com/containerd/runwasi/issues/573
- `containerd-shim-spin` issue tracking inability to run two Spin apps in the
  same container due to fixed listen address:
  https://github.com/spinkube/containerd-shim-spin/issues/52
- `containerd-shim-spin` issue considering which `spin up` flags to support in
  the shim: https://github.com/spinkube/containerd-shim-spin/issues/130

## Alternatives Considered

### Configuring Spin Execution Alternatives

The following are alternate strategies for configuring the Spin runtime in the
`containerd-shim-spin`.

1. Spin execution options could be configured in in Pod annotations; however, it
   is not granular enough for the case of two Spin apps in a Pod, for example,
   [listen ports could still
   conflict](https://github.com/spinkube/containerd-shim-spin/issues/52).

2. As shown in the [Spin runtime configuration
   workaround](#wasm-runtime-specific-options-workaround), options could be set
   as container arguments, but setting flags in CRDs is less ergonomic than
   environment variables, and parsing logic would need to be added to the shim.
   However, taking this path would potentially enable reusing Spin's reconciling
   of the mutual exclusiveness of flags.

3. Spin Runtime config (`runtime-config.toml`) could be enhanced to support
   configuring triggers. Right now, the Spin runtime config file is used to
   configure host components. So far, it has been used the describe how to
   configure the host environment for the guest. That could encompass
   considering how the host should be triggered to invoke the guest. This may be
   the better long term solution; however, it will take design to consider the
   scope of Spin runtime config.

4. Alternatively, there may be places each of these configuration options fit
   outside of CLI flags. For example, the port for HTTP triggered apps could be
   configured in the trigger specification in the `spin.toml`. Log directory is
   already a part of [runtime
   config](https://github.com/fermyon/spin/blob/b3db535c9edb72278d4db3a201f0ed214e561354/crates/trigger/src/runtime_config.rs#L203-L204)
   despite the fact that users are more comfortable configuring it through the
   Spin CLI.

5. Having an extensible way to configure trigger execution will be important to
   trigger developers. Spin could consider supporting a trigger config to do
   this.
6. Flags for `spin up` could be exposed as container arguments rather than
   environment variables, such as the following in a Kubernetes PodSpec:
    ```sh
      containers:
      - name: spin-test
        image: ghcr.io/deislabs/containerd-wasm-shims/examples/spin-rust-hello:v0.10.0
        command: ["/"]
        args: ["--listen", "0.0.0.0:3000"]
    ```
    While the shim currently has access to these arguments from the
    [`RuntimeContext`
    args](https://github.com/containerd/runwasi/blob/f5f497c4b21a5d55613095a3ae878c4ef4b83b91/crates/containerd-shim-wasm/src/container/context.rs#L14),
    the shim does not parse these arguments and would need to be enlightened in a
    similar manner as the Spin CLI to discern which are for which trigger. While
    this is approach is technically feasible, the user experience is poor. It
    unnecessarily requires the user to understand the Spin CLI in order to use the
    Spin runtime on Kubernetes.