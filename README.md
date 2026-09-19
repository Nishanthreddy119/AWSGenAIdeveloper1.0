# AWSGenAIdeveloper
# BLA4
# Agentic AI on AWS: Troubleshooting Guide
# Linkedin Activity: https://lnkd.in/p/gucfqvyH , https://lnkd.in/p/gT-6tB7y , https://lnkd.in/p/ghzxy5r8
# View Tutorial on Youtube: https://youtu.be/AlVW02vPPJw , https://youtu.be/ECNBEARj1Xo , https://youtu.be/tfBv7sao9PA

**Three real-world troubleshooting walkthroughs based on the Section 4 labs and hands-on exercises of the *AWS Certified Generative AI Developer – Professional***

### 1: Bedrock Agent Can't Call Its Action Group Lambda

**Related lab:** Hands On: Amazon Bedrock Agents Classic (Section 4)
**Concepts:** LLM agents, action groups, AWS Lambda, resource-based policies, agent versions and aliases, trace

### Scenario

An agent is configured with an action group backed by an AWS Lambda function (for example, a `getWeather` or `getOrderStatus` tool). The Lambda works when tested on its own in the Lambda console, but when the agent is tested in the Bedrock console it fails to use the tool.

### Symptoms

The test window returns an error similar to:

```
Access denied while invoking Lambda function arn:aws:lambda:us-east-1:111122223333:function:agent-orders-fn.
Check the permissions on Lambda function and retry the request.
```

After that is fixed, a second error can appear:

```
The server encountered an error processing the Lambda response.
Check the Lambda response and retry the request.
```

And sometimes: changes to the instructions or action group don't seem to have any effect.

### Diagnosis

1. **Open the trace** in the agent test window. The *Orchestration* step shows the model's rationale and the `invocationInput` for the action group. If the model *chose* the tool correctly but the call failed, the problem is permissions or the Lambda response — not the prompt.
2. **Check the Lambda's resource-based policy** (Lambda console → Configuration → Permissions → Resource-based policy statements), or:

   ```
   aws lambda get-policy --function-name agent-orders-fn
   ```
3. **Check CloudWatch Logs** for the Lambda (`/aws/lambda/agent-orders-fn`). If the function ran but the agent still errors, the returned JSON is in the wrong shape.

### Fix

#### 1. Allow Bedrock to invoke the Lambda

```bash
aws lambda add-permission \
  --function-name agent-orders-fn \
  --statement-id AllowBedrockAgentInvoke \
  --action lambda:InvokeFunction \
  --principal bedrock.amazonaws.com \
  --source-account 111122223333 \
  --source-arn arn:aws:bedrock:us-east-1:111122223333:agent/AGENT_ID
```
Scoping with `--source-arn` means only *this* agent can call the function (least privilege).

#### 2. Return the response envelope Bedrock expects
For an action group defined with **function details**:

```python
def lambda_handler(event, context):
    action_group = event["actionGroup"]
    function = event["function"]
    params = {p["name"]: p["value"] for p in event.get("parameters", [])}

    order_id = params.get("orderId")
    body = f"Order {order_id} has shipped and arrives in 2 days."

    return {
        "messageVersion": "1.0",
        "response": {
            "actionGroup": action_group,
            "function": function,
            "functionResponse": {
                "responseBody": {"TEXT": {"body": body}}
            },
        },
        "sessionAttributes": event.get("sessionAttributes", {}),
        "promptSessionAttributes": event.get("promptSessionAttributes", {}),
    }
```

#### 3. Prepare the agent after every change

Click **Prepare** in the console (or call `aws bedrock-agent prepare-agent --agent-id AGENT_ID`). The built-in test alias (`TSTALIASID`) points to the DRAFT version. For a real application, create a new version and point your alias at it.

### Verification
Ask the agent: *"What is the status of order 1234?"*
Trace shows: rationale → action group invocation → observation containing the Lambda's text → final answer.
Lambda CloudWatch log shows one invocation with the `orderId` parameter.



### Example 2: Strands Agent Works Locally but Fails on AgentCore Runtime

**Related lab:** Lab: Strands Agents, Amazon Bedrock AgentCore, Agent Squad (Section 4)
**Concepts:** Strands Agents, AgentCore Runtime, cross-Region inference profiles, IAM execution roles, observability


### Scenario

A simple Strands agent runs fine on a laptop with `python agent.py`. After deploying it to Amazon Bedrock AgentCore Runtime with the starter toolkit (`agentcore configure` → `agentcore launch`), every invocation fails.

```python
# agent.py
import os
from strands import Agent
from strands.models import BedrockModel
from bedrock_agentcore.runtime import BedrockAgentCoreApp

app = BedrockAgentCoreApp()

model = BedrockModel(
    model_id=os.environ.get("MODEL_ID", "anthropic.claude-sonnet-4-20250514-v1:0"),
    region_name=os.environ.get("AWS_REGION", "us-east-1"),
)
agent = Agent(model=model, system_prompt="You are a helpful study assistant.")

@app.entrypoint
def invoke(payload):
    result = agent(payload.get("prompt", "Hello"))
    return {"result": str(result)}

if __name__ == "__main__":
    app.run()
```


### Fix

#### A. Use the inference profile ID

```bash
# US Regions
export MODEL_ID="us.anthropic.claude-sonnet-4-20250514-v1:0"
# EU users (see the course note for EU users) use the eu. prefix instead
# export MODEL_ID="eu.anthropic.claude-sonnet-4-20250514-v1:0"
```
Pass it as an environment variable when launching, or change the default in `agent.py`. Check which profiles exist in your Region:

```bash
aws bedrock list-inference-profiles --region us-east-1 \
  --query "inferenceProfileSummaries[].inferenceProfileId"
```

#### B. Grant the execution role both resources

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "InvokeViaInferenceProfile",
      "Effect": "Allow",
      "Action": ["bedrock:InvokeModel", "bedrock:InvokeModelWithResponseStream"],
      "Resource": [
        "arn:aws:bedrock:*:111122223333:inference-profile/us.anthropic.claude-*",
        "arn:aws:bedrock:*::foundation-model/anthropic.claude-*"
      ]
    }
  ]
}
```

Attach it to the Runtime execution role, then redeploy with `agentcore launch`.

Also confirm **model access** is enabled for the model in the Bedrock console (Anthropic models may require a one-time use-case form).





### Example 3: Amazon Q Business Says "No Answer Found" for Documents That Exist

**Related labs:** Amazon Q Business – Hands On, Amazon Q Business – Cleanup (Section 4); the same ideas apply to grounding in Amazon Quick
**Concepts:** Retrieval grounding, data source sync, ACL-aware retrieval, IAM Identity Center users, response settings, cost cleanup


### Scenario

An Amazon Q Business application is created with an Amazon S3 data source containing company policy PDFs (for example, `travel-policy.pdf`). An admin can see the files in S3, but when a test user asks *"What is the per-diem limit for international travel?"* in the web experience, Q replies that it could not find an answer.

### Symptoms

```
Sorry, I could not find relevant information to complete your request.
```

### Fix

1. **Grant the data source role read access** to the bucket:

   ```json
   {
     "Effect": "Allow",
     "Action": ["s3:GetObject", "s3:ListBucket"],
     "Resource": ["arn:aws:s3:::company-policies", "arn:aws:s3:::company-policies/*"]
   }
   ```

2. **Correct the ACL identities** so they match IAM Identity Center (same email/username), or add the user to the group listed in the ACL.
3. **Re-sync the data source** and wait for `SUCCEEDED`:

   ```bash
   aws qbusiness start-data-source-sync-job \
     --application-id APP_ID --index-id INDEX_ID --data-source-id DS_ID
   ```

4. **Confirm the user's subscription** in the application's user access settings.
5. Decide deliberately whether to allow direct LLM answers. For a policy assistant, *company knowledge only* is usually correct — just know that it makes empty retrieval visible as "no answer".

### Verification

Sync history shows `SUCCEEDED` with the expected number of documents added.
The test user asks the same question and gets an answer **with a citation** to `travel-policy.pdf`.
A user who is *not* in the ACL still gets no answer from that document — proving ACLs work as intended.

### Cleanup (avoid surprise charges)



## Common debugging checklist for agentic AI on AWS

1. **Check the trace / logs first** — agent trace, AgentCore Observability, CloudWatch Logs.
2. **Separate reasoning problems from execution problems** — did the model pick the wrong tool, or did the right tool fail?
3. **Check who is calling whom** — execution roles vs. resource-based policies vs. end-user ACLs.
4. **Check Region and model ID** — inference profiles (`us.` / `eu.` / `global.`) and model access.
5. **Redeploy correctly** — Prepare the agent, relaunch the runtime, re-sync the data source.
6. **Clean up** — OpenSearch Serverless collections, Q Business apps, AgentCore runtimes and ECR images.

