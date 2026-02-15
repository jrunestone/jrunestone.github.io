+++
title = "How to configure NextJS Open Telemetry (OTLP) instrumentation in Aspire"
template = "post.html"
date = 2026-02-08
path = "/how-to-configure-nextjs-otlp-telemetry-aspire"
[taxonomies]
tags = ["Aspire", "NextJs", "Open Telemetry"]
+++

Get all the Open Telemetry traces and logs from your NextJS application with `pino` and `next-logger`. Works great with [Aspire's GRPC OTLP endpoints](https://aspire.dev/fundamentals/telemetry/) when the included default NextJS setup just doesn't work.

<!-- toc -->

## What's included
* Export traces and logs.
* Forward logs from NextJS with [next-logger](https://www.npmjs.com/package/next-logger).
* Log with [pino](https://www.npmjs.com/package/pino).

## Create an instrumentation file
Create `instrumentation.ts` in your `src` folder of your NextJS application. This is automatically called by NextJS on startup.

Add the following contents: 

```typescript
import { FetchInstrumentation, registerOTel } from "@vercel/otel"
import { OTLPLogExporter } from "@opentelemetry/exporter-logs-otlp-grpc";
import { BatchLogRecordProcessor } from "@opentelemetry/sdk-logs";
import { AlwaysOnSampler, BatchSpanProcessor } from "@opentelemetry/sdk-trace-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-grpc";
import { OTLPMetricExporter } from "@opentelemetry/exporter-metrics-otlp-grpc";
import { PeriodicExportingMetricReader } from "@opentelemetry/sdk-metrics";
import { PinoInstrumentation } from "@opentelemetry/instrumentation-pino";

export async function register() {
  // all settings for endpoints and log levels etc are defined in your environment variables
  registerOTel({
    instrumentations: [new FetchInstrumentation(), new PinoInstrumentation()],
    spanProcessors: [new BatchSpanProcessor(new OTLPTraceExporter())],
    logRecordProcessors: [new BatchLogRecordProcessor(new OTLPLogExporter())],
    metricReaders: [new PeriodicExportingMetricReader({ exporter: new OTLPMetricExporter() })],
    traceSampler: new AlwaysOnSampler() // could use something like: new TraceIdRatioBasedSampler(0.1)
  });

  await import("pino");
  
  // @ts-expect-error No type definition available.
  await import("next-logger");
}
```

## Required NextJS configuration
The following configuration entry in `next.config.ts` is required to make the `pino` logger work:

```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  serverExternalPackages: ["@opentelemetry/instrumentation-pino", "pino"]
};

export default nextConfig;
```

## Result
You don't have to configure your Aspire AppHost since it will already configure the relevant OTLP environment variables for you.

*NOTE: I haven't gotten metrics to export. I think something more is needed on the nodejs side.*

Logs:
<img src="/images/nextjs-otlp-logs.png" alt="NextJS logs in Aspire">

Traces:
<img src="/images/nextjs-otlp-traces.png" alt="NextJS traces in Aspire">