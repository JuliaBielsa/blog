---
title: "SOC Bare Minimum: Getting Out of the Matrix"
date: 2026-09-10
slug: soc-bare-minimum
draft: false
author: "Julia Bielsa"
---

How could we miss an inbound SSH connection from an external IP? The "T1133: External Remote Services" cell in our MITRE matrix was green.

Of course it was. We had a rule for RDP, another two for our VPN, even one for some fancy Docker abuse. But not for SSH.

Crazy, right? And yet this is how most SOCs measure their detection gaps.

ATT&CK is good at showing and organizing what you cover, but it won't tell you how well you cover it, and it definitely won't tell you if you're missing the basics.

A cell turns green when there is content for a technique, regardless of what that content is. The matrix has no axis for maturity, which means a use case for a very sophisticated, rarely-seen technique colors the cell just as green as the most basic one. And the gap isn't only about maturity, it's about scope too. A technique can be thoroughly covered in Azure and completely ignored in AWS. And once again, the cell will be green.

**This post is about how to define the SOC's bare-minimum detections, the ones that MITRE contains but won't highlight as non-negotiable.**

## Why coverage goes wrong

The problem starts when choosing the use cases to deploy. **Coverage in most SOCs is distributed by content availability, not by necessity**. The criteria for picking a use case are usually "a blog had a query to copy," "the log source was already onboarded and we need to justify its cost," or "I just read about this technique and it sounded interesting."

One issue that highlights this lack of criteria is the inconsistency across platforms. It's common to see a detection for Active Directory but not its equivalent for Entra ID, or the other way around. Why would a technique be worth detecting on one platform but not the other? Why detect it in Azure but not in AWS?

## Start with an inventory

It sounds hard to believe, but sometimes (usually, I would say) a SOC doesn't have a complete overview of the platforms that make up the organization. And you can't monitor what you don't know exists. You might be doing a great job covering Azure while having no idea your company keeps sensitive customer data in GCP.

So before selecting any use case, build an inventory of every area you have, or at least the ones you want to monitor.

## The non-negotiable detections

After many years of post-audit and post-incident embarrassment, I identified several detection categories that are non-negotiable. They are concepts, not specific rules, which means they can be applied, with some tailoring, to any area you want to cover.

1. **Source/telemetry health**. It does not make sense to build detections on top of unreliable log sources. If possible, monitor health per appliance, not per source: a single source can aggregate several servers, and if some of them fail, the rules keep running on the others while you quietly lose visibility. Many companies are required to store these logs for compliance. That means a prolonged ingestion gap is not just a blind spot, it is also a compliance problem.
2. **New privileged accounts**. Identify, in each area, which groups or roles count as privileged and monitor when accounts are added to them.
3. **Unused privileged accounts.** Monitor the use of privileged accounts that should never be used, such as break-glass accounts or built-in Administrators, as well as any change made to them. In general, almost any event involving these accounts should raise an alert.
4. **Critical platform change**. An architecture-level change to the platform's trust or authentication plane — a new federation trust, a schema change, a modification of the root or organization trust.
5. **Security-lowering platform change**. Any platform-level change that weakens your security posture: removing a conditional access policy, turning off a native security feature, deleting backups.
6. **Inbound remote-admin connections from outside.** Administering the internal network from an external address should go through controlled channels only — a bastion, the VPN, a gateway. Anything else, like a direct admin connection from outside, is something you want to know about. Identify the standard connection methods in each platform and write a rule for every one of them.

These categories share some properties that make them non-negotiable:

- **Impact.** Every one of these detects something critical: a disruption in your security monitoring, a downgrade of your security posture, or an incident in itself.
- **Easy implementation.** Once you have mapped the concept to your area and identified the action to detect, the rule itself is straightforward. It usually involves one or a few specific events, with no behavioral baseline and no external feeds.
- **Near zero false positives.** These detections barely produce false positives, because they all describe actions you must know about. Even when performed legitimately, someone has to confirm it was intentional, and revoke it when it goes against the security policy.

This list is not exhaustive. Every organization is its own world, and you may identify categories I haven't. The properties, though, are universal: **if a category is easy to implement, barely produces false positives, and has real impact, it is non-negotiable.**

## The almost non-negotiable detections

Once all the above categories are implemented across **all areas**, you can move on to the next detection step. These categories don't have the impact of the detections above, they are usually not so easy to implement and sometimes it can be a pain to deal with the false positive rate. That is why they don't belong to the previous group.

1. **IOC checking.** I am not a big fan of IOCs, but they can be very useful sometimes, especially to spot forgotten devices with no EDR. In my experience, the key point is to keep it simple: just high-confidence indicators, such as Tor exit nodes or fresh feeds from a very reliable source.
    - For inbound traffic, only successful connections matter: a failed sign-in from a malicious IP is just someone trying a password, but a successful one is a compromised account.
    - For outbound, success is irrelevant. A blocked outbound connection to a well-known IOC is still a real incident. If the device is trying to reach a C2, it is compromised.
2. **Disabling the security control itself.** Any EDR or security tool being turned off, any disabling of logging, any log clearing. It is in this tier because, in practice, it creates more false positives than you would expect: agent updates, maintenance windows, etc.
3. **Execution of known hacking tools.** No behavioral detection needed here, just the name and signature. Some people say it is not worth detecting AzureHound by user agent because it can be easily changed, but then they don't implement an alternative detection either. Imagine missing AzureHound even when the attacker was sloppy enough to leave the default user agent.

## Where MITRE helps

All these categories fit most environments because they don't depend on your products or your industry. Add whatever categories you found non-negotiable in your own environment. **Treat them like a template. You just need to take the areas from your inventory and then instantiate each concept on each of them.** This is where MITRE comes in handy to give you specific techniques worth monitoring.

One thing before jumping into rule creation. **Once you have identified a detection you want to deploy, check first whether it is already covered by built-in tooling in that area**: EDR, Microsoft Defender for Identity, Entra ID Protection, whatever cloud security platform you have in place. Developing and maintaining use cases takes real effort, so there is no point reinventing the wheel.

## What comes after

Once the detections above are implemented, you can move on. You are probably missing many standard use cases that these categories deliberately leave out: unusual sign-in, impossible travel, brute force, network scanning. They matter, but they are not a priority, and they should not be built before the bare minimum. In practice they are also much harder than they look: they depend on thresholds and baselines that only make sense once you know the environment well.

## It works because it is trivial

These categories may feel basic, even trivial. That is because they are, and yet I bet most environments are missing at least a few of them.

My intention was never to write another list of use cases, but to create a method that allows anyone to write their own.

Would you have caught the inbound SSH?
