# MVPs for Data-Producing Enterprise Security Tools

1. TOC
{:toc}

## Introduction

During my first pentesting job out of college, I happened to be reading Neuromancer. Working on web application security tests by day and reading about Case and co. by night, I couldn't help but think how helpful it would be to have my own Dixie Flatline - some autonomous agent who could assist me with the finer points without reaching out to (human) senior talent. 

As my competence grew, and web testing began to feel more rote, I couldn't help but feel much of the work I was doing could be automated. How wonderful would it be if I could spend most of my pentesting engagement focused on arcane one-off bugs instead of checking for necessary but boring-to-test vulnerabilities. And how useful it would be to customers! I'm confident nearly all security testers have had that thought at one point or another. 

With only that offensive security experience to speak of, I left my job for grad school in data science, where I learned about deep learning and started to pull together the technical pieces that might be necessary to construct intelligent automated security tools. I've given this a fair amount of [thought](https://sjcaldwell.github.io/2020/04/28/deep-rl-infosec.html), and started to build some [software](https://github.com/phreakAI/MetasploitGym) that I think, scaled up with a reasonable amount of training data, could start to technically perform some interesting work.

While working on this software and thinking about these problems, it's important to recognize my experience was only on the offensive side of things. I had never been on a blue-team, and had very little understanding of the circumstances that resulted in the vulnerabilities I found in offensive security in the first place.

These days I'm more interested in productionizing enterprise security software. With the interest in selling a security product, there was a key piece of experience missing: I knew nothing whatsoever about the usability of enterprise security software. Once my red-team or web-app testing reports were passed off to the internal security team, what did they do with it? I had a general sense that when working with the same companies I tended to find the same bugs, those lazy developers never seemed to get around to remediating the vast majority of my cool bugs.

This year, for the first time, I had the experience of being on the defensive side of things. I got a surface level view of the problems that need to be solved on a daily basis by the blue team, and its provided a new bar of what's required for this far-off automated pentester to be useful. I'm writing this up partially as a note to myself, and partially to those who find themselves building tools for a blue-team they've never been a part of directly.

## The current experience: too much noise, not enough signal

Run the average scanning tool. Run five of them. Cross-reference the ones from different tools. Dozens of "findings" for each hosted web application, for each domain. If you have one or two of these, no problem. But if you're of the size and scale that you would even have an internal security team anyway, the findings are probably in thousands if not the millions. 

So you've got millions of findings, a team of maybe a dozen, and your job isn't so much to make sure that every single thing gets resolved, but to prioritize and ensure that every vulnerability is at least _tracked_ and that anything critical is resolved in a reasonable amount of time. 

The team is going to be invariably overwhelmed. So, should you be designing for this hypothetical team, consider the following: 

## Human readable output is not enough

The days where it's acceptable to have only human readable output in a provided report is in the past. When you give a client an engagement report, you're handing off _work_. Your work might be done, but their work is just starting. They're going to take your report, triage the findings, prioritize and remediate those findings, and finally re-test to ensure their remediation was successful.

This work needs to be trackable. These days at a large organization, work is tracked in ticketing systems. You must provide easy-to-use integrations to the systems your hypothetical customers are already using. You want to be easy to drop in with what's common. Track vulnerability remediation in Asana or Jira, flag criticals to Slack or Teams, export data to Splunk. 

These integrations will invariably not cover everything you need, so it's important to have a well-documented, easy-to-use, and relatively bug-free API so that data can be pulled out of these systems.

Crucially, these API-keys must be able to be generated as service-accounts and scoped to the appropriate level by administrators. Nobody wants their custom-written data ingestion pipelines to fail because the person who wrote the code left their job! These all seem like quality-of-life features that early adopters wouldn't require, but at the size of organizations who are facing these problems, it's unlikely any of your findings will even be triaged if they can't slot easily into current workflows. 

## False positives are high. Stupid high.

As a machine-learning person, it's not surprising to me that reported false positive rates are basically garbage. Say a scanner claims to have a 1% false-positive rate. That means that in whatever test set they constructed (you don't get to see those) they only had 1% of false positives. Say they had 100 vulnerabilities they looked at and 1 falses positive. There you go. 

With enough tests, you can construct an arbitrary false-positive set that is at least defensible, from a marketing perspective. This will have no bearing on your reality when observing the software yourself. 

Having now been given the job to triage a system being tested in production to find how many of the vulnerabilities were being reported accurately, that verification is completely mind-numbing work. So much of your time is spent just trying to figure out how such-and-such finding could have possibly gone off that you almost forget your job is just to confirm them or not confirm them. As you go down the list of vulnerability after vulnerability, finding that highs and criticals are almost always false-positive (it's hard to know if your cross-site scripting was really "reflected" in a way that would execute) and informationals tend to always be accurate because they're based on some regular expression (that cookie _could_ use the proper flags - does anybody really care?) you find yourself scrolling through the list. Your brain doesn't want to review the materials because you know your time is being wasted, and ultimately it just becomes another alarm blaring in your room full of alarms you were already ignoring. If the alarms are always going off, the alarms are never going off.

However you get to accuracy, you have to have it. False positives are an easy answer to a cover-your-ass question. If some vulnerability is exploited, you _need_ to be able to say that your tool found it, and it's simply that your analysts didn't act on the information. It won't matter that your tool provided 10,000 equally menacing-looking but totally useless pieces of information. Your tool can't change any of that if it's loud.

Think of technologies like [Burp Collaborator](https://portswigger.net/burp/documentation/collaborator): by providing internet accessible services and attempting to trigger vulnerabilities that cause a target to reach out to external entities, they're able to provide higher confidence. Where possible, I think tools should be able to confirm the existence of vulnerabilities with these kinds of methods, and ranked confirmed vulnerabilities more highly. General communication around the confidence of an individual result to the user is useful, so surface it! Is this just a regular expression finding? Was there some kind of external trigger that was confirmed? Did you spin up a browser to confirm the cross-site scripting popped? Surface this context to the user and allow them to prioritize their work based on that information.


## Be Readable to Developers

Security isn't just for analysts. At an enterprise scale, it can't be. Most of the vulnerabilities are going to occur in code. Whether from application developer's source code, or the terraform written by a devops or sysadmin that defines the configurations of the cloud software powering most company's IT infrastructure. Ultimately, these security issues will need to be resolved by these workers anyway, so it's important the the "findings" be legible to them.

The reproduciblity steps, impact, and remediation of the vulnerability must be clearly communicated. It can be useful to base some of this remediation/description information on templates that can be changed at will by the owner of the software, giving them more freedom to meet the developers at their level of seniority and experience when explaining remediation and impact. 


## MVP includes usability

MVP for security products doesn't just mean the technology _works_ it means it's usuable. An engine doesn't mean anything until it moves a car. The ultimate goal of a security product isn't to find vulnerabilities before an attacker does, but to empower the security, operations, and development teams to understand their potential impact and resolve them. 

Consider the following features in your pre-release roadmap before you attempt enterprise sales: 

- False positives are low (communicate confidence/confirmation of a vulnerability when possible!)
- A full-featured API for exporting the data (with API keys tied to service accounts, not user level)
- Built-in integrations with common enterprise tools - don't lock your useful vulnerability information behind yet another application
- Configurable description/remediation information with a bend towards developers

If you've got any other features you think are a requirement, shoot me an email! 