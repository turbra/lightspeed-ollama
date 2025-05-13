# 🔧 Configure OpenShift Lightspeed to Use Your Own Ollama Server

This guide walks you through configuring **OpenShift Lightspeed** to connect to a locally hosted **Ollama** server, enabling support for open source LLMs like `qwen3:32b`.

> ✅ **Prerequisites**:
> - OpenShift Lightspeed Operator is already installed and running.
> - A deployment of [Ollama](https://ollama.com) is accessible from within the OpenShift cluster.

---

## 📁 Files Included

This repository includes two required manifests:

- `olsconfig.yaml` – Configures OpenShift Lightspeed to use your Ollama server as a model provider.
- `ollama-credentials-secret.yaml` – Supplies a dummy API key required by the Lightspeed operator, even if your Ollama instance does not use authentication.

---

## 📝 Configuration Steps

### 🔐 Create a Secret for the API Token

Create a Secret named `ollama-credentials` in the `openshift-lightspeed` namespace with a **non-empty base64-encoded string**. This key is required even if your Ollama server does not validate it.

```yaml
# ollama-credentials-secret.yaml

apiVersion: v1
kind: Secret
metadata:
  name: ollama-credentials
  namespace: openshift-lightspeed
type: Opaque
data:
  apitoken: <base64_encoded_dummy_token>
```
You can generate a dummy token like this: 
```bash
echo -n "DUMMY" | base64
```

### ⚙️ Define Your OLSConfig
This manifest registers your Ollama server as an LLM provider using the OpenAI-compatible API endpoint and sets it as the default model provider for Lightspeed.
```yaml
# olsconfig.yaml

apiVersion: ols.openshift.io/v1alpha1
kind: OLSConfig
metadata:
  name: cluster
spec:
  llm:
    providers:
      - credentialsSecretRef:
          name: ollama-credentials
        models:
          - contextWindowSize: 50000
            name: '<your-model-name>'  # e.g., 'qwen3:32b'
            parameters:
              maxTokensForResponse: 30000
        name: ollama-provider
        type: openai
        url: 'http://<your-ollama-host>:<port>/v1'  # e.g., http://ollama.apps.cluster.local:11434/v1
  ols:
    defaultModel: '<your-model-name>'
    defaultProvider: ollama-provider
    logLevel: INFO
```
Replace:
- `<your-model-name>` with the model you’ve pulled into Ollama (e.g., qwen3:32b)
- `<your-ollama-host>:<port>` with the internal hostname or service IP reachable from within your OpenShift cluster

### 🚀 Apply the Configuration
Apply both manifests using `oc` or import them via GitOps tools like ArgoCD or from the Web Console
```bash
oc apply -f ollama-credentials-secret.yaml -n openshift-lightspeed
oc apply -f olsconfig.yaml -n openshift-lightspeed
```
Lightspeed should now be configured to route requests to your Ollama instance.

### ✅ Verification
Check that the `lightspeed-app-server` pod starts without errors and references the correct model/provider.
```bash
oc logs -n openshift-lightspeed deployment/lightspeed-app-server
```
Look for the following log entries:
```bash
INFO: HTTP Request: POST http://<your-ollama-host>:<port>/v1/chat/completions "HTTP/1.1 200 OK"
INFO: LLM connection checked - LLM is ready
```
These confirm:
- Lightspeed was able to reach your Ollama endpoint.
- The model responded successfully.
- The system is ready to handle requests.
If you don’t see these, double-check:
- The URL in your OLSConfig
- Network access to your Ollama server
- That your Ollama deployment has the required model pulled and running


### 🧠 Notes
- Even if Ollama doesn't require authentication, the Lightspeed operator requires a non-empty apitoken secret. Use a dummy string if needed.
- This setup assumes your Ollama server is either exposed internally via a Service or accessible via IP.

---

### 🪟 `contextWindowSize`

* **What it does:**
  Defines the **maximum number of tokens** (words, symbols, punctuation) the model can "see" in a single request — this includes:

  * the prompt (user question),
  * the system instructions,
  * and any historical messages in a conversation (chat history).

* **Why it matters:**
  A larger context window lets the model consider more information, which improves continuity in multi-turn conversations. However, it also consumes more memory and CPU.

* **Example:**
  `contextWindowSize: 50000` means the model can process up to 50,000 tokens in a single request (prompt + response combined), **assuming the model itself supports that size**.

---

### ✍️ `maxTokensForResponse`

* **What it does:**
  Sets the **maximum number of tokens** the model is allowed to generate in a single response.

* **Why it matters:**
  Prevents the model from giving excessively long answers. Also helps avoid timeout issues and keeps resource usage predictable.

* **Example:**
  `maxTokensForResponse: 30000` allows the model to generate up to 30,000 tokens in a single reply — **assuming there’s enough room left in the `contextWindowSize` after the prompt**.

---

### ⚖️ Relationship Between the Two

The model's **total token budget** per request is limited by `contextWindowSize`.

**So:**

```plaintext
contextWindowSize = prompt_tokens + maxTokensForResponse
```

If your prompt takes 10,000 tokens and you’ve set `maxTokensForResponse` to 30,000, the total is 40,000 — which is OK under a 50,000-token context window.

---

### 🔧 How to Tune These

| Use Case                | Suggested Values                        |
| ----------------------- | --------------------------------------- |
| Short Q\&A              | 8,000 context / 1,000 response          |
| Long-form generation    | 16,000+ context / 4,000+ response       |
| Code generation or chat | 32,000–50,000 context / 8k–30k response |
| Limited resources       | Lower both values (e.g., 4k / 1k)       |

> 💡 **Tip:** These values should not exceed the model’s actual limits — for example, `qwen3:32b` must support a 50k context to use `contextWindowSize: 50000`.

