---
title: Escalate a Job in Job Router
titleSuffix: An Azure Communication Services how-to guide
description: Use Azure Communication Services SDKs to escalate a Job
author: jasonshave
ms.author: jassha
ms.service: azure-communication-services
ms.topic: how-to 
ms.date: 10/14/2021
ms.custom: template-how-to, devx-track-extended-java, devx-track-js
zone_pivot_groups: acs-js-csharp-java-python

#Customer intent: As a developer, I want to manage the behavior of my jobs in a queue.
---

# Escalate a job

[!INCLUDE [Public Preview Disclaimer](../../includes/public-preview-include-document.md)]

This guide shows you how to escalate a Job in a Queue by using an Exception Policy.

## Prerequisites

- An Azure account with an active subscription. [Create an account for free](https://azure.microsoft.com/free/?WT.mc_id=A261C142F).
- A deployed Communication Services resource. [Create a Communication Services resource](../../quickstarts/create-communication-resource.md).
- Optional: Complete the quickstart to [get started with Job Router](../../quickstarts/router/get-started-router.md)
- Optional: Review the [Job classification how-to guide](job-classification.md)

## Escalation Overview

Escalation can take the form of several different behaviors including moving a Job to a different Queue and/or specifying a higher priority. Jobs with a higher priority are distributed to Workers before jobs with a lower priority. For this how-to guide, we use a Classification Policy and an Exception Policy and to achieve this goal.

## Classification policy configuration

Create a Classification Policy to handle the new label added to the Job. This policy evaluates the `Escalated` label and assigns the Job to either Queue. The policy also uses the [Rules Engine](../../concepts/router/router-rule-concepts.md) to increase the priority of the Job from `1` to `10`.

::: zone pivot="programming-language-csharp"

```csharp
var classificationPolicy = await administrationClient.CreateClassificationPolicyAsync(
    new CreateClassificationPolicyOptions(classificationPolicyId: "Classify_XBOX_Voice_Jobs")
    {
        Name = "Classify XBOX Voice Jobs",
        QueueSelectors =
        {
            new ConditionalQueueSelectorAttachment(
                condition: new ExpressionRouterRule("job.Escalated = true"),
                queueSelectors: new List<RouterQueueSelector>
            {
                new (key: "Id", labelOperator: LabelOperator.Equal, value: new LabelValue("XBOX_Escalation_Queue"))
            })
        },
        PrioritizationRule = new ExpressionRouterRule("If(job.Escalated = true, 10, 1)"),
    });
```

::: zone-end

::: zone pivot="programming-language-javascript"

```typescript
var classificationPolicy = await administrationClient.createClassificationPolicy("Classify_XBOX_Voice_Jobs", {
    name: "Classify XBOX Voice Jobs",
    queueSelectors: [{            
        kind: "conditional",
        condition: {
            kind: "expression-rule",
            expression: 'job.Escalated = true'
        },
        queueSelectors: [{
            key: "Id",
            labelOperator: "equal",
            value: "XBOX_Escalation_Queue"
        }]
    }],
    prioritizationRule: {
        kind: "expression-rule",
        expression: "If(job.Escalated = true, 10, 1)"
    }});
```

::: zone-end

::: zone pivot="programming-language-python"

```python
classification_policy: ClassificationPolicy = administration_client.create_classification_policy(
    classification_policy_id = "Classify_XBOX_Voice_Jobs",
    classification_policy = ClassificationPolicy(
        name = "Classify XBOX Voice Jobs",
        queue_selectors = [
            ConditionalQueueSelectorAttachment(
                condition = ExpressionRouterRule(expression = 'job.Escalated = true'),
                queue_selectors = [
                    RouterQueueSelector(key = "Id", label_operator = LabelOperator.EQUAL, value = "XBOX_Escalation_Queue")
                ]
            )
        ],
        prioritization_rule = ExpressionRouterRule(expression = "If(job.Escalated = true, 10, 1)")))
```

::: zone-end

::: zone pivot="programming-language-java"

```java
ClassificationPolicy classificationPolicy = administrationClient.createClassificationPolicy(
    new CreateClassificationPolicyOptions("Classify_XBOX_Voice_Jobs")
        .setName("Classify XBOX Voice Jobs")
        .setQueueSelectors(List.of(new ConditionalQueueSelectorAttachment(
            new ExpressionRouterRule("job.Escalated = true"),
            List.of(new RouterQueueSelector("Id", LabelOperator.EQUAL, new LabelValue("XBOX_Escalation_Queue"))))))
        .setPrioritizationRule(new ExpressionRouterRule("If(job.Escalated = true, 10, 1)")));
```

::: zone-end

## Exception policy configuration

Create an exception policy attached to the queue, which is time triggered and takes the action of the Job being reclassified.

::: zone pivot="programming-language-csharp"

```csharp
var exceptionPolicy = await administrationClient.CreateExceptionPolicyAsync(new CreateExceptionPolicyOptions(
    exceptionPolicyId: "Escalate_XBOX_Policy",
    exceptionRules: new Dictionary<string, ExceptionRule>
    {
        ["Escalated_Rule"] = new(
            trigger: new WaitTimeExceptionTrigger(TimeSpan.FromMinutes(5)),
            actions: new Dictionary<string, ExceptionAction?>
            {
                ["EscalateReclassifyExceptionAction"] =
                    new ReclassifyExceptionAction(classificationPolicyId: classificationPolicy.Value.Id)
                    {
                        LabelsToUpsert = { ["Escalated"] = new LabelValue(true) }
                    }
            }
        )
    }) { Name = "Add escalated label and reclassify XBOX Job requests after 5 minutes" });
```

::: zone-end

::: zone pivot="programming-language-javascript"

```typescript
await administrationClient.createExceptionPolicy("Escalate_XBOX_Policy", {
    name: "Add escalated label and reclassify XBOX Job requests after 5 minutes",
    exceptionRules: {
        Escalated_Rule: {
            trigger: { kind: "wait-time", thresholdSeconds: 5 * 60 },
            actions: { EscalateReclassifyExceptionAction: {
                kind: "reclassify", classificationPolicyId: classificationPolicy.id, labelsToUpsert: { Escalated: true }
            }}
        }
    }
});
```

::: zone-end

::: zone pivot="programming-language-python"

```python
administration_client.create_exception_policy(
    exception_policy_id = "Escalate_XBOX_Policy",
    exception_policy = ExceptionPolicy(
        name = "Add escalated label and reclassify XBOX Job requests after 5 minutes",
        exception_rules = {
            "Escalated_Rule": ExceptionRule(
                trigger = WaitTimeExceptionTrigger(threshold_seconds = 5 * 60),
                actions = { "EscalateReclassifyExceptionAction": ReclassifyExceptionAction(
                    classification_policy_id = classification_policy.id,
                    labels_to_upsert = { "Escalated": True }
                )}
            )
        }
    )
)
```

::: zone-end

::: zone pivot="programming-language-java"

```java
administrationClient.createExceptionPolicy(new CreateExceptionPolicyOptions("Escalate_XBOX_Policy",
    Map.of("Escalated_Rule", new ExceptionRule(new WaitTimeExceptionTrigger(5 * 60),
        Map.of("EscalateReclassifyExceptionAction", new ReclassifyExceptionAction()
            .setClassificationPolicyId(classificationPolicy.getId())
            .setLabelsToUpsert(Map.of("Escalated", new LabelValue(true))))))
).setName("Add escalated label and reclassify XBOX Job requests after 5 minutes"));
```

::: zone-end

## Queue configuration

Create the necessary Queues for regular and escalated Jobs and assigns the Exception Policy to the regular Queue.

> [!NOTE]
> This step assumes you have created a distribution policy already with the name of `Round_Robin_Policy`.

::: zone pivot="programming-language-csharp"

```csharp
var defaultQueue = await administrationClient.CreateQueueAsync(
    new CreateQueueOptions(queueId: "XBOX_Queue", distributionPolicyId: "Round_Robin_Policy")
    {
        Name = "XBOX Queue",
        ExceptionPolicyId = exceptionPolicy.Value.Id
    });

var escalationQueue = await administrationClient.CreateQueueAsync(
    new CreateQueueOptions(queueId: "XBOX_Escalation_Queue", distributionPolicyId: "Round_Robin_Policy")
    {
        Name = "XBOX Escalation Queue"
    });
```

::: zone-end

::: zone pivot="programming-language-javascript"

```typescript
await administrationClient.createQueue("XBOX_Queue", {
    distributionPolicyId: "Round_Robin_Policy",
    exceptionPolicyId: exceptionPolicy.id,
    name: "XBOX Queue"
});

await administrationClient.createQueue("XBOX_Escalation_Queue", {
    distributionPolicyId: "Round_Robin_Policy",
    name: "XBOX Escalation Queue"
});
```

::: zone-end

::: zone pivot="programming-language-python"

```python
administration_client.create_queue(
    queue_id = "XBOX_Queue",
    queue = RouterQueue(
        distribution_policy_id = "Round_Robin_Policy",
        exception_policy_id = exception_policy.id,
        name = "XBOX Queue"))

administration_client.create_queue(
    queue_id = "XBOX_Escalation_Queue",
    queue = RouterQueue(
        distribution_policy_id = "Round_Robin_Policy",
        name = "XBOX Escalation Queue"))
```

::: zone-end

::: zone pivot="programming-language-java"

```java
administrationClient.createQueue(new CreateQueueOptions("XBOX_Queue", "Round_Robin_Policy")
    .setExceptionPolicyId(exceptionPolicy.getId())
    .setName("XBOX Queue"));

administrationClient.createQueue(new CreateQueueOptions("XBOX_Escalation_Queue", "Round_Robin_Policy")
    .setName("XBOX Escalation Queue"));
```

::: zone-end

## Job lifecycle

When you submit the Job, it is added to the queue `XBOX_Queue` with the `voice` channel. For this particular example, the requirement is to find a worker with a label called `XBOX_Hardware`, which has a value greater than or equal to the number `7`.

::: zone pivot="programming-language-csharp"

```csharp
await client.CreateJobAsync(new CreateJobOptions(jobId: "job1", channelId: "voice", queueId: defaultQueue.Value.Id)
{
    RequestedWorkerSelectors =
    {
        new RouterWorkerSelector(key: "XBOX_Hardware", labelOperator: LabelOperator.GreaterThanEqual, value: new LabelValue(7))
    }
});
```

::: zone-end

::: zone pivot="programming-language-javascript"

```typescript
await client.createJob("job1", {
    channelId: "voice",
    queueId: defaultQueue.id,
    requestedWorkerSelectors: [{ key: "XBOX_Hardware", labelOperator: "GreaterThanEqual", value: 7 }]
});
```

::: zone-end

::: zone pivot="programming-language-python"

```python
administration_client.create_job(
    job_id = "job1",
    router_job = RouterJob(
        channel_id = "voice",
        queue_id = default_queue.id,
        requested_worker_selectors = [
            RouterWorkerSelector(key = "XBOX_Hardware", label_operator = LabelOperator.GreaterThanEqual, value = 7)
        ]))
```

::: zone-end

::: zone pivot="programming-language-java"

```java
administrationClient.createJob(new CreateJobOptions("job1", "voice", defaultQueue.getId())
    .setRequestedWorkerSelectors(List.of(
        new RouterWorkerSelector("XBOX_Hardware", LabelOperator.GREATER_THAN_EQUAL, new LabelValue(7)))));
```

::: zone-end

The following lifecycle steps occur once the configuration is complete and the Job is submitted:

1. The Job is sent to Job Router and produces the `RouterJobReceived` and `RouterJobQueued` events.
2. Next, the 5-minute timer begins and triggers if no matching worker is assigned. After 5 minutes, Job Router emits a `RouterJobExceptionTriggered` and another `RouterJobQueued` event.
3. At this point, the Job moves to the `XBOX_Escalation_Queue` and the priority is set to `10`.
