# llm-routing

---

This is a sample Apigee proxy to demonstrate the routing capabilities of Apigee
across different LLM providers. In this sample we will use Google VertexAI,
Mistral, and HuggingFace as the LLM providers. Apigee will route requests to one
of the providers based on a custom attribute configured for each distinct API
Key.

## Pre-Requisites

1. [Provision Apigee X](https://cloud.google.com/apigee/docs/api-platform/get-started/provisioning-intro)

2. Configure [external access](https://cloud.google.com/apigee/docs/api-platform/get-started/configure-routing#external-access) for API traffic to your Apigee X instance

3. Enable Vertex AI in your project

   You can do this in the [Cloud Console](https://console.cloud.google.com/apis/library).

4. Create a [HuggingFace Account](https://huggingface.co/) and create an Access
   Token. When [creating](https://huggingface.co/docs/hub/en/security-tokens)
   the token choose 'Read' for the token type.

5. Similar to HuggingFace, create a [Mistral Account](https://console.mistral.ai/) and create an API Key

6. Make sure the following tools are available in your terminal's $PATH (Cloud Shell has these preconfigured)
    - [gcloud SDK](https://cloud.google.com/sdk/docs/install)
    - [apigeecli](https://github.com/apigee/apigeecli)
    - unzip
    - curl
    - jq

Here are the steps we'll be going through:

1. Ensure that Vertex AI is enabled in your Google Cloud project.
2. Provision all the Apigee artifacts - proxy, KVM, product, developer, app, credential.
3. Invoke the APIs.

Let's get started!

---

## Authenticate to Google Cloud

Ensure you have an active Google Cloud account selected in the Cloud shell. Follow the prompts to authenticate.

```sh
gcloud auth login
```

You may see a prompt: _You are already authenticated with gcloud ..._  Proceed anyway.

---

## Set your environment

Navigate to the `llm-route-by-client` directory in the Cloud shell.

```sh
cd llm-route-by-client
```

In that directory, edit the provided sample `env.sh` file, and set the environment variables there.

Click <walkthrough-editor-open-file filePath="llm-route-by-client/env.sh">here</walkthrough-editor-open-file> to open the file in the editor

**PLEASE NOTE:**

 - The env.sh file has a setting for the `VERTEXAI_REGION`.  Vertex AI is not
   available in all regions, and not all Vertex AI capabilities are available in
   all Vertex AI regions. For example, as of this writing, the supported regions
   for the OpenAI-compatible chat completions API are:

   | Geography     | regions                                                                  |
   |---------------|--------------------------------------------------------------------------|
   | United States | us-central1, us-east1, us-east4, us-east5, us-south1, us-west1, us-west4 |
   | Europe        | europe-west1, europe-west4, europe-west9, europe-north1                  |
   | Asia Pacific  | asia-northeast1, asia-southeast1                                         |

   For the latest information, please consult the documentation on [Vertex AI
   locations](https://docs.cloud.google.com/vertex-ai/docs/general/locations#available-regions)



Then, source the `env.sh` file in the Cloud shell.

```sh
source ./env.sh
```

---

## Ensure Vertex AI is enabled

This may have been done for you previously, elsewhere.
Check that the service is enabled:

```sh
gcloud services list --enabled --project "$PROJECT_ID" --format="value(name.basename())" | grep aiplatform
```

If the output list does not include `aiplatform.googleapis.com` and if you have
the permissions to enable services in your project, you can enable Vertex AI using
this gcloud command:

```sh
gcloud services enable aiplatform.googleapis.com --project "$PROJECT_ID"
```


## Deploy Apigee configurations

Next, let's deploy the sample to Apigee. Just run

```bash
./deploy-llm-route-by-client.sh
```

Export the three `APIKEY` variables as mentioned in the command output.

---

## Verification

This setup will have created three distinct registered clients or applications
within Apigee, each with a distinct API Key.  Each one has a different model provider associated to it. To check this data,
run the following commands. The output should show the different model providers registered for each distinct app.


```sh
apigeecli apps get --name "llm-route-by-client-1"   --org "$PROJECT_ID" --token "$TOKEN" | jq  ."[0].attributes[0]" -r
```

You should see `google`.

```sh
apigeecli apps get --name "llm-route-by-client-2"   --org "$PROJECT_ID" --token "$TOKEN" | jq  ."[0].attributes[0]" -r
```
You should see `mistral`.
```sh
apigeecli apps get --name "llm-route-by-client-3"   --org "$PROJECT_ID" --token "$TOKEN" | jq  ."[0].attributes[0]" -r
```
You should see `huggingface`.


The simple name of the model provider isn't enough of course. To route a request to an upstream model, the API Proxy needs this information:
- the API endpoint of the upstream
- the credential to pass to the upstream
- the actual model name to request, at the particular model provider

All of this data is stored in the Apigee KVM, and at request time, the Apigee
proxy performs a lookup, using the model provider as the key, to retrieve those
settings. Then the proxy applies them to the upstream request.

### Testing it

You can test the sample with the curl command shown below, passing different
APIKeys. Before invoking the command in several variations, you may wish to
start a debug session in the Apigee UI. When you have that set up, send the
requests.

First, pass `$APIKEY1` as the value for the `x-apikey` header in the inbound
request. The Apigee proxy will retrieve metadata associated to the registered
client app, and route to the configured model. In the case of `$APIKEY1`, the model is Google.

Run this curl command:

```sh
curl -i --location "https://$APIGEE_HOST/v1/samples/llm-routing/chat/completions" \
--header "Content-Type: application/json" \
--header "x-apikey: $APIKEY1" \
--data '{"messages": [{"role": "user","content": [{"type": "text","text": "Suggest few names for a flower shop"}]}],"max_tokens": 250,"stream": false}'
```

In this request, you are passing the APIKEY created during setup to the
Apigee-managed endpoint. Apigee is configured to verify this key, then apply a
_different_ credential for the upstream service.

In this request, Apigee also retrieves the endpoint for the model provider via a
lookup, and routes to that endpoint.

If you examine the headers in the response, you will see a `selected-model`
header. This tells you which model Apigee selected for the request.

- To use Mistral, Execute the same curl command as above, but provide `$APIKEY2`
  as the api key. In the response, you will see a `selected-model` header
  indicating the use of Mistral.

- To use Hugging Face and the Meta/Llama model, execute the same curl command as
  above, but provide `$APIKEY3` as the api key.  In the response, you will see a
  `selected-model` header indicating the use of Hugging Face.

---

## Conclusion

<walkthrough-conclusion-trophy></walkthrough-conclusion-trophy>

Congratulations! You've successfully deployed Apigee proxy to route calls to
different LLM providers, based on the inbound API Key.

This is an example showing how Apigee can route requests to multiple upstream
systems. In this case, the Apigee proxy used an attribute attached to the API
Key to choose the upstream target. But Apigee can use other data for routing
decisions, including:

- any part of the inbound request, such as headers, query params, url path segments, or payload elements
- an attribute on the API Product, or the Developer who owns the registered app
- the time of day or day of the week
- a dynamically determined load-factor
- prior token usage or consumption
- a weighted random selection for A/B testing
- some combination of the above

<walkthrough-inline-feedback></walkthrough-inline-feedback>

## Cleanup

If you want to clean up the artifacts from this example in your Apigee
Organization, first source your `env.sh` script, and then run the clean up script.

```bash
./clean-llm-route-by-client.sh
```
