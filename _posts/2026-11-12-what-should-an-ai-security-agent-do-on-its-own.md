---
layout: post
title: "Treat your AI security agent like a new hire"
date: 2026-11-12 13:30:00 +0530
categories: [infrastructure]
---

Picture an analyst who joined your team last week, on call for the first time. At 2:07 a.m., an admin account signs in from a country its owner has never used, then downloads a lot of data. The analyst checks the sign-in history and the ticket history, then says: "This looks compromised. I'd like to disable the account."

Would you let them?

Probably not without a phone call. The owner may be travelling. The download may be a scheduled export. The account may be the one the deployment pipeline uses, so disabling it blocks the release and the engineers trying to respond. The analyst did good work. They just don't know your environment yet.

An AI agent that investigates alerts is in the same position. It is fast and it never gets tired, and it is new to your systems. The useful question is not "is the agent smart enough?" but "what would I hand to a new hire, and when?"

> **About this post.** 
>
> My perspective is shaped by work in alert-driven automation and policy engines.

## The same strengths, the same gaps

A good new analyst reads quickly and writes a clear summary. They also don't know which alerts are routine here, which accounts are special, or that maintenance is planned for Thursday. An agent has the same gaps, plus one more: it can be fooled by text.

Logs, emails and ticket comments are written, at least in part, by people outside your control. A username or a browser string is whatever the sender chose to send. If an agent treats text like that as an instruction, an attacker can try to steer it. This is often called prompt injection, and [OWASP lists it first](https://owasp.org/www-project-top-10-for-large-language-model-applications/) in its Top 10 for LLM applications.

So one rule comes first: everything the agent reads is data, never instructions.

## Authority comes in stages

Nobody gives a new hire every key in their first week. They watch, then suggest, then act with a senior person signing off, and later they act alone on routine work. An agent can follow the same path:

1. **Shadow.** It runs on real alerts and only logs what it would have done.
2. **Recommend.** It proposes an action with evidence, and a person decides.
3. **Act, signed off.** It performs the action after a person approves it.
4. **Act and report.** It acts, then tells you.

Moving up a stage needs evidence. Compare the agent's proposals with what your people decided, and look at its mistakes. A confident tone is not evidence. An agent can sound certain and be wrong.

## The ceiling depends on the consequence

Not every action needs the same path. I would ask three things about each one:

- **Reach.** If it is wrong, who is affected: one user, a team, the whole company?
- **Reversibility.** Can it be undone, and has anyone tried the undo?
- **Speed of harm.** How quickly does a wrong action hurt, and would anyone notice in time?

Here is my own starting point. It is not a standard, so adjust it to your environment.

![A map of six actions and the highest stage each should reach, from summarizing evidence up to act and report, to deleting evidence, which stays recommend only.]({{ "/assets/images/ai-agent-authority-map.png" | relative_url }})

Summaries and tags are low-impact, so they can reach "act and report" fairly early. Ending a single session is small and reversible, so it can start with sign-off and move up later, with a record to back it. Disabling an account, isolating a host or revoking keys can lock people out or stop a service, so I would keep a person in the loop every time. Deleting data or evidence cannot be undone, so the agent only recommends.

## What you would give a new hire

Most safeguards for agents are things we already do for people:

| For a new analyst | For an agent |
|---|---|
| Their own login, read-only at first | Its own identity, with only the tools the current job needs |
| A second pair of eyes | An approval step that shows the evidence, what could not be checked, what will change and how to undo it |
| Shadowing a senior colleague | Dry-run mode on real alerts |
| "Ask before doing this twenty times" | Rate limits and cooldowns |
| Working from a ticket | A unique key per action, so a retry does not repeat it |
| Case notes | An audit trail of what it saw and why it acted |
| Authority you can withdraw | A stop switch and an undo you have tested |

The approval row is where it is easiest to go wrong. If the approver sees only "Approve?", approval becomes a rubber stamp.

In my own rule-based automation, writing the rule was the easy part. Deciding what it was allowed to do, and recording what it did, took most of the work. I expect that to hold for agents, but I have not tested it.

One more habit from that work: silence must not look like health. If a log query times out, "no evidence found" must never be reported as "nothing is wrong". The agent should say that the check failed.

## Could you build this with open-source tools?

Yes, the pieces exist. This is how I would sketch it, not something I have built.

- **Alerts in.** An open-source detection platform such as Wazuh raises the alert and sends it to your workflow.
- **The agent.** An open-source agent framework such as LangGraph can run the investigation. Its `interrupt()` call pauses the run before a tool executes, saves the state, and waits until a person resumes it with an approve or reject answer. That is the "act, signed off" stage.
- **The ceiling.** Don't leave the limits in the prompt. Put a plain table of actions and their highest stage in code, as in the diagram, so the agent cannot skip approval by being persuasive.
- **Tools.** Wrap each action in a small function with its own narrow credentials, and give disruptive ones a unique action key so a retry does not run twice.
- **Record.** Write every step to an append-only log or table, with the evidence shown and who approved.

Start with the agent in shadow mode and read its logs for a few weeks before it can act on anything.

## Back to 2:07 a.m.

On its own, I would let the agent gather the sign-in history, tag the alert, write the summary and page the on-call engineer, including a list of what it could not check. With approval, it could disable the account, and the approver would see that this account is used by the deployment pipeline. The agent shortens the time to a decision. The decision stays with a person who can be woken up and asked why.

## Questions before you grant access

- What is the worst thing this tool can do, and who would it affect?
- Would I be comfortable if someone who joined last week did this?
- Which inputs could an attacker influence?
- Who approves high-impact actions, and what do they see?
- If a check fails, does the agent say so?
- Can we stop it quickly and reconstruct what it saw and why it acted?
- What evidence would make us give it more authority, or less?

## The takeaway

Authority should follow evidence and consequence, not enthusiasm. Let the agent read and propose. Let it earn the right to act, one low-impact action at a time.

*How do you handle automation and approval in your own environment? Where has an approval step helped, and where has it got in the way? I would like to hear.*

---

*Written by Pavan Aravapalli, based on experience building alert-driven automation. A design perspective, not a report on a system I built.*
