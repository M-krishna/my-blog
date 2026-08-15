+++
title = "AWS IAM, explained as a building full of locked doors"
date = 2026-08-15
description = "I learned IAM properly this week. Here is the mental model that made it click, plus five small exercises you can run in the console in under an hour."

[taxonomies]
tags = ["aws", "iam", "lambda"]
+++

I have been working on an application that runs on AWS for a while now, without really knowing what AWS was doing underneath. CloudFront, Lambda, stacks, roles. I could use them. I could not explain them.

So I sat down and learned IAM from first principles. This post is my attempt to explain it the way I wish someone had explained it to me. If you can already write a policy from memory, this is not for you. If your reaction to "just attach the right role" is a quiet panic, read on.

# Start with the problem, not the definition
Every IAM article opens by telling you that IAM stands for Identity and Access Management. That sentence taught me nothing, so let us skip it.

Here is the actual situation.

Imagine AWS is one enormous building. Millions of rooms. Some of the rooms are yours. A room with your files in it. A room with a machine that runs your code. A room with your customer list.

Your rooms are not on a private floor. They sit in the same corridors as everybody else's rooms. Netflix's rooms. A stranger's rooms. There is no separate wing for you, and no fence around your part of the building.

That is not a metaphor streched too far. It is literally true. Your S3 bucket is reachable at a public address. So is the API that creates and deletes your servers. There is no inside.

So there is exactly one thing standing between your data and the entire internet. For every single request, AWS has to decide: should this be allowed?

**IAM is that decision**. That is the whole service.

# Every door has a guard
In our building, every door is locked and every door has a guard.

The guard is not clever. You cannot reason with the guard, or explain that it is urgent. The guard does one thing. It reads a rulebook and asks three questions.
1. Who are you?
2. What do you want to do?
3. To which room?

Then it looks for a line in the rulebook that matches all three. If a matching line says yes, you go in. If there is no matching line, you do not go in.

That last part is the one that surprised me, so let me say it slowly.

**No rule means no.**

Not "probably fine". Not "well, nobody said you could not". Silence in the rulebook is a refusal. In AWS terms this is called default deny, and it means a brand new role can do absolutely nothing until you say otherwise.

This is why so many people's first AWS Lambda fails with Access Denied. Nothing is broken when that happens. The system is doing its only job.

One thing makes this whole design problem, and it is worth noticing: **everything in AWS is an API call**. Clicking a button in the console is an API call. Your code reading a file is an API call. There is no back door and no local mode. One single chokepoint means one single system can guard all of it.

# A policy is just a page in the rulebook
When AWS documentation says "policy", read it as "a page of rules the guard reads". That is all it is. A rule in plain English:

> Allow / the delivery robot / to take photos / from the storage room.

The same rule in AWS's language:
```json
{
    "Effect": "Allow",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-storage-room/*"
}
```

Four things. Yes or no. Who. What. Which room. When you open a policy that looks intimidating, find those four and ignore all the surrounding punctuation. Most real policies are just this same shape, repeated a few times in a list.

Go and look at one for yourself. In the console, open IAM, then Policies, then search for `AmazonS3ReadOnlyAccess` and click the JSON tab. Then open `AdministratorAccess`. It is four lines: allow every action on every resource. Admin is not a special mode. It is a wildcard.

# Deny always wins
There is one extra rule, and only one.

A rule can also say Deny. And a Deny beats every Allow, everywhere, always.

If one page says "let her in" and another page says "never let her in", she does not get in. It does not matter which page was written first, or how specific either one is. Red beats green.

That is the entire evaluation logic. Three lines:
1. By default, no.
2. Any matching Allow makes it yes.
3. Any matching Deny makes it no, and that is final.

Which means debugging Access Denied is always one of two things. Either no Allow matched, or a Deny did. In pratice it is almost always the first.

# Badges and costumes
Now back to the guard's first question. Who are you? How do you answer it?

**One way is a badge**. Your name, your photo, yours forever, kept in your pocket. In AWS this is called a user. It is meant for actual humans.

The problem with badges is that they get copied. Write your badge number on a whiteboard and someone else is now you. Email it to a collegue and there are two of you. Badges also do not expire, so a copy made three years ago still opens the door today, and you would never know it existed.

That is the real cost of long lived secret. It is not that it might leak once. It is that it quietly accumulate copies you cannot track. Logs, screenshots, an old git branch, someone's terminal history.

**The other way is a costume on a hook**. Hanging in the corridor is a uniform labelled "Cleaner". The costume is not a person. It belongs to nobody. It just hangs there.

When the cleaning robot starts its shift, it puts the costume on, and every guard in the building now treats it as the cleaner. At the end of the shift the costume goes back on the hook. And the best part: the costume falls apart after about an hour. Steal one and you have stolen something that turns to dust almost immediately.

In AWS, a costume is a **role**. In plain words:

> A role is a set of permissions that is not attached to anybody. Things put it on when they need it, and take it off when they are done.

If you only remember one thing from this post, remember that one.

# Every costume has two labels
This is the part I kept getting confused about, and it is simpler than I thought. A role carries two policies, and they answer completely different questions.

* The **trust policy** says who is allowed to wear this costume.
* The **permissions policy** says what doors it opens.

Who may wear it. What it can do. Two questions, both stored on the same role.

You can see this in the console. Open any role and there are two tabs. Trust relationships is the first label. Permissions is the second. Look at both until it feels boring, because once you see the two tabs as two different questions, roles stop being mysterious.

Worth noting: the trust policy is what makes a role a role. Take it away and you just have a bag of permissions that nobody can pick up.

# Two places to write the same rule
One more small piece, then we are done with theory.

You can write a rule on the visitor's page: "this robot may enter the storage room." Or you can pin the rule to the room's own door: "this room may be entered by that robot."

Both work. The guard reads both and adds them together. This is why AWS has two kinds of policy floating around in the docs. Identity policies are written from the visitor's side. Resource policies, like an S3 bucket policy, are written from the room's side. Same door, two places to write the rule.

Within one AWS account, either side is enough on its own. Across two different accounts, both sides have to agree independently. That is not redundancy. It means nobody else can hand out access to your rooms, and you cannot help yourself to theirs.

# Now go and break something
Reading about IAM did not teach me IAM. Watching a denial happen did. These take about an hour in total and cost nothing. Delete everything afterwards.

## 1. Make a role and look at its two side
Go to IAM, then Roles, then Create role. Trusted entity: AWS service, use case Lambda. Attach `AmazonS3ReadOnlyAccess`. Name it `my-practice-role`.

Now open it and read both tabs. Trust relationship says `lambda.amazonaws.com`, which is **who may wear it**. Permission says S3 read only, which is **what it opens**.

## 2. Ask the guard a hypothetical question
IAM has a policy simulator at the bottom of the left menu. Nothing gets created, you are only asking "would this be allowed".

Pick your role, tick the policy, choose S3 as the service, and tick both `GetObject` and `DeleteObject`. Run it.

GetObject is allowed. DeleteObject is denied. Same role, same second. One is allowed because a rule says so, the other is denied because no rule mentions it. That is default deny, visible.

## 3. Watch a real denial
Create a lambda function. Node.js runtime, and let it create its own new role. Paste this code and deploy:
```typescript
import { S3Client, ListBucketsCommand } from '@aws-sdk/client-s3';
const s3 = new S3Client({});
export const handler = async () => {
  const out = await s3.send(new ListBucketsCommand({}));
  return { buckets: out.Buckets.map(b => b.Name) };
};
```
Small trap I fell into here. The console creates the file as `index.mjs`, which is an ES module, so `import` works and `require` does not. If you paste `require(...)` from an older tutorial you get `require is not defined in ES module scope`. Useful thing to notice: that error arrives before AWS is even contacted, so it is a JavaScript problem, not a permissions problem. The guard was never asked anything.

Run the test. You get Access Denied. Lambda gave your function a role that can only write logs, and no rule anywhere mentions S3.

## 4. Fix it without touching the code
Go to Lambda configuration, then Permissions, and click through to the role. Attach `AmazonS3ReadOnlyAccess`. Then go straight back to the Test tab and run it again. Do not redeploy.

It works.

I found this genuinely surprising the first time. The code did not change. Nothing was deployed. The function's abilities changed while it sat there doing nothing. Permissions are not part of your program. They are a separate rulebook that the guards re-read every single time.

## 5. Allow one room, not every room
`AmazonS3ReadOnlyAccess` allows every bucket in the account, which is never what you actually want. So let us scope it down.

Create two buckets, `pratice-a-something` and `practice-b-something`. Names have to be gloabally unique, so add something random.

Change the function so the bucket comes in through the event, which lets you switch targets without redeploying:
```typescript
import { S3Client, ListObjectsV2Command } from '@aws-sdk/client-s3';
const s3 = new S3Client({});
export const handler = async (event) => {
  const out = await s3.send(new ListObjectsV2Command({ Bucket: event.bucket }));
  return { bucket: event.bucket, count: out.KeyCount ?? 0 };
};
```
Remove `AmazonS3ReadOnlyAccess` from the role. Add an inline policy instead:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "s3:ListBucket",
    "Resource": "arn:aws:s3:::practice-a-something"
  }]
}
```
Now test with `{ "bucket": "practice-a-something" }`. It works. Test with bucket b. Denied.

Same role, same action, same code. Different room, different answer. That is the difference between a toy policy and a real one.

That `Resource` line is an **ARN**, an Amazon Resource Name, which is just a unique address for one thing. Read it left to right: `arn`, the partition `aws`, the service `s3`, two empty slots, then the bucket name. The empty slots are where region and account normally go. S3 bucket names are globally unique already, so they are left blank.

## 6. The trap that catches everyone
Upload a text file into bucket a, and change the function to read it:
```typescript
import { S3Client, GetObjectCommand } from '@aws-sdk/client-s3';
const s3 = new S3Client({});
export const handler = async (event) => {
  const out = await s3.send(new GetObjectCommand({ Bucket: event.bucket, Key: event.key }));
  return { body: await out.Body.transformToString() };
};
```
Denied. Even though your policy very clearly grants bucket a.

Here is why. Your rule names the bucket, `arn:aws:s3:::pratice-a-something`. The file inside it is a different resource with a different address, `arn:aws:s3:::pratice-a-something/my-file.txt`. Listing a room and opening a box inside that room are two different permissions on two different things.

Add a second statement with `"Action": "s3:GetObject"` and `"Resource: "arn:aws:s3:::pratice-a-something/*"`, where `/*` means anything inside the bucket. Now it works.

When somebody loses an afternoon to an S3 permission error, this is usually it. Bucket and bucket slash star.

Delete your buckets, your function and your roles when you are done.

# The question that ties it together
Look back at exercise 3. That function is five lines long. There is no password in it. No key, no token, nothing I typed anywhere. And yet AWS knew exactly who was asking. It knew precisely enough to refuse, and then, after one change to a rulebook, to allow. So how did AWS know who my function was?

Here is the path, and it is the payoff for everthing above. The function has a role attached to it. That role's trust policy says the Lambda service is allowed to wear this costume. So when the function starts up, before my code runs at all, the Lambda service assumes the role and drops fresh temporary credentials into the environment. Then `new S3Client({})` looks around, finds them, and uses them without being told.

The costume was already on before my first line ran. And it expires by itself, so there is nothing to rotate and nothing to leak.

That is why every AWS guide tells you to use roles instead of access keys, and now I can say why rather than just repeating it.

# What I would tell my past self
* It is one question, asked over and over. Who, what action, which resource.
* No rule means no. Access Denied is usually the system working, not breaking.
* Deny beats Allow, always.
* A role is a costume, not a badge. Nobody owns it, anything approved can wear it, and it dissolves after an hour.
* Every role has two labels. who may wear me, and what I open.
* The bucket and the things inside the bucket are two different resources.