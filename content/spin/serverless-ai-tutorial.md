title = "Serverless AI Tutorial"
template = "spin_main"
date = "2023-09-05T09:00:00Z"
enable_shortcodes = true
[extra]
url = "https://github.com/fermyon/developer/blob/main/content/spin/serverless-ai-tutorial.md"

---

AI Inferencing performs well on GPUs. However, GPU infrastructure is both scarce and expensive. This tutorial will show you how to use Fermyon Serverless AI to quickly build advanced AI-enabled serverless applications that can run on Fermyon Cloud. Your applications will benefit from 50 millisecond cold start times and operate 100x faster than other on-demand AI infrastructure services. Take a quick look at the video below, and make sure you sign up here to be one of the first to access the Fermyon [Serverless AI private beta](https://developer.fermyon.com/cloud/serverless-ai).

<iframe width="854" height="480" src="https://www.youtube.com/embed/01oOh3D9cVQ?si=wORKmuOkeFMGYBsQ" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>



TODO create page based on Karthik's JS example here https://www.notion.so/fermyon/Spin-AI-Inferencing-Setup-e86964bf27fe48bdaf68d374d23b0e51

* Create a Spin application with `spin new`
* Use the Serverless AI SDK to perform inferencing

## Tutorial Prerequisites

### Spin 

You will need the latest version of Spin. Please go ahead and [install](./install.md) Spin or [upgrade](./upgrade.md) (to the [latest](https://github.com/fermyon/spin/releases/latest)) before we begin.

### Templates

You will need the latest version of Spin templates. Please go ahead and [update Spin templates](https://developer.fermyon.com/spin/managing-templates) before we begin.

### Plugins

You will need the latest version of Spin plugins. Please go ahead and [update Spin plugins](https://developer.fermyon.com/spin/managing-plugins) before we begin.

## Serverless AI Inferencing With Spin Applications 

Now, let's dive deep into a comprehensive tutorial and unlock your potential to use Fermyon Serverless AI.

## Creating a New Spin Application

{{ tabs "sdk-type" }}

{{ startTab "Rust"}}

<!-- @selectiveCpy -->

```bash
$ spin new 

```

{{ blockEnd }}

{{ startTab "TypeScript" }}

<!-- @selectiveCpy -->

```bash
$ spin new http-js
Enter a name for your new application: sentiment-analysis
Description: A sentiment analysis API that demonstrates using LLM inference and KV stores together
HTTP base: /
HTTP path: /...
```

{{ blockEnd }}

{{ startTab "TinyGo"}}

<!-- @selectiveCpy -->

```bash
$ spin new http-go sentiment-analysis --accept-defaults

```

{{ blockEnd }}
{{ blockEnd }}


## Fetch AI Model

Next, we create a folder and fetch a pre-trained AI model for our application:

<!-- @selectiveCpy -->

```bash
$ cd sentiment-analysis
$ mkdir -p .spin/llms
$ wget https://huggingface.co/TheBloke/Llama-2-13B-chat-GGML/resolve/main/llama-2-13b-chat.ggmlv3.q3_K_L.bin
$ mv llama-2-13b-chat.ggmlv3.q3_K_L.bin .spin/llms/llama2-chat
```
## Application Configuration

Place the following lines into the application's manifest (the `spin.toml` file) within the `[[component]]` section:

```toml
ai_models = ["llama2-chat"]
key_value_stores = ["default"]
```

Note the positioning, of the `ai_models` configuration, shown below:

```toml
[[component]]
id = "sentiment-analysis"
source = "target/spin-http-js.wasm"
exclude_files = ["**/node_modules"]
key_value_stores = ["default"]
ai_models = ["llama2-chat"]
[component.trigger]
route = "/api/..."
[component.build]
command = "npm run build"
watch = ["src/**/*", "package.json", "package-lock.json"]
```

## Source Code

Now let's use the Spin SDK to access the model from our app:

{{ tabs "sdk-type" }}

{{ startTab "Rust"}}

<!-- @nocpy -->

```rust
TODO
```

{{ blockEnd }}

{{ startTab "TypeScript"}}

<!-- @nocpy -->

```typescript
import {
  HandleRequest,
  HttpRequest,
  HttpResponse,
  Llm,
  InferencingModels,
  InferencingOptions,
  Router,
  Kv,
} from "@fermyon/spin-sdk";

interface SentimentAnalysisRequest {
  sentence: string;
}

interface SentimentAnalysisResponse {
  sentiment: "negative" | "neutral" | "positive";
}

const decoder = new TextDecoder();

const PROMPT = `\
You are a bot that generates sentiment analysis responses. Respond with a single positive, negative, or neutral.

Hi, my name is Bob
neutral

I am so happy today
positive

I am so sad today
negative

<SENTENCE>
`;

async function performSentimentAnalysis(request: HttpRequest) {
  // Parse sentence out of request
  let data = request.json() as SentimentAnalysisRequest;
  let sentence = data.sentence;
  console.log("Performing sentiment analysis on: " + sentence);

  // Prepare the KV store
  let kv = Kv.openDefault();

  // If the sentiment of the sentence is already in the KV store, return it
  if (kv.exists(sentence)) {
    console.log("Found sentence in KV store returning cached sentiment");
    return {
      status: 200,
      body: JSON.stringify({
        sentiment: decoder.decode(kv.get(sentence)),
      } as SentimentAnalysisResponse),
    };
  }
  console.log("Sentence not found in KV store");

  // Otherwise, perform sentiment analysis
  console.log("Running inference");
  let options: InferencingOptions = { max_tokens: 10, temperature: 0.5 };
  let inferenceResult = Llm.infer(
    InferencingModels.Llama2Chat,
    PROMPT.replace("<SENTENCE>", sentence),
    options
  );
  console.log(
    `Inference result (${inferenceResult.usage.generatedTokenCount} tokens): ${inferenceResult.text}`
  );
  let sentiment = inferenceResult.text.split(/\s+/)[0]?.trim();

  // Clean up result from inference
  if (
    sentiment === undefined ||
    !["negative", "neutral", "positive"].includes(sentiment)
  ) {
    sentiment = "neutral";
    console.log("Invalid sentiment, marking it as neutral");
  }

  // Cache the result in the KV store
  console.log("Caching sentiment in KV store");
  kv.set(sentence, sentiment);

  return {
    status: 200,
    body: JSON.stringify({
      sentiment,
    } as SentimentAnalysisResponse),
  };
}

let router = Router();

// Map the route to the handler
router.post("/api/sentiment-analysis", async (_, req) => {
  console.log(`${new Date().toISOString()} POST /sentiment-analysis`);
  return await performSentimentAnalysis(req);
});

// Catch all 404 handler
router.all("/api/*", async (_, req) => {
  return {
    status: 404,
    body: "Not found",
  };
});

// Entry point to the Spin handler
export const handleRequest: HandleRequest = async function (
  request: HttpRequest
): Promise<HttpResponse> {
  return await router.handleRequest(request, request);
};
```

{{ blockEnd }}

{{ startTab "TinyGo"}}

<!-- @selectiveCpy -->

```go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"strings"

	spinhttp "github.com/fermyon/spin/sdk/go/http"
	"github.com/fermyon/spin/sdk/go/key_value"
	"github.com/fermyon/spin/sdk/go/llm"
)

type sentimentAnalysisRequest struct {
	Sentence string
}

type sentimentAnalysisResponse struct {
	Sentiment string
}

const prompt = `\
You are a bot that generates sentiment analysis responses. Respond with a single positive, negative, or neutral.

Hi, my name is Bob
neutral

I am so happy today
positive

I am so sad today
negative

<SENTENCE>
`

func init() {
	spinhttp.Handle(func(w http.ResponseWriter, r *http.Request) {
		router := spinhttp.NewRouter()
		router.POST("/sentiment-analysis", performSentimentAnalysis)
		router.ServeHTTP(w, r)
	})
}

func performSentimentAnalysis(w http.ResponseWriter, r *http.Request, ps spinhttp.Params) {
	var req sentimentAnalysisRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}
	fmt.Printf("Performing sentiment analysis on: %q\n", req.Sentence)

	// Open the KV store
	store, err := key_value.Open("default")
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	defer key_value.Close(store)

	// If the sentiment of the sentence is already in the KV store, return it
	exists, err := key_value.Exists(store, req.Sentence)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	if exists {
		fmt.Println("Found sentence in KV store returning cached sentiment")
		value, err := key_value.Get(store, req.Sentence)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		res := sentimentAnalysisResponse{
			Sentiment: string(value),
		}
		if err := json.NewEncoder(w).Encode(res); err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		return
	}
	fmt.Println("Sentence not found in KV store")

	// Otherwise, perform sentiment analysis
	fmt.Println("Running inference")
	params := &llm.InferencingParams{
		MaxTokens:   10,
		Temperature: 0.5,
	}

	result, err := llm.Infer("llama2-chat", strings.Replace(prompt, "<SENTENCE>", req.Sentence, 1), params)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	fmt.Printf("Inference result (%d tokens): %s\n", result.Usage.GeneratedTokenCount, result.Text)

	var sentiment string
	if fields := strings.Fields(result.Text); len(fields) > 0 {
		sentiment = fields[0]
	}

	// Cache the result in the KV store
	fmt.Println("Caching sentiment in KV store")
	key_value.Set(store, req.Sentence, []byte(sentiment))

	res := &sentimentAnalysisResponse{
		Sentiment: sentiment,
	}
	if err := json.NewEncoder(w).Encode(res); err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
}

func main() {}

```

{{ blockEnd }}

{{ blockEnd }}

## Additional Functionality

This application also includes two more components, a key/value explorer and static-fileserver component. Let's quickly go ahead and create those (letting Spin do all of the scaffolding for us).

### Static Fileserver Component For The UI

We use the `spin add` command to add the new `static-fileserver` that we will name `ui`:

<!-- @selectiveCpy -->

```bash
$ spin add static-fileserver
Enter a name for your new component: ui
HTTP path: /static/...
Directory containing the files to serve: assets
```
### Key Value Explorer

For this, we install use a pre-made template by pointing to the templates GitHub repository:

<!-- @selectiveCpy -->

```bash
$ spin templates install --git https://github.com/radu-matei/spin-kv-explorer
```

Then, we again use `spin add` to add the new component. We will name the component  `kv-explorer`):

<!-- @selectiveCpy -->

```bash
$ spin add kv-explorer --accept-defaults
Enter a name for your new application: kv-explorer
```

We create an `assets` directory where we can store files to serve statically (see the `spin.toml` file for more configuration information):

<!-- @selectiveCpy -->

```bash
mkdir assets
```

## Application Manifest

As shown below, the Spin framework has done all of the scaffolding for us:

```toml
spin_manifest_version = "1"
authors = ["tpmccallum <tim.mccallum@fermyon.com>"]
description = "A sentiment analysis API that demonstrates using LLM inference and KV stores together"
name = "sentiment-analysis"
trigger = { type = "http", base = "/" }
version = "0.1.0"

[[component]]
id = "sentiment-analysis"
source = "target/sentiment-analysis.wasm"
exclude_files = ["**/node_modules"]
[component.trigger]
route = "/..."
[component.build]
command = "npm run build"

[[component]]
source = { url = "https://github.com/fermyon/spin-fileserver/releases/download/v0.0.3/spin_static_fs.wasm", digest = "sha256:38bf971900228222f7f6b2ccee5051f399adca58d71692cdfdea98997965fd0d" }
id = "ui"
files = [ { source = "assets", destination = "/" } ]
[component.trigger]
route = "/static/..."

[[component]]
source = { url = "https://github.com/radu-matei/spin-kv-explorer/releases/download/v0.9.0/spin-kv-explorer.wasm", digest = "sha256:07f5f0b8514c14ae5830af0f21674fd28befee33cd7ca58bc0a68103829f2f9c" }
id = "kv-explorer"
# add or remove stores you want to explore here
key_value_stores = ["default"]
[component.trigger]
route = "/internal/kv-explorer/..."
```

## Building and Deploying Your Spin Application

Now let's build and deploy our Spin Application locally. Run the following command to build your application: 

<!-- @selectiveCpy -->

```bash
$ npm install
$ spin build --up
```

## TODO Deploy to Fermyon Cloud

<!-- @selectiveCpy -->

```bash
$ spin cloud deploy TODO
```

## TODO Testing 

<!-- @selectiveCpy -->

```bash
# Create a new POST request TODO
$ curl -X POST localhost:3000/test -H 'Content-Type: application/json' -d '{"todo":"todo"}' -v

TODO
```


## Integrating Custom Domain and Storage

The groundbreaking Fermyon Serverless AI introduces a revolutionary addition to the full-stack developer's arsenal. You can now seamlessly integrate the Fermyon [NoOps SQL Database](https://www.fermyon.com/blog/announcing-noops-sql-db), [Key-Value Storage](https://www.fermyon.com/blog/introducing-fermyon-cloud-key-value-store), and even your [Fermyon Cloud Custom Domains](https://www.fermyon.com/blog/announcing-custom-domains) with the launch of your very own advanced AI-enabled serverless applications.

## Conclusion

We want to get feedback on the Serverless AI API. We are curious about what models you would like to use and what applications you are building using Serverless AI.

## Next Steps

Please feel free to ask questions and also share your thoughts in [our Discord community](https://discord.gg/AAFNfS7NGf).
