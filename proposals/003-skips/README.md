# SKIP 003 - Configuring Runtime Options in the `containerd-shim-spin`

Summary: Configuring Spin runtime options in the `containerd-shim-spin`

Owner: Kate Goldenring <kate.goldenring@fermyon.com> 

Impacted Projects:

- [ ] spin-operator
- [ ] `spin kube` plugin
- [x] runtime-class-manager
- [x] containerd-shim-spin
- [ ] Governance
- [ ] Creates a new project

Created: 30/05/2024

## Overview

There is currently no defined way to configure the execution of the
`containerd-shim-spin` SpinKube executor. This proposal aims to address this at
two levels: Spin application execution configuration (Spin runtime and
components) and Shim execution (containerd shim) The former is for the lifetime
of a Spin application and the later is for the lifetime of a Spin executor.
These configurations will be set via Custom Resource Definitions (CRDs) in
Kubernetes, specifically extending the SpinApp and SpinAppExecutor CRDs. This
approach will allow operators to easily configure both the application and the
platform execution environments.

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

| Term                       | Definition                                                                                                                                                                                                                                                                   |
|----------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Container runtime          | A program that runs and manages the lifecycle of containers, or in this case Spin apps, i.e. Youki.                                                                                                                                                                          |
| CRD                        | Custom Resource Definition, a Kubernetes extension mechanism for defining custom resources                                                                                                                                                                                   |
| Environment Variables      | Variables that are part of the environment in which a process runs                                                                                                                                                                                                           |
| PodSpec                    | A Kubernetes object that describes the specification of a pod, including its containers, volumes, and other properties                                                                                                                                                       |
| Shim                       | In this document, a shortened reference for the `containerd-shim-spin` containerd shim                                                                                                                                                                                       |
| Spin                       | A CLI tool for scaffolding, building and running serverless Wasm components                                                                                                                                                                                                  |
| SpinApp                    | A custom resource definition (CRD) in Kubernetes that represents a Spin application                                                                                                                                                                                          |
| SpinAppExecutor            | A custom resource definition (CRD) in Kubernetes that represents a Spin executor (such as the Spin containerd shim)                                                                                                                                                          |
| SpinKube                   | A Kubernetes operator for managing Spin applications                                                                                                                                                                                                                         |
| Spin runtime               | A Wasm execution engine that executes Wasm components using Wasmtime                                                                                                                                                                                                         |
| Spin Application Variables | Variables defined in a Spin application manifest (in the `[variables]` section) with values that can be dynamically updated by an [application variables provider](https://developer.fermyon.com/spin/v2/dynamic-configuration#application-variables-runtime-configuration). |
| Wasmtime                   | The WebAssembly runtime that Spin uses to execute WebAssembly components                                                                                                                                                                                                     |

## Proposal

As laid out in the [background](#background), this SKIP aims to enable the
following categories of configuration:

- Spin application execution configuration
- Executor configuration

The first category pertains to the execution of a single Spin application, while
the later discusses how to configure the platform. 

This SKIP focuses on how the `containerd-shim-spin` and `runwasi` will consume
configuration. While this SKIP encourages the use of the SpinKube `SpinApp` and
`SpinAppExecutor` CRDs to set configuration, it does not propose the structure
of the changes to the CRDs to add configuration. That should be proposed in a
future SKIP.

### Spin Application Execution Configuration

Spin application execution can be configured via container environment
variables, which are exposed to the shim. The following is a subset of
configuration values that may be supported:

| Key                         | Spin CLI                                                    | Example Value           |
|-----------------------------|-------------------------------------------------------------|-------------------------|
| SPIN_HTTP_LISTEN_ADDR       | `spin up --listen`                                          | "0.0.0.0:3000"          |
| OTEL_EXPORTER_OTLP_ENDPOINT | `OTEL_EXPORTER_OTLP_ENDPOINT=http://123.4.5.6:4318 spin up` | "http://123.4.5.6:4318" |
| SPIN_LOG_DIR                | `spin up --log-dir /tmp/log`                                | "/tmp/log"              |
| SPIN_RUNTIME_CONFIG_FILE    | `spin up --runtime-config-file /var/config.toml`            | "/var/config.toml"      |
| AWS_DEFAULT_REGION          | `AWS_DEFAULT_REGION="us-west-2" spin up`                    | "us-west-2"             |
| AWS_ACCESS_KEY_ID           | `AWS_ACCESS_KEY_ID="ABC" spin up`                           | "ABC"                   |
| AWS_SECRET_ACCESS_KEY       | `AWS_SECRET_ACCESS_KEY="123" spin up`                       | "123"                   |
| AWS_SESSION_TOKEN           | `AWS_SESSION_TOKEN="token" spin up`                         | "token"                 |

These values will be configured as environment variables in the Container spec.
For example, the following sets the listen address for the HTTP trigger:

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

The `containerd-shim-spin` instance will look for execution environment
variables and configure execution accordingly.

At first, this feels strange, since it differs from user's previous
understanding of Linux container environment variables. Deploying a Spin app
should not be compared to deploying a Linux container. A Linux container has one
virtualized environment and listener. In a Linux container context, environment
variables are set in the app execution and affect the business logic. On the
other hand, a Spin app can have N many triggers and components. The Wasm runtime
embedded in Spin (Wasmtime) will isolate each component in its own virtualized
environment upon execution. While Spin does support setting environment
variables in components, they are static. While the Spin CLI does support
setting the same set of environment variables in all components of an app (`spin
up --env`), it is more of a developer shortcut and not advised. It makes each
sandbox bigger than necessary and doesn't emphasize the component model's
strengths. See the [Spin Application Variables
section](#spin-application-variables) for a discussion on how variables will be
exposed to application components.

#### Spin Application Variables

Instead of configuring their applications with environment variables, SpinKube
users should use [Spin application
variables](https://developer.fermyon.com/spin/v2/variables#application-variables).

These variables are declared in a Spin application manifest (`spin.toml`) and
their values can be configured in the [`SpinApp`
CRD](https://www.spinkube.dev/docs/spin-operator/reference/spin-app/#spinappspecvariablesindex)
if using the environment variable provider. Application variables can also be
dynamically updated using [Azure Key
Vault](https://developer.fermyon.com/spin/v2/dynamic-configuration#azure-key-vault-application-variable-provider)
or
[Vault](https://developer.fermyon.com/spin/v2/dynamic-configuration#azure-key-vault-application-variable-provider).

However, requiring that all variables be set by an administrator is likely too
limiting for Kubernetes contexts where there are some cases in which the
scheduler or webhooks inject environment variables into containers that
components wish to access. For example, by default Kubernetes injects [cluster
information environment
variables](https://kubernetes.io/docs/concepts/containers/container-environment/#cluster-information)
such as `KUBERNETES_SERVICE_HOST` and `KUBERNETES_SERVICE_PORT` into each
container. Many Kubernetes libraries expect these to be available and would be
incompatible otherwise. However, the shim should not simply inject all container
environment variables into every component's environment. Instead, the shim will
expose environment variables as application variable, by resetting them in the
environment with the `SPIN_VARIABLE` prefix. It will only do this the
application variables listed in the Spin manifest (`spin.toml`). This means that
Spin components can get access to these variables if specifically configured in
the Spin manifest to have access. For example, say a a component wants access to
the `KUBERNETES_SERVICE_ADDRESS` container environment variable, it would
configure this in the Spin.toml as follows:

```toml
[variables]
kubernetes_service_address = { required = true }

[component.example]
source = "target/wasm32-wasi/release/example.wasm"
allowed_outbound_hosts = []
[component.example.variables]
kubernetes_service_address = "{{ kubernetes_service_address }}"
```

Then, it can be accessed in the Spin app using the variables SDK:

```rust
let addr = variables::get("kubernetes_service_address")?;
```

In documentation, we should clarify that Spin Wasm components can only be
configured with Spin application variables. The variable names must be known and
set in the Spin manifest, as in the above example.

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

| Key                             | Example Value | Scope                                                                                                                                                           |
|---------------------------------|---------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| DISABLE_PRECOMPILATION          | "true"        | containerd-shim-spin or [Runwasi](https://github.com/containerd/runwasi/blob/7cc5e5a8025f1508c264dee1faf1c8c57cf645a1/crates/containerd-shim-wasm/src/sandbox/containerd/client.rs#L38)                                                                                                                                                            |
| GRPC_MAX_WRITE_CHUNK_SIZE_BYTES | "15728640"    | [Runwasi](https://github.com/containerd/runwasi/blob/7cc5e5a8025f1508c264dee1faf1c8c57cf645a1/crates/containerd-shim-wasm/src/sandbox/containerd/client.rs#L38) |

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

To automate the above steps for a SpinKube user, a log level field could be
added to the `SpinAppExecutor` and the Runtime Class Manager could handle
configuring and restarting containerd.

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
    the shim does not parse these arguments and would need to be enlightened in
    a similar manner as the Spin CLI to discern which are for which trigger.
    While this is approach is technically feasible, the user experience is poor.
    It unnecessarily requires the user to understand the Spin CLI in order to
    use the Spin runtime on Kubernetes.
7. Instead of setting container environment variables as Spin application
   variables, all container   environment variables could be injected into all
   components of the Spin application by default. As explained in the [Spin
   Application Variables](#spin-application-variables), this expands the Wasm
   sandbox and does not fit with the objectives of the component model.
