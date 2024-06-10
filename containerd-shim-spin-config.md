# SKIP 00X - Configuring Runtime Options in the `containerd-shim-spin`

Summary: Configuring runtime options in the `containerd-shim-spin`

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

The `containerd-shim-spin` SpinKube executor currently lackd the ability to configure runtime flags for Spin applications. This proposal aims to address this by enabling three types of configurations: Spin execution settings, shim execution settings, and shim execution environment variables. These configurations will be set via Custom Resource Definitions (CRDs) in Kubernetes, specifically extending the SpinExecutor and SpinApp CRDs. This approach will allow operators to easily configure both the application and the platform execution environments.

## Background

SpinKube executors expose various options and arguments in order to configure how an application is executed. Take, for example, the following command:

```sh
spin up --listen "127.0.0.1:3000" --disable-pooling --log-dir /var/logs --runtime-config-file /var/spin/config.toml
```

The previous command tells the Spin runtime to do the following:

- Listen for requests to the Spin app on IP address and port `127.0.0.1:3000`
- Disable Wasmtime's pooling instance allocator
- Use `/var/logs` as the log directory for the stdout and stderr of components
- Use `/var/spin/config.toml` as the runtime configuration file.

Currently, there is no way for configure any of these runtime flags through the `containerd-shim-spin`.

Operators may also wish to configure Wasm runtime environment variables, such as specifying that the traces of a Spin app be emitted to a specific exporter address (analogous to `OTEL_EXPORTER_OTLP_ENDPOINT=http://127.0.0.1:4318 spin up`).

A Runwasi containerd shim does more than execute a Wasm runtime such as Spin or Wasmtime or WasmEdge. It also pulls and prepares Wasm applications. The Shim may also have extra settings it would like to expose to configure how it prepares a Spin application. For example, a user should be able to disable pre-compilation for an application. The environment, such as the `RUST_LOG` level, of the shim should be configurable, too.

In summary, the following are types of configuration that should be supported:

- Wasm runtime specific options (oftentimes in the form of CLI flags: `spin up --listen "127.0.0.1:3000"`)
- Wasm runtime environment variables (such as `OTEL_EXPORTER_OTLP_ENDPOINT=http://127.0.0.1:4318 spin up` )
- Shim execution options (such as disabling pre-compilation)
- Shim execution environment variables (such as `RUST LOG`)

Combining the first two categories and focusing on the Spin runtime, the list can be simplified:

- Spin execution configuration
- Shim execution configuration
- Shim execution environment variables

### Current Workarounds

#### Wasm Runtime Specific Options Workaround

None, unless an environment variable is also exposed for the flag and can be configured with the [Wasm runtime environment variables workaround](#Wasm-runtime-environment-variables-workaround).

These flags could be set in the runtime arguments for the container, such as the following in a Kubernetes PodSpec:

```sh
  containers:
  - name: spin-test
    image: ghcr.io/deislabs/containerd-wasm-shims/examples/spin-rust-hello:v0.10.0
    command: ["/"]
    args: ["--listen", "0.0.0.0:3000"]
```

While the shim currently has access to these arguments from the [`RuntimeContext` args](https://github.com/containerd/runwasi/blob/f5f497c4b21a5d55613095a3ae878c4ef4b83b91/crates/containerd-shim-wasm/src/container/context.rs#L14), the shim does not parse these arguments and would need to be enlightened in a similar manner as the Spin CLI to discern which are for which trigger.

#### Wasm Runtime Environment Variables Workaround

There is no SpinKube native way to configure environment variables on the runtime process. Working around the Spin operator, these environment variables can be configured by setting them in the container spec of a Spin app deployment:

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
  - name: FOO
    value: "bar"
```

#### Shim Execution Environment Variables Workaround

Containerd shims inherit the containerd environment variables. Therefore, a shim environment variable can be configured by setting it on the containerd process. For example, to configure the `RUST_LOG` level:

```sh
RUST_LOG=info containerd --address /tmp/containerd.sock
```

If the process is managed by systemd, you can add an environment variable to the systemd service file. First, stop the service and bring up the service file:

```sh
systemctl stop containerd
systemctl edit containerd
```

At the top of the file, add the environment variable to the service:

```sh
[Service]
Environment="RUST_LOG=info"
```

Finally, restart the containerd service:

```sh
systemctl restart containerd
```

> note, for k3s, edit the k3s service (`systemctl edit k3s`)

## Proposal

As laid out in the [background](#background), this SKIP aims to enable the following categories of configuration:

1. Spin execution configuration
2. Shim execution configuration
3. Shim execution environment variables

The first category discusses how to configure an app's execution, while the later two discuss how to configure the platform.

### User Interface

How does a user set this configuration? As is standard with Kubernetes, configuration should be set in Custom Resource Definitions. Rather than creating a new CRD for configuration, the `SpinExecutor` and `SpinApp` CRDs can be extended to support configuring the platform and app execution, respectively. Users should be able to configure Spin execution next to their app configuration in the `SpinApp` CRD. On the other hand, platform configuration is not done on a per app basis. It could be configured in the `SpinAppExecutor` CRD. This would help users easily see how the platform is configured for the executor their app is using. It would also enable creating multiple executors with distinct configuration.

### Configuring Spin Execution in Container Environment Variables

All Spin execution configuration will be defined as known environment variables with an expected range of values. All trigger configuration keys have the prefix "SPIN_<TRIGGER NAME>". Each executor will define which environment variables it supports. The following is a subset of configuration values that may be supported:

| Key | Spin CLI |  Example Value|
| ---- | ---- | ---- |
| SPIN_HTTP_LISTEN_ADDR | `spin up --listen` | "0.0.0.0:3000" |
| OTEL_EXPORTER_OTLP_ENDPOINT | `OTEL_EXPORTER_OTLP_ENDPOINT=http://123.4.5.6:4318" spin up` | "http://123.4.5.6:4318" |
| SPIN_LOG_DIR | `spin up --log-dir /tmp/log` | "/tmp/log" |
| RUNTIME_CONFIG_FILE | `spin up --runtime-config-file /var/config.toml` | "/var/config.toml" |

These values will be configured as environment variables in the Container spec. At first, this feels unadvisable, since it differs from user's previous understanding of Linux container environment variables. Deploying a Spin app shouldn't be compared to deploying a Linux container. A Linux container has one virtualized environment and listener. In a Linux container context, environment variables are set in the app execution and affect the business logic. On the other hand, a Spin app can have N many triggers and components, each of which has its own virtualized environment. While Spin does support setting environment variables in components, they are static. We have never encouraged the use of `spin up --env`, as it is more of a developer shortcut, since it sets the same set of environment variables in all components of an app. This makes each sandbox bigger than necessary and doesn't emphasize the component model's strengths.

That being said, we should steer Spin developers towards using configuration variables, which are already supported in the `SpinApp` CRD. In documentation, we should clarify the distinction between a Spin app and a Linux container, being sure to specify that container environment variables configure the Spin runtime with environment variables and that Spin Wasm components can only be configured with Spin configuration variables.

Each of these environment variables should be configurable from the SpinApp CRD under `SpinApp.spec.runtimeEnvironment`

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

The Spin Operator will then add each key-value pair to the Deployment of the SpinApp:

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

The `containerd-shim-spin` instance will look for all supported runtime environment variables and configure execution accordingly.

### Configuring Shim Execution in Containerd Config Options

Containerd supports [configuring options for runtimes](https://github.com/containerd/containerd/blob/main/docs/man/containerd-config.toml.5.md#multiple-runtimes), such as the Spin shim. These options are set in the containerd configuration file (`config.toml`), which is where the runtime class manager notified containerd of the existence of the shim. The Runtime Class Manager could also add shim options when setting up the shim. For example, it may make the following additions to a containerd configuration file:

```toml
  # Tell containerd where to find the shim (it parses this as /containerd/path/containerd-shim-spin-v2)
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes."spin"]
  runtime_type = "io.containerd.spin.v2"
  # Add options to each execution of the shim
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes."spin".options]
  Foo = "bar"
  DISABLE_PRECOMPILATION = "true"
```

> For an example implementation, see [gVisor's containerd runtime options configuration documentation](https://gvisor.dev/docs/user_guide/containerd/configuration/).

When a container is executed with CRI, these options are sent to Runwasi in a [`CreateTaskRequest`](https://github.com/containerd/containerd/blob/main/core/runtime/v2/README.md#container-level-shim-configuration). However, right now Runwasi does not parse them. Runwasi should parse the options,and pass them to the shim `Engine` through a new `configure_options` method that should run prior to any other methods. Runwasi may also extract it's own configuration from these containerd runtime options.

The following are Spin shim and Runwasi options that should be configurable:

| Key | Example Value | Scope | 
| ---- | ---- | ---- |
| DISABLE_PRECOMPILATION | "true" | shim |
| GRPC_MAX_WRITE_CHUNK_SIZE_BYTES | "15728640" | [Runwasi](https://github.com/containerd/runwasi/blob/7cc5e5a8025f1508c264dee1faf1c8c57cf645a1/crates/containerd-shim-wasm/src/sandbox/containerd/client.rs#L38) |

Ideally, these options could be set in the `SpinAppExecutor` CRD. Then, the RuntimeClassManager can watch the resource and update and restart containerd as needed.

### Configuring Shim Environment Variables via Containerd Environment

Any environment variables that need to be set on the shim process will need to continue to be set on the containerd process as described in the [Shim execution environment variables](#shim-execution-environment-variables).

## References

- Runwasi issue for tracking containerd config options for shims: https://github.com/containerd/runwasi/issues/573
- `containerd-shim-spin` issue tracking inability to run two Spin apps in the same container due to fixed listen address: https://github.com/spinkube/containerd-shim-spin/issues/52
- `containerd-shim-spin` issue considering which `spin up` flags to support in the shim: https://github.com/spinkube/containerd-shim-spin/issues/130

## Alternatives Considered

### Configuring Spin Execution Alternatives

The following are alternate strategies for configuring the Spin runtime in the `containerd-shim-spin`.

1. Spin execution options could be configured in in Pod annotations; however, it is not granular enough for the case of two Spin apps in a Pod, for example, [listen ports could still conflict](https://github.com/spinkube/containerd-shim-spin/issues/52).

2. As shown in the [runtime configuration workaround](#wasm-runtime-specific-options-workaround), options could be set as container arguments, but setting flags in CRDs is less ergonomic than environment variables, and parsing logic would need to be added to the shim. However, taking this path would potentially enable reusing Spin's reconciling of the mutual exclusiveness of flags.

3. Spin Runtime config (`runtime-config.toml`) could be enhanced to support configuring triggers. Right now, runtime config is used to configure host components. So far, it has been used the describe how to configure the host environment for the guest. That could encompass considering how the host should be triggered to invoke the guest. This may be the better long term solution; however, it will take design to consider the scope of Spin runtime config.

4. Alternatively, there may be places each of these configuration options fit outside of CLI flags. For example, the port for HTTP triggered apps could be configured in the trigger specification in the `spin.toml`. Log directory is already a part of [runtime config](https://github.com/fermyon/spin/blob/b3db535c9edb72278d4db3a201f0ed214e561354/crates/trigger/src/runtime_config.rs#L203-L204) despite the fact that users are more comfortable configuring it through the Spin CLI.

5. Having an extensible way to configure trigger execution will be important to trigger developers. Spin could consider supporting a trigger config to do this.
