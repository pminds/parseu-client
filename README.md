# @productminds/parseu

An Effect-based TypeScript client for the Parseu API, with its API contracts
included in the package.

## Install

Install the compiled SDK directly from this public GitHub Release. No access to
the private application repository or npm authentication is required. Imports
remain `@productminds/parseu`.

For this Effect prerelease, pin the shared Node platform dependency in
`pnpm-workspace.yaml` before installing:

```yaml
overrides:
  "@effect/platform-node-shared": 4.0.0-rc.112
```

```sh
pnpm add https://github.com/pminds/parseu-client/releases/download/v0.1.0/productminds-parseu-0.1.0.tgz effect@4.0.0-rc.112 @effect/platform-node@4.0.0-rc.112
```

## Configure the client

Supply a project API key and your chosen Effect HTTP client. This example uses
the Node.js Undici transport:

```ts
import { ParseuClient } from "@productminds/parseu";
import { Config, Layer } from "effect";
import { NodeHttpClient } from "@effect/platform-node";

const clientLayer = ParseuClient.layerConfig({
  apiKey: Config.redacted("PARSEU_API_KEY"),
}).pipe(Layer.provide(NodeHttpClient.layerUndici));
```

The default API URL is `https://api.parseu.ai`. To use a different server, pass
`apiUrl` as a `URL`:

```ts
import { Redacted } from "effect";

const localClientLayer = ParseuClient.layer({
  apiUrl: new URL("http://localhost:8000"),
  apiKey: Redacted.make("your-project-api-key"),
}).pipe(Layer.provide(NodeHttpClient.layerUndici));
```

`ParseuClient.layer` requires an `HttpClient` service. You choose the transport
when composing the layer; the package does not install one automatically.

## Upload and transcribe an audio file

This complete Node.js example uploads a WAV file, starts a transcription, polls
the job, and retrieves its text. Set `PARSEU_API_KEY` in your environment and
replace `./audio.wav` with your audio file's path.

```ts
import { readFile } from "node:fs/promises";
import { ParseuClient } from "@productminds/parseu";
import { Config, Effect, Layer, Option, Schedule } from "effect";
import { NodeHttpClient } from "@effect/platform-node";

const clientLayer = ParseuClient.layerConfig({
  apiKey: Config.redacted("PARSEU_API_KEY"),
}).pipe(Layer.provide(NodeHttpClient.layerUndici));

const program = Effect.gen(function* () {
  const client = yield* ParseuClient;
  const audio = yield* Effect.promise(() => readFile("./audio.wav"));
  const form = new FormData();
  form.append("file", new Blob([new Uint8Array(audio)], { type: "audio/wav" }), "audio.wav");

  const uploaded = yield* client.upload(form);
  const accepted = yield* client.transcribe({
    input: uploaded.uri,
    language: "auto",
    retention: "default",
  });

  const text = yield* Effect.gen(function* () {
    const job = yield* client.getJob({ jobId: accepted.id });
    if (job.status === "failed" || job.status === "canceled") {
      return yield* Effect.fail(
        new Error(`Transcription ${accepted.id} ended with status ${job.status}`),
      );
    }

    if (job.status === "succeeded") {
      const result = yield* client.getResult({ jobId: accepted.id });

      if (result._tag === "Retained" && "text" in result.body) {
        return Option.some(result.body.text);
      }
      if (result._tag !== "Pending") {
        return yield* Effect.fail(new Error(`Unexpected result: ${result._tag}`));
      }
    }

    return Option.none<string>();
  }).pipe(
    Effect.repeat({
      until: Option.isSome,
      schedule: Schedule.spaced("3 seconds"),
    }),
    Effect.timeout("10 minutes"),
  );

  return Option.getOrThrow(text);
});

export const run = () => Effect.runPromise(program.pipe(Effect.provide(clientLayer)));

// Call and await run() from your application's entry point to get the text.
```

Client methods accept plain objects and construct the schema classes internally.
Uploaded assets use `retention: "default"`;
`retention: "none"` is for direct HTTPS-source transcription.

`getResult` returns an Effect tagged enum (`JobResult`) with these variants:

| `_tag`     | Meaning                                              | Fields                          |
| ---------- | ---------------------------------------------------- | ------------------------------- |
| `Retained` | Completed retained result                            | `jobId`, `body`                 |
| `Zdr`      | Completed transient result requiring acknowledgement | `jobId`, `body`, `resultSha256` |
| `Pending`  | Result recovery is still in progress                 | `jobId`, `body`                 |
| `Deleted`  | Result is deleted or expired                         | `jobId`, `body`                 |

Use `_tag` to narrow results, or import `JobResult` and use its `$match` helper.
The client reads job metadata to distinguish retained and ZDR results; ETag
format does not determine retention. Transcription content lives in
`result.body`. Polling continues if result recovery is still
pending after the job succeeds. `Effect.repeat` polls successful pending responses
every three seconds; request errors and failed jobs stop immediately, and the
ten-minute timeout bounds the wait.

## Zero data retention (ZDR)

For ZDR transcription, submit a directly accessible HTTPS audio URL instead of
uploading the file to Parseu. Set `retention: "none"`. The result is transient:
after you have received and processed it, acknowledge the SHA-256 checksum from
its `resultSha256` field to trigger verified deletion. The client extracts and
validates this checksum from the HTTP ETag internally.

```ts
import { ParseuClient } from "@productminds/parseu";
import { NodeHttpClient } from "@effect/platform-node";
import { Config, Effect, Layer, Option, Schedule } from "effect";

const clientLayer = ParseuClient.layerConfig({
  apiKey: Config.redacted("PARSEU_API_KEY"),
}).pipe(Layer.provide(NodeHttpClient.layerUndici));

const program = Effect.gen(function* () {
  const client = yield* ParseuClient;
  const accepted = yield* client.transcribe({
    input: "https://your-storage.example.com/audio.wav",
    language: "auto",
    retention: "none",
  });

  const completed = yield* Effect.gen(function* () {
    const job = yield* client.getJob({ jobId: accepted.id });
    if (job.status === "failed" || job.status === "canceled") {
      return yield* Effect.fail(
        new Error(`Transcription ${accepted.id} ended with status ${job.status}`),
      );
    }

    if (job.status === "succeeded") {
      const result = yield* client.getResult({ jobId: accepted.id });

      if (result._tag === "Zdr" && "text" in result.body) {
        return Option.some({ text: result.body.text, resultSha256: result.resultSha256 });
      }
      if (result._tag !== "Pending") {
        return yield* Effect.fail(new Error(`Unexpected result: ${result._tag}`));
      }
    }

    return Option.none<{ text: string; resultSha256: string }>();
  }).pipe(
    Effect.repeat({
      until: Option.isSome,
      schedule: Schedule.spaced("3 seconds"),
    }),
    Effect.timeout("10 minutes"),
  );

  const result = Option.getOrThrow(completed);
  const text = result.text;
  // Process or persist the result here before acknowledging receipt.
  const receipt = yield* client.acknowledgeResult({
    jobId: accepted.id,
    resultSha256: result.resultSha256,
  });

  return { text, deletionState: receipt.deletion_state };
});

export const run = () => Effect.runPromise(program.pipe(Effect.provide(clientLayer)));
```

The acknowledgement confirms receipt. Its `deletion_state` can be `"pending"`
while deletion completes or `"deleted"` once verified. To observe completion,
poll `getJob` until `job.result.type === "deleted"`; fail if it becomes
`"deletion_failed"`. Once deleted or expired, the transcript is no longer
retrievable. Deletion applies to Parseu's transient data, not the source file
on your storage server.

`Zdr` results always include a validated `resultSha256`. A missing or invalid
ZDR checksum fails with a schema error. `Retained` results have no checksum
field and do not require acknowledgement.

