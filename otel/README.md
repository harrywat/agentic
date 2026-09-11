# Local Copilot OpenTelemetry Demo

This stack receives OpenTelemetry from GitHub Copilot CLI and provides:

- **Prometheus** for metrics
- **Tempo** for traces and conversation details
- **Loki** for OTLP logs
- **Grafana** for dashboards and exploration
- **OpenLIT** for an LLM-focused telemetry view backed by ClickHouse

Tempo, Prometheus, and Loki data is ephemeral. OpenLIT data is stored in named
Docker volumes and survives `docker compose down`; use `docker compose down -v`
to remove it too.

## Start the stack

```bash
cd ../otel
docker compose up -d
docker compose ps
```

Grafana is available at <http://localhost:3000>.

- Username: `admin`
- Password: `admin`
- Dashboard: <http://localhost:3000/d/copilot-cli-overview/github-copilot-cli-overview>

The dashboard and datasources are provisioned from the files under
`grafana/` whenever Grafana starts.

OpenLIT is available at <http://localhost:3001>. Complete its initial account
setup in the browser the first time it starts.

## What OpenLIT collects

The local OpenTelemetry Collector forwards every received OTLP signal to
OpenLIT as well as the Grafana backends:

- Traces, including model calls, tool executions, latency, token usage, cost,
  conversation IDs, and captured message or tool content
- Metrics, including model latency, time to first token, token usage, and tool
  call counts
- Logs from producers that emit OTLP log records

OpenLIT does not independently instrument Copilot CLI in this stack; it receives
a copy of the telemetry sent to ports `4317` and `4318`. Copilot CLI currently
exports conversation content as trace attributes rather than log records, so
those details appear under traces. OpenLIT stores the forwarded data in its
local ClickHouse container.

## Start ngrok

Authenticate once if needed:

```bash
ngrok config add-authtoken YOUR_NGROK_AUTHTOKEN
```

Expose the collector's OTLP/HTTP port and generate a public URL:

```bash
ngrok http 4318
```

Copy the HTTPS `Forwarding` URL printed by ngrok and use it as
`OTEL_EXPORTER_OTLP_ENDPOINT`. The ngrok inspector is normally available at
<http://localhost:4040>.

## Start Copilot CLI with telemetry

Export these variables in the same terminal before starting Copilot CLI:

Sample with public exposed ngrok
```bash
export OTEL_EXPORTER_OTLP_ENDPOINT="https://xxxxxxxxxxxxx.ngrok-free.app"
export OTEL_EXPORTER_OTLP_PROTOCOL="http/protobuf"
export OTEL_SERVICE_NAME="testing-local-cli"
export COPILOT_OTEL_ENABLED="true"
export OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT="true"

copilot
```

Or local only

```bash
export export OTEL_EXPORTER_OTLP_ENDPOINT="http://localhost:4318"
export OTEL_EXPORTER_OTLP_PROTOCOL="http/protobuf"
export OTEL_SERVICE_NAME="testing-local-cli"
export COPILOT_OTEL_ENABLED="true"
export OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT="true"

copilot
```

Message-content capture exports prompts, responses, system instructions, tool
arguments, and tool results. Do not use it with secrets or sensitive content.

## Explore data in Grafana

Open **Explore** and select the datasource matching the query type.

### Tempo: traces and conversations

Find all Copilot CLI traces:

```traceql
{ resource.service.name = "testing-local-cli" }
```

Find a specific Copilot CLI session:

```traceql
{ span.gen_ai.conversation.id = "CONVERSATION_ID" }
```

Open a trace and inspect spans such as:

- `github.copilot.user.message`
- `chat <model>`
- `execute_tool <tool>`
- `permission`

Useful span attributes include:

- `gen_ai.input.messages`
- `gen_ai.output.messages`
- `gen_ai.tool.call.arguments`
- `gen_ai.tool.call.result`
- `gen_ai.usage.input_tokens`
- `gen_ai.usage.output_tokens`
- `github.copilot.cost`

### Prometheus: metrics

Total tool calls:

```promql
sum(github_copilot_tool_call_count_total{service_name="testing-local-cli"})
```

Tool calls by tool:

```promql
sum by (gen_ai_tool_name) (
  github_copilot_tool_call_count_total{service_name="testing-local-cli"}
)
```

Input and output tokens:

```promql
sum by (gen_ai_token_type) (
  gen_ai_client_token_usage_sum{service_name="testing-local-cli"}
)
```

95th-percentile model response latency:

```promql
histogram_quantile(
  0.95,
  sum by (le, gen_ai_response_model) (
    gen_ai_client_operation_duration_seconds_bucket{
      service_name="testing-local-cli"
    }
  )
)
```

95th-percentile time to first token:

```promql
histogram_quantile(
  0.95,
  sum by (le, gen_ai_response_model) (
    gen_ai_client_operation_time_to_first_chunk_seconds_bucket{
      service_name="testing-local-cli"
    }
  )
)
```

Failed tool calls:

```promql
sum by (gen_ai_tool_name) (
  github_copilot_tool_call_count_total{
    service_name="testing-local-cli",
    success="false"
  }
)
```

### Loki: logs

Copilot CLI currently emits its conversation content as trace attributes, not
OTLP log records. If a producer sends logs, query them with:

```logql
{service_name="testing-local-cli"}
```

## Stop or reset

Stop containers while retaining their current writable container layers:

```bash
docker compose stop
```

Restart stopped containers:

```bash
docker compose start
```

Remove and recreate the ephemeral stack:

```bash
docker compose down
docker compose up -d
```