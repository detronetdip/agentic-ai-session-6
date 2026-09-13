# Safety, grading, and publish

Copy a step, paste it into Cursor, run what it builds, check it worked, then move on. Don't skip ahead.

Today: turn the safety filter on, collect sample questions, check LangSmith, grade the assistant, then publish it.

---

## Step 1 — Safety filter demo: off vs on

Prerequisite — create the standard `monk-research-guardrail` first (AWS console, or this CLI command; region `us-east-1`, needs `bedrock:CreateGuardrail`):

```bash
cat > ./topic-policy.json <<'EOF'
{"topicsConfig":[{"name":"Cooking and Recipes","definition":"Any request for cooking recipes, ingredients, or step-by-step food preparation instructions.","examples":["give me a recipe for","how do I cook","ingredients for"],"type":"DENY"}]}
EOF

cat > ./content-policy.json <<'EOF'
{"filtersConfig":[
  {"type":"HATE","inputStrength":"HIGH","outputStrength":"HIGH"},
  {"type":"INSULTS","inputStrength":"HIGH","outputStrength":"HIGH"},
  {"type":"SEXUAL","inputStrength":"HIGH","outputStrength":"HIGH"},
  {"type":"VIOLENCE","inputStrength":"HIGH","outputStrength":"HIGH"},
  {"type":"MISCONDUCT","inputStrength":"HIGH","outputStrength":"HIGH"},
  {"type":"PROMPT_ATTACK","inputStrength":"HIGH","outputStrength":"NONE"}
]}
EOF

cat > ./pii-policy.json <<'EOF'
{"piiEntitiesConfig":[{"type":"PHONE","action":"ANONYMIZE"},{"type":"EMAIL","action":"ANONYMIZE"}]}
EOF

aws bedrock create-guardrail \
  --region us-east-1 \
  --name "monk-research-guardrail" \
  --description "Monk bootcamp standard guardrail for Project 1" \
  --blocked-input-messaging "Sorry — the Monk Research Assistant only handles business and technology research, not cooking questions." \
  --blocked-outputs-messaging "Sorry — the Monk Research Assistant only handles business and technology research, not cooking questions." \
  --topic-policy-config file://topic-policy.json \
  --content-policy-config file://content-policy.json \
  --sensitive-information-policy-config file://pii-policy.json
```

Copy the printed `guardrailId`, then: `export BEDROCK_GUARDRAIL_ID=<id>` and `export BEDROCK_GUARDRAIL_VERSION=DRAFT`.

> Make a short before/after safety demo.
>
> The test question is exactly: `Give me a step-by-step recipe to make a cake.`
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

