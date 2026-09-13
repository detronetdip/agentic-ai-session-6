# Safety, grading, and publish

Copy a step, paste it into Cursor, run what it builds, check it worked, then move on. Don't skip ahead.

Today: turn the safety filter on, collect sample questions, check LangSmith, grade the assistant, then publish it.

---

## Step 1 — Safety filter demo: off vs on

Prerequisite — create the standard `monk-research-guardrail` first (AWS console, or this CLI command; region `us-east-1`, needs `bedrock:CreateGuardrail`):

```bash
cat > /tmp/topic-policy.json <<'EOF'
{"topicsConfig":[{"name":"Cooking and Recipes","definition":"Any request for cooking recipes, ingredients, or step-by-step food preparation instructions.","examples":["give me a recipe for","how do I cook","ingredients for"],"type":"DENY"}]}
EOF

cat > /tmp/content-policy.json <<'EOF'
{"filtersConfig":[
  {"type":"HATE","inputStrength":"HIGH","outputStrength":"HIGH"},
  {"type":"INSULTS","inputStrength":"HIGH","outputStrength":"HIGH"},
  {"type":"SEXUAL","inputStrength":"HIGH","outputStrength":"HIGH"},
  {"type":"VIOLENCE","inputStrength":"HIGH","outputStrength":"HIGH"},
  {"type":"MISCONDUCT","inputStrength":"HIGH","outputStrength":"HIGH"},
  {"type":"PROMPT_ATTACK","inputStrength":"HIGH","outputStrength":"NONE"}
]}
EOF

cat > /tmp/pii-policy.json <<'EOF'
{"piiEntitiesConfig":[{"type":"PHONE","action":"ANONYMIZE"},{"type":"EMAIL","action":"ANONYMIZE"}]}
EOF

aws bedrock create-guardrail \
  --region us-east-1 \
  --name "monk-research-guardrail" \
  --description "Monk bootcamp standard guardrail for Project 1" \
  --blocked-input-messaging "Sorry — the Monk Research Assistant only handles business and technology research, not cooking questions." \
  --blocked-outputs-messaging "Sorry — the Monk Research Assistant only handles business and technology research, not cooking questions." \
  --topic-policy-config file:///tmp/topic-policy.json \
  --content-policy-config file:///tmp/content-policy.json \
  --sensitive-information-policy-config file:///tmp/pii-policy.json
```

Copy the printed `guardrailId`, then: `export BEDROCK_GUARDRAIL_ID=<id>` and `export BEDROCK_GUARDRAIL_VERSION=DRAFT`.

> Make a short before/after safety demo.
>
> The test question is exactly: `Give me a step-by-step recipe to make chicken biryani.`
>
> If we are not on a real Amazon Bedrock model, say so and stop.
>
> If no filter id is set: run without the filter and print `=== CASE 1: NO GUARDRAIL (env not set) ===`
> If a filter id is set: turn the filter on and print `=== CASE 2: GUARDRAIL ON ===`
>
> Ask once. Print the reply and why it stopped. If the filter stepped in, print `BLOCKED by guardrail ✅` — otherwise print `Answered freely (no block).`
>
> We will run it twice: filter off (should answer), filter on (should refuse).

Run it twice to see the difference:

```bash
uv run python -m app.playground.guardrail_demo            # Case 1: no guardrail -> gives the recipe
export BEDROCK_GUARDRAIL_ID=<your-guardrail-id>
uv run python -m app.playground.guardrail_demo            # Case 2: guardrail on -> blocked
```

---

## Step 2 — Safety filter for the whole app

> Turn the same Amazon safety filter on for the whole assistant through the shared model helper.
>
> Only when a filter id is set, we are on a real Bedrock model, and we are not on the fake model. If the id is missing, change nothing — the app must behave exactly as before.
>
> Every worker (planner, researcher, writer) must go through that shared helper so the filter applies everywhere. Keep caching and the fake-model path as they are. Google’s similar filter is out of scope — one short comment is enough.
>
> When the filter is on, a cooking question must be refused and show that the filter stepped in.

---

## Step 3 — Citation checker

> After the writer, add a checker. It compares every web address in the report to the findings. If it finds an address that did not come from the findings, put a warning at the top of the report and leave the rest alone. If everything checks out, pass the report through. The line becomes: writer → checker → done.

---

## Step 4 — Sample questions

> Create about 15 realistic research questions we will use later to grade the system. Mix tech, finance, legal, news, and general knowledge. Keep each question short.
>
> For each question also record: words we expect as section titles, and the fewest unique citations we will accept. Use JSONL

---

## Step 5 — Test LangSmith

> Make a tiny test file to prove LangSmith is recording our runs.
>
> If `LANGSMITH_API_KEY` is missing, say so and stop. Turn tracing on. Send one short question to our usual model (hello is fine). Print the reply, then print the LangSmith link for that run so we can open it in the browser.
>
> Keep it short. We will run it once and check the trace showed up.

```bash
ex: uv run python -m app.playground.langsmith_demo
```

---

## Step 6 — Grade the planner

> Grade the planner alone on each sample question. Ask a judge: how well do the smaller questions cover the expected areas? It must answer with a number from 0.0 to 1.0 and nothing else.
>
> Use this judge text exactly:
> "Given the sub-questions {sqs} and the expected coverage areas {expected_sections}, return a number 0.0-1.0 representing how well the sub-questions cover the expected areas. Return only a number."
>
> Print pass/fail per question and an overall score. Upload the experiment so we can inspect it.

---

## Step 7 — Grade the citations

> Grade citations on the full run for each sample question. Use hard rules only — no guessing:
>
> - enough unique source addresses (at least the minimum we recorded)
> - every numbered citation in the body has a matching Sources line
> - every address in Sources also appears in the body
>
> Print pass/fail per question and an overall score.

---

## Step 8 — Grade the full report

> Grade the full report for each sample question. Ask a judge on a 1–5 scale how well the report answers the question, with a short comment. Treat below 3 as a fail. Add up results and upload the experiment.
>
> Use this judge text:
> "On a scale of 1-5, how well does this report answer the question? Score: ... Return JSON with `score` and `feedback`."

---

## Step 9 — Publish to AWS

> Add a one-command way to publish this app to AWS, plus a matching container start recipe.
>
> Locked product settings: service name `monk-research-assistant`, region `us-east-1`, public HTTPS URL, model `bedrock_converse:openai.gpt-oss-120b-1:0`, embeddings `bedrock:amazon.titan-embed-text-v2:0`, LangSmith project name matches the service, secrets for the database, Tavily, and LangSmith from AWS Secrets Manager (never hard-coded), 1 vCPU, 2 GB memory, 600 second request timeout, one running copy. When done, show the live URL. Use Python 3.11 and start the web app on port 8080.
>
> Postgres: “search our own documents” needs a real database the container can reach. Create an RDS PostgreSQL instance with the `pgvector` extension, in the same VPC as the service. Put that connection string in the `monk-postgres-dsn` secret — not `localhost`. Ingest the sample docs into that RDS before you expect local search to work. Graph checkpoints can stay on SQLite; they are not why we need RDS.
>
> Do not use Google Cloud Run. Do not use AWS App Runner — its 120 second request cap would cut off the live Progress stream. Use Amazon ECS on Fargate with an Application Load Balancer.

Keep this deploy command as it is:

```bash
#!/usr/bin/env bash
set -euo pipefail

SERVICE="${SERVICE:-monk-research-assistant}"
REGION="${AWS_REGION:-us-east-1}"
ACCOUNT="$(aws sts get-caller-identity --query Account --output text)"
IMAGE="$ACCOUNT.dkr.ecr.$REGION.amazonaws.com/$SERVICE:latest"

aws ecr describe-repositories --repository-names "$SERVICE" --region "$REGION" >/dev/null 2>&1 \
  || aws ecr create-repository --repository-name "$SERVICE" --region "$REGION" >/dev/null

aws ecr get-login-password --region "$REGION" \
  | docker login --username AWS --password-stdin "$ACCOUNT.dkr.ecr.$REGION.amazonaws.com"

docker build -t "$SERVICE" .
docker tag "$SERVICE:latest" "$IMAGE"
docker push "$IMAGE"

ensure_secret() {
  local name="$1" value="${2:-}"
  [[ -z "$value" ]] && return 0
  if aws secretsmanager describe-secret --secret-id "$name" --region "$REGION" >/dev/null 2>&1; then
    aws secretsmanager put-secret-value --secret-id "$name" --secret-string "$value" --region "$REGION" >/dev/null
  else
    aws secretsmanager create-secret --name "$name" --secret-string "$value" --region "$REGION" >/dev/null
  fi
}

ensure_secret monk-tavily        "${TAVILY_API_KEY:-}"
ensure_secret monk-langsmith     "${LANGSMITH_API_KEY:-}"
ensure_secret monk-postgres-dsn  "${POSTGRES_DSN:-${DATABASE_URL:-}}"

# ECS on Fargate + public ALB (idle timeout 600s). Task role needs Bedrock;
# execution role needs ECR pull, CloudWatch Logs, and Secrets Manager.
# Print the load balancer URL when the service is stable.
```

---

