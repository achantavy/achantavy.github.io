---
layout: post
author: alex
title:  "Rambling misadventures in Cartography error handling - Part 1"
categories: cartography reliability
date: 2020-10-14 10:44:00 -0700

---
This evening I tried adding a feature to handle Cartography exceptions consistently. I started by adding a new CLI arg:

```python
parser.add_argument(
    '--on-exception',
    type=str,
    default='continue',
    help=(
        'Valid values: "continue" (default) or "stop". Determines what cartography will do when it encounters'
        'an unhandled sync exception. "continue" attempts to skip failed instructions and keep going, and'
        '"stop" makes the sync stop.'
    ),
```

And then I added it to the config object itself:

```python
class Config:
	...
	self.on_exception = on_exception
	...
```

The trouble came when I tried to handle it in a sync. I first thought to do it at the sync level: if the AWS sync fails, then there's no reason why we should make the GCP sync not even start.

But then I thought, we will almost _definitely_ want more fine-grained control on this behavior; if the AWS **IAM** sync fails, then we should still continue with the AWS **EC2** sync. This meant that I would need to pass the global config object into each intelmodule.

The chain looks like this: 

```python
def start_aws_ingestion(neo4j_session, config):
```

calls

```python
_sync_multiple_accounts(neo4j_session, aws_accounts, config.update_tag, common_job_parameters)
```

which calls

```python
_sync_one_account(neo4j_session, boto3_session, account_id, sync_tag, common_job_parameters)
```

which then calls
```python
iam.sync(neo4j_session, boto3_session, account_id, sync_tag, common_job_parameters)
```

As you see, I would need to either alter the function signature of everything in that chain to pass in `config.on_exception`, or I would need to supply it as a `common_job_parameter`. I opted for the latter.

And then comes the matter of actually handling the exception. What would this look like?

In `iam.sync()`, I could do something like

```python
def sync(neo4j_session, boto3_session, account_id, update_tag, common_job_parameters):
    logger.info("Syncing IAM for account '%s'.", account_id)
    on_exception = common_job_parameters['on_exception']
    
    functions = [
        sync_users, sync_groups, sync_roles, sync_group_memberships, sync_user_access_keys
    ]
    for func in functions:
        try:
            func(neo4j_session, boto3_session, account_id, update_tag, common_job_parameters)
        except Exception as e:
            if on_exception == 'stop':
                raise
            else:
                logger.error(e)
                continue
```

The problem here though is that new module authors would need to copy this (garbage) boilerplate code and this is not a scaleable, general enough solution.

Let's step back a bit, what am I trying to do here?

I need the sync to be more reliable. If it fails, then I need to know about it, but I don't want it to give up on the rest of my data because a full sync run takes about 10 hours and I don't want to wait 12 hours for the next cron job to start, and I also don't want to be bothered kicking off a manual run.

So, now I'm back to working on DAGs. I don't care about parallelism in DAGs just yet, I just want this sync to be a bit more reliable. One thing to keep in mind with DAGs though is that if there is a failure at any point, the Drift Detection sync stage cannot rely on the data because missing and incorrect nodes may fire false alerts.
