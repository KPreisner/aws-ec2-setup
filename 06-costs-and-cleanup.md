# 6. Costs and cleanup

Approximate `us-east-2` on-demand prices (October 2026). Check current ones with the
[AWS Pricing Calculator](https://calculator.aws/).

| Item | Rate | Per month if left on |
|---|---|---|
| m7i-flex.2xlarge | $0.383/hour | ~$280 nonstop |
| gp3 storage (40 + 100 GiB) | $0.08/GiB-month | ~$11 (billed even when stopped) |
| Public IPv4 / Elastic IP | $0.005/hour | ~$3.60 (billed even when stopped) |
| **Total** | | **~$295 nonstop** |

Each day resized to m7i-flex.4xlarge adds about $9 over the 2xlarge rate ($0.766 vs $0.383 an
hour). Don't leave it on 4xlarge: nonstop, that's ~$575/month, more than double, and possibly past
your spend limit, which pauses the whole project.

Credits are used up first, then the card is charged. A project spend limit pauses
the project when reached, so set budget alerts below it. Data transfer *in* is free; *out* to the
internet is $0.09/GB after the first 100 GB a month, which a demo won't reach.

## Alerts

In the **Billing and Cost Management** console → **Budgets → Create budget** → Monthly cost
budget below your spend limit (e.g. $300 for a $500 limit), with email alerts at 50%, 80%, and
100%, so you hear about it well before the limit pauses the project. Remaining credits:

```
aws freetier get-account-plan-state
```

## Stop vs. terminate

| | Stop | Terminate |
|---|---|---|
| Instance-hours | stop billing | stop billing |
| EBS volumes | kept, still billed (~$11/month) | root deleted; data kept (Delete on termination: No) |
| Elastic IP | kept, still billed | stays allocated, still billed until released |
| Undo | Start it again | Gone |

Stop it at the end of a session: **EC2 → Instances → select → Instance state → Stop**. Or from the
Mac:

```
aws ec2 stop-instances --instance-ids <instance-id>
aws ec2 start-instances --instance-ids <instance-id>
```

Stop and start (not reboot) also moves the instance to new hardware, which fixes a failed
*system* status check.

## Backups

Snapshot the data volume before risky changes: **EC2 → Volumes → select → Actions → Create
snapshot**. Snapshots are incremental and cost $0.05/GiB-month for the changed blocks. A
snapshot can also become a new volume in a different Availability Zone, which is how you move
data to an instance in another zone.

## Deleting everything

When the box is done:

1. **Instances:** turn off Termination protection (Actions → Instance settings), then Terminate.
2. **Elastic IPs:** select → Release.
3. **Volumes:** delete the data volume (State "available") if you don't need it. Snapshot it first
   if you might.
4. **Snapshots** and **Security Groups** you no longer use.
5. Check **Billing → Bills** the next day for anything still accruing.
