---
layout: post
title: "Turning Alerts Into Action: A Policy Engine Then and Now."
date: 2026-11-02 13:00:00 +0530
categories: [infrastructure]
---

A decade ago, I built a policy engine on top of an open-source monitoring tool. The idea was simple: when a metric crosses a line, do something about it, automatically.

It ran in production for a long time, and that is the part I am proud of. The rest of this post is about what I would change if I started today, and what I would not build at all, because the tools and the expectations are very different now.

> **About this post.** I am writing this from my work experience, using generic technical terms. I want to share what was done a decade ago and how it could be done differently now, in 2026. When I say "today", I mean what I would choose now, not something I built.

## What it did

The monitoring tool collected metrics from servers and services on a cloud management platform. It could raise alerts, but an alert is only a message. Somebody still had to read it and act.

The policy engine closed that gap. Here is how it worked, in six steps.

![Policy engine flow: policy templates, agent, monitoring server, wrapper layer, scheduled jobs, notification and recommendation framework]({{ "/assets/images/policy-engine-2015-flow.png" | relative_url }})

1. A monitoring agent was installed when a VM was launched on the platform. Either it was shipped inside the image, already pointing at the monitoring server, or it was set up live after launch.
2. Policies decided what to watch. They came from predefined templates, or from new templates created for the resource.
3. The agent collected metadata and metrics and sent them to the central monitoring server, which stored them.
4. A wrapper layer, so the rest of the platform could read that data without talking to the monitoring tool directly.
5. Scheduled jobs ran at fixed intervals. They fetched the data through the wrappers and checked it against the policy. On a violation or threshold breach, they called the recommendation framework and sent a mail or push notification.
6. The recommendation framework acted only if the user had opted in while creating the policy. For utilisation-based policies it scaled the resource up or down. For security-related ones it started the matching job.

Cloud providers also had alarms that could trigger actions, such as auto scaling or stopping and recovering an instance. They worked only inside that provider's own cloud. Ours had to work on a private platform, where nothing like that existed.

## What worked

Keeping the rules separate from the monitoring tool. The tool measured, and the engine decided. That split is the reason it survived for years.

## Where it hurt

The weak spots all came from the scheduled jobs.

- **A slow run.** If a job took longer than its interval, the next run started late or overlapped with it. Checks fell behind, and an alert could arrive well after the problem began.
- **Broken data retrieval.** If the job could not read the data, there was nothing to check and nothing to say it had failed. A quiet gap looks the same as a healthy system.
- **Manual recovery.** When runs were missed, someone had to restart the jobs by hand and work out what had been skipped. Nothing caught up on its own.

None of these needed a clever fix. They needed the engine to know its own state.

## What I would do differently today

![Detect, decide, act, with an audit trail]({{ "/assets/images/policy-engine-today.png" | relative_url }})

The three stages stay separate, and every stage writes to the audit trail. The new part is the engine health check. It tracks the last good run, raises an alert when data goes missing or the engine stops, and lets detection resume from the last good run.

**1. Split it into three parts: detect, decide, act.**
In my version, one scheduled job fetched the data, checked it and triggered the next step in a single pass. Today I would keep the three apart, with a clear event passing between each stage, and react to events instead of polling where the source allows it. Detection says "this happened". Decision says "this is what we do about it". Action does it. When something goes wrong, you know which of the three failed.

**2. Write policies as data, and keep them in version control.**
A policy should be a small declarative file that someone can review in a pull request, not a setting buried in a screen.

```yaml
policy: high-load-restart
when:
  metric: cpu_utilisation
  above: 90
  for: 10m
then:
  action: restart_service
  cooldown: 30m
  max_per_day: 3
  notify: ops-team
```

You get a history of who changed what and why, you can roll a bad policy back, and you can test it before it goes live.

**3. Treat the action as the dangerous part.**
Writing the rule is easy. Making the action safe is the real work. A rule that restarts a service when load is high can restart it again and again while the real problem is somewhere else, and you have made the outage worse.

So every action needs limits: a cooldown between runs, a maximum number of runs per day, and a way to stop and hand over to a human. The example above has these built in on purpose.

**4. Make every action safe to run twice.**
Messages get delivered twice, and engines retry. An action like "restart" or "scale to N" should give the same result if it runs again. Avoid actions like "add one more server", because a retry quietly adds two.

**5. Add a dry-run mode from day one.**
Run the policy, log what it would have done, and do nothing. Let it run for a few days against real traffic before it is allowed to act. It is the cheapest way to find a bad rule.

**6. Keep an audit trail.**
Every action should record which policy fired, which metric value triggered it, what ran, and what the result was. When someone asks at two in the morning why a service restarted, the answer should take one search.

**7. Make the engine watch itself.**
Record when each policy was last checked successfully, and start every run from that point, so a missed window is caught up automatically instead of by hand. Stop two runs of the same job from overlapping, for example with a lock that expires if its holder dies. Treat "no data" as its own alert, because silence is not the same as healthy. And alert when the engine itself stops running.

Catching up has its own risk. If the engine was down for two hours, it should not replay old readings and act on a problem that has already gone away, such as restarting a service that recovered an hour ago. Catch-up should still respect the cooldown and the daily cap from point 3, and for old windows it should usually notify and not act.

**8. Do not tie the engine to one monitoring tool.**
My wrapper layer already hid the monitoring server from the rest of the platform, and I would keep that. But the policies still assumed what that one tool could measure. Today I would define the metric names and units myself and map each source onto them, so the source can change without touching a policy.

## What I would not build in 2026

Much of what I wrote by hand is now a standard feature. Here is how each piece maps to today.

| What I built | What I would use today |
|---|---|
| Agent shipped inside the VM image | A standard collector or exporter, such as OpenTelemetry, that most platforms already support |
| Wrapper layer over the monitoring tool | A standard metrics query API, so the rest of the platform reads data without a custom adapter |
| Scheduled polling jobs | Alert rules evaluated by the monitoring system itself, which sends an event when one fires |
| Mail and push notification code | An alert-routing tool with grouping, silencing and escalation |
| Scale up and down logic | The built-in autoscalers, such as Kubernetes autoscaling, event-driven scalers like KEDA, or a provider's autoscaling |
| Policy screens and templates | Policy files in Git, reviewed like any other change |

## What is still worth building

- **The actions that are specific to your platform.** Nobody else knows how to safely drain, resize or isolate your own services.
- **Rules that span several systems.** A single tool sees only its own metrics.
- **The brakes.** Cooldowns, daily caps, dry-run, approval for risky actions, and the audit trail. Off-the-shelf tools give you the alert. They rarely give you these.

There is one new temptation in 2026: letting an AI model choose the action. I would use a model to summarise an alert or suggest a next step, and keep the execution behind the same limits, the same approvals and the same audit trail as any other rule. If an engine that acts on its own is only as good as its brakes, that applies even more to one whose decisions you cannot fully predict.

## Should you build one at all?

For most teams, not at first. On a public cloud, start with the alarms and autoscaling your provider already gives you. On a private cloud or a product you sell, start with open-source alerting and autoscaling tools. In both cases, build only what is specific to your platform: its own actions, its safety limits and its audit trail. The rule language is the easy part.

## The takeaway

An engine that acts on its own is only as good as its brakes. Detecting a problem is well supported by tools now. Deciding when not to act, and proving afterwards what you did, is where the engineering is.

*If you have built or inherited something like this, I would like to hear what went wrong first.*

---

*Written by Pavan Aravapalli, based on hands-on development experience with monitoring and automation. Generic overview; no employer, product or customer details.*
