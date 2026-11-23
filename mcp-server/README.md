# mcp-server

An MCP server for testing NVSentinel on a kind cluster. An MCP client such as an AI coding agent can
read what NVSentinel reports, inject a fault into the simulated GPU, and check that NVSentinel cordons
and drains the node, through a small set of checked operations instead of a general `kubectl`.

It is a test tool. The chart does not install it, and no other module depends on it. It speaks MCP
over stdio only and opens no port.

**Results from it prove the pipeline against the simulated hostengine (`nvsentinel-fake-dcgm`), not
real GPU health.**

## Tools

| Tool | Changes the cluster | What it does |
|---|---|---|
| `lab_status` | no | Nodes (cordon, NVSentinel state, failing GPU checks), pods, the workload |
| `list_scenarios` | no | Scenarios, stages, allowed XIDs, log sources |
| `wait_for_stage` | no | Polls until a stage holds, such as `cordoned` |
| `component_logs` | no | The last lines of a NVSentinel component's logs |
| `node_events` | no | Recent events on the GPU node |
| `inject_fault` | yes | Injects XID 95 into the simulated GPU |
| `clear_fault` | yes | Restarts the simulated hostengine, which drops the fault |
| `reset_lab` | yes | Undoes a DCGM outage and an injected fault |
| `run_test_scenario` | yes | Runs a scenario end to end and reports each step |

| Scenario | What it checks |
|---|---|
| `xid95` | Inject XID 95: detected, cordoned, workload evicted, drain succeeded. Clear it: healthy, uncordoned, workload back |
| `dcgm_outage` | Take the hostengine away: the monitor reports it lost DCGM. Restore it: the condition clears |

Every tool's inputs, outputs and annotations are in [docs/tools.md](docs/tools.md), generated from
what the server returns for `tools/list`, with the raw result in [docs/tools.json](docs/tools.json).
A test fails if they no longer match the server; `make docs` regenerates them.

## Guardrails

- Tools that change anything are off unless `NVSENTINEL_MCP_ALLOW_CHANGES=true`.
- Even then, changes need the configured `kind-` context and `kind://` nodes, and inject only into a
  pod in `gpu-operator` that runs nothing but `ghcr.io/nvidia/nvsentinel-fake-dcgm`.
- Every argument is an enum or a bounded integer. No tool takes a context, namespace, pod name, path
  or command.
- Text read from the cluster (logs, events) is capped, cleaned and fenced before the client sees it.
- One change runs at a time, each is logged, and a scenario resets the cluster on failure,
  cancellation or SIGTERM.
- The server uses one kubeconfig, set by the operator, and never reads `~/.kube/config` or
  `KUBECONFIG`. [deploy/rbac.yaml](deploy/rbac.yaml) gives it its own ServiceAccount with only the
  verbs its calls use, and no access to Secrets.

The threat model, and the test behind each control, is in [docs/security.md](docs/security.md).

## Run it

It works against the Tilt dev environment (`make dev-env`, context `kind-nvsentinel`) or the
[fault injection demo](../demos/local-fault-injection-demo/README.md) (context
`kind-nvsentinel-demo`, set `NVSENTINEL_MCP_LAB_CONTEXT`). Both run the simulated hostengine in
`gpu-operator`.

**Never apply `deploy/rbac.yaml` where a real GPU Operator runs.** It allows exec into and delete of
pods in `gpu-operator`, which only holds the simulated hostengine on kind.

```shell
CTX=kind-nvsentinel
KCFG=~/.kube/nvsentinel-lab-mcp.yaml
kubectl --context "$CTX" apply -f mcp-server/deploy/rbac.yaml

# A private kubeconfig with the server's own identity, valid for 8 hours
kubectl --context "$CTX" config view --minify --flatten > "$KCFG"
chmod 600 "$KCFG"
TOKEN=$(kubectl --context "$CTX" -n nvsentinel-mcp create token nvsentinel-mcp --duration=8h)
kubectl --kubeconfig "$KCFG" config set-credentials nvsentinel-mcp --token "$TOKEN"
kubectl --kubeconfig "$KCFG" config set-context "$CTX" --user nvsentinel-mcp
kubectl --kubeconfig "$KCFG" config unset "users.$CTX"   # drop kind's admin credentials

cd mcp-server && go build -o bin/mcp-server .
```

Then register it with your MCP client, for example in a `.mcp.json`:

```json
{
  "mcpServers": {
    "nvsentinel-lab": {
      "command": "mcp-server/bin/mcp-server",
      "env": { "NVSENTINEL_MCP_ALLOW_CHANGES": "true" }
    }
  }
}
```

Leave `NVSENTINEL_MCP_ALLOW_CHANGES` out for a client that should only read.

## Settings

| Variable | Default | What it does |
|---|---|---|
| `NVSENTINEL_MCP_KUBECONFIG` | `~/.kube/nvsentinel-lab-mcp.yaml` | The one kubeconfig it uses. Must be private (`chmod 600`) |
| `NVSENTINEL_MCP_ALLOW_CHANGES` | off | Exactly `true` allows the change tools, on kind only |
| `NVSENTINEL_MCP_LAB_CONTEXT` | `kind-nvsentinel` | The context changes need. Must start with `kind-` |
| `LOG_LEVEL` | `info` | As in the other modules; `--debug` sets `debug`. Logs go to stderr only |

## Tests

```shell
make -C mcp-server lint-test   # no cluster: client-go's fake client and an in-memory MCP session
make -C mcp-server e2e         # the scenarios through the real binary, over MCP, on the kind cluster
```

## What is not tested

- A real GPU, or any real hostengine. The server refuses to inject into one.
- Remediation past cordon and drain on kind.
- The syslog monitor, which a kind node cannot safely feed.
