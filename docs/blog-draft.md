# SaansSaathi: turning Delhi's air forecast into a school timetable, with one tap from the principal

*Draft for AWS Builder Center. Built for the WeMakeDevs x AWS "Environmental Hacks" hackathon (Air track).*

## The problem

On a bad winter morning in Delhi, the air is at its worst between about 6 and 10 AM. That is exactly when children stand in assembly and run in PE. By early afternoon the air is often much cleaner. On 18 November 2024, a GRAP Stage IV day, the forecast we replayed showed AQI (est.) around 318 at 7 AM and about 92 by 1 PM.

The forecast already exists. What doesn't exist is the step from "AQI 318" to "move Class 6–8 PE from 8:00 to 12:20, hold assembly indoors, keep windows shut until 10". A principal has to make that call before 7 AM, alone, on a phone. And the decision has to reach parents, many of whom would rather hear a short Hindi voice note than read an English circular.

## What SaansSaathi does

Every school morning at 5:30 IST:

1. Pulls the hourly PM2.5 and PM10 forecast for the school's location (Open-Meteo, CAMS model).
2. Converts each hour with the CPCB National AQI breakpoints. We call it "AQI (est.)" because the official index uses 24-hour averages.
3. A rule engine decides each timetable block: keep, indoors, or reschedule to the cleanest free slot (only if that slot is 200 or below). Every decision carries its numbers: `PE (Classes 6-8) 8:00 → 12:20: AQI (est.) 317 → 104`.
4. A Strands agent on Amazon Bedrock writes three messages from that plan: a principal summary, a Hindi parent note under 60 words, and the same in English.
5. The principal gets the plan in Telegram with **Approve** and **Skip**.
6. On Approve, Amazon Polly (Kajal, neural, hi-IN) reads the parent note, and every parent gets the text and the voice note.
7. A counter adds up child-hours of outdoor activity moved out of AQI 201+ air. One demo school, one morning: 940.

## The stack

- **Strands Agents SDK** for the agent, with typed tools (`get_school`, `compute_plan`, `get_forecast`) and a Pydantic output model.
- **Amazon Bedrock** (Claude Sonnet) for the wording.
- **Amazon Polly** for the Hindi voice.
- **AWS Lambda**: a fast API function for the Telegram webhook and the dashboard, plus a worker function for the slow parts, invoked asynchronously so Telegram always gets a quick 200.
- **API Gateway HTTP API**, **EventBridge Scheduler** (cron in Asia/Kolkata, Monday to Saturday), **DynamoDB** single table, **S3**, **CloudFront** for the dashboard, **SSM Parameter Store** for the bot token.
- **AWS SAM** for the whole stack, with a Makefile build so the function zip holds only the app and a layer holds the dependencies.

## Design decision: rules decide, the model words

The first version of the idea had the LLM "decide what to do about the air". We dropped that early. A principal has to trust the plan, and a judge has to be able to check it. So:

- All thresholds live in `config/rules.yaml`.
- The planner is deterministic and fully unit-tested, including every NAQI breakpoint edge.
- The agent's tools return the already-computed plan, so it physically can't approve something different from what the rules produced.
- Every agent message is checked: the Hindi one must be mostly Devanagari and under 60 words. Anything that fails falls back, field by field, to a template built from the rule output. If Bedrock is down entirely, all three messages come from templates and the flow still works.

That last point turned out to matter a lot (see below).

## What fought back

**1. The forecast underestimates Delhi's worst days.** CAMS global runs on a grid of roughly 45 km. On 18 Nov 2024 the CPCB stations reported a 24-hour AQI near 490; CAMS peaked around 318 (AQI est.). Two of our three demo schools, in RK Puram and Dwarka, sit in the same grid cell and get identical numbers. We kept the real numbers and wrote the limitation down rather than inflating them. Station bias correction is the obvious next step.

**2. A packing bug in rescheduling.** Our first planner only tried start times on a 30-minute grid. When two PE periods both needed the clean afternoon, the second one never found the slot right after the first (12:40) and went indoors instead. A failing test caught it; the fix was to also try "right after each busy block" as a candidate start.

**3. A brand-new AWS account is not a ready AWS account.** This was the big one. On the new Free account plan, a service control policy explicitly denied Lambda, CloudFormation and Polly. Upgrading to the Paid plan (with a $20 hard spend limit, which is a lovely feature) didn't lift it straight away, and Bedrock said the account was still "being verified". Our newest target model wasn't available to the account yet either, so we moved to Claude Sonnet 4.6, which was. And Claude needed the one-time Anthropic use-case form before the agent's streaming calls would go through.
   Because the agent falls back to templates and the bot runs locally with long polling, we could still test the full principal → parent flow on a real phone while waiting on AWS Support.

**4. Small local-dev traps.** `aws login` credentials need `botocore[crt]` in the Python environment. And our SVG chart, drawn at a fixed 640-unit width, shrank its labels to unreadable on a phone; measuring the container with a `ResizeObserver` fixed it.

## What's next

- Correct the CAMS forecast with nearby CPCB station readings.
- A WhatsApp channel behind the same `Channel` interface.
- Let a school paste its real timetable instead of the demo one.

## Try it

Code, setup and demo steps: https://github.com/Yashika2211/santhisathi

*Built with help from Claude Code. Forecast data: Open-Meteo / CAMS, CC BY 4.0. AQI breakpoints: CPCB.*
