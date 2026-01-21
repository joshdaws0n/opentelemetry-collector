# OpenTelemetry Collector Stack

A centralized OpenTelemetry Collector deployment that receives telemetry data (traces, metrics, and logs) via OTLP and exports it to Grafana Cloud.

## Overview

This project runs an OpenTelemetry Collector Contrib distribution configured to:
- Receive telemetry data via OTLP (gRPC and HTTP protocols)
- Process data with batching and memory limiting
- Export all telemetry to Grafana Cloud
- Provide health checks and diagnostic endpoints

## Architecture

### Receivers
- **OTLP gRPC**: Port 4317 (default)
- **OTLP HTTP**: Port 4318 (configured at `0.0.0.0:4318`)

### Processors
- **Batch Processor**: Batches telemetry data for efficient export
- **Memory Limiter**: Prevents memory overload (80% limit, 15% spike allowance)

### Exporters
- **Grafana Cloud**: Exports traces, metrics, and logs via OTLP HTTP
- **Debug**: Basic verbosity logging for troubleshooting

### Extensions
- **Health Check**: Service health monitoring
- **pprof**: Performance profiling (port 1888)
- **zpages**: Diagnostic pages (port 55679)
- **Basic Auth**: Authentication for Grafana Cloud

## Configuration

### Environment Variables

The following environment variables must be configured in Railway:

| Variable | Description | Example |
|----------|-------------|---------|
| `GRAFANA_CLOUD_OTLP_ENDPOINT` | Your Grafana Cloud OTLP endpoint | `https://otlp-gateway-prod-us-east-0.grafana.net/otlp` |
| `GRAFANA_CLOUD_INSTANCE_ID` | Your Grafana Cloud instance ID | `123456` |
| `GRAFANA_CLOUD_API_KEY` | Your Grafana Cloud API key | `glc_xxxxxxxxxxxxx` |

### Getting Grafana Cloud Credentials

1. Log in to your Grafana Cloud account
2. Navigate to **Connections** → **Add new connection** → **OpenTelemetry**
3. Copy your OTLP endpoint, instance ID, and generate an API token

## Deployment on Railway

### Initial Setup

1. **Create a new project in Railway**
   - Connect your GitHub repository
   - Railway will automatically detect the [`Dockerfile`](Dockerfile:1)

2. **Configure environment variables**
   - Go to your service settings in Railway
   - Add the three required environment variables listed above

3. **Deploy**
   - Railway will automatically build and deploy the Docker container
   - The service will be available on Railway's internal network

### Continuous Deployment

- **Automatic deployments**: Any commit pushed to the `main` branch will trigger a new deployment
- **Build process**: Railway builds the Docker image using the [`Dockerfile`](Dockerfile:1)
- **Configuration**: The collector uses [`otel-collector-config.yaml`](otel-collector-config.yaml:1)

### Exposed Ports

The following ports are exposed by the service:

| Port | Service | Description |
|------|---------|-------------|
| 4317 | OTLP gRPC | Primary receiver for OTLP over gRPC |
| 4318 | OTLP HTTP | Primary receiver for OTLP over HTTP |
| 1888 | pprof | Performance profiling endpoint |
| 8888 | Prometheus | Metrics endpoint (if enabled) |
| 8889 | Prometheus | Metrics endpoint (if enabled) |
| 13133 | Health Check | Health check endpoint |
| 55679 | zpages | Diagnostic pages |

## Usage

### Sending Telemetry Data

Configure your applications to send telemetry to the collector's Railway URL:

#### Using OTLP gRPC (Port 4317)
```bash
export OTEL_EXPORTER_OTLP_ENDPOINT="https://your-railway-service.railway.app:4317"
```

#### Using OTLP HTTP (Port 4318)
```bash
export OTEL_EXPORTER_OTLP_ENDPOINT="https://your-railway-service.railway.app:4318"
export OTEL_EXPORTER_OTLP_PROTOCOL="http/protobuf"
```

### Example: Python Application

```python
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

trace.set_tracer_provider(TracerProvider())
otlp_exporter = OTLPSpanExporter(
    endpoint="https://your-railway-service.railway.app:4318/v1/traces"
)
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(otlp_exporter)
)
```

## Health Checks

### Health Check Endpoint
```bash
curl https://your-railway-service.railway.app:13133
```

### zpages Diagnostics
Access diagnostic information at:
```
https://your-railway-service.railway.app:55679/debug/tracez
https://your-railway-service.railway.app:55679/debug/pipelinez
```

## Local Development

### Running Locally with Docker

1. **Set environment variables**:
   ```bash
   export GRAFANA_CLOUD_OTLP_ENDPOINT="https://otlp-gateway-prod-us-east-0.grafana.net/otlp"
   export GRAFANA_CLOUD_INSTANCE_ID="your-instance-id"
   export GRAFANA_CLOUD_API_KEY="your-api-key"
   ```

2. **Build the Docker image**:
   ```bash
   docker build -t otel-collector .
   ```

3. **Run the container**:
   ```bash
   docker run -p 4317:4317 -p 4318:4318 -p 13133:13133 \
     -e GRAFANA_CLOUD_OTLP_ENDPOINT \
     -e GRAFANA_CLOUD_INSTANCE_ID \
     -e GRAFANA_CLOUD_API_KEY \
     otel-collector
   ```

4. **Test the collector**:
   ```bash
   curl -X POST http://localhost:4318/v1/traces \
     -H "Content-Type: application/json" \
     -d '{"resourceSpans":[]}'
   ```

## Troubleshooting

### Collector Not Receiving Data
- Verify the OTLP endpoint URL in your application configuration
- Check that ports 4317 (gRPC) or 4318 (HTTP) are accessible
- Review Railway logs for connection errors

### Data Not Appearing in Grafana Cloud
- Verify environment variables are set correctly in Railway
- Check the `GRAFANA_CLOUD_OTLP_ENDPOINT` format
- Ensure the API key has the correct permissions
- Review collector logs for authentication errors

### Memory Issues
- The memory limiter is configured to use 80% of available memory
- Adjust `limit_percentage` in [`otel-collector-config.yaml`](otel-collector-config.yaml:22) if needed
- Monitor memory usage via Railway metrics

### Viewing Logs
```bash
# In Railway dashboard, go to your service and click "View Logs"
# Or use Railway CLI:
railway logs
```

## Configuration Files

- [`otel-collector-config.yaml`](otel-collector-config.yaml:1) - OpenTelemetry Collector configuration
- [`Dockerfile`](Dockerfile:1) - Container image definition

## Version

- **OpenTelemetry Collector Contrib**: v0.144.0

## License

See [`LICENSE`](LICENSE:1) file for details.

## Resources

- [OpenTelemetry Collector Documentation](https://opentelemetry.io/docs/collector/)
- [Grafana Cloud OTLP Documentation](https://grafana.com/docs/grafana-cloud/send-data/otlp/)
- [Railway Documentation](https://docs.railway.app/)
