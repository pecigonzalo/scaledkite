# Scaledkite
### Dynamic Buildkite Agent scheduling with AWS EventBridge, Lambda, and EKS.

Scaledkite helps you reduce AWS spend on your CI workers by only running workers when you need them while also helping you avoid build queuing by having a limitless (in theory) number of workers available.

_Note: Scaledkite can really only help you reduce CI spend if you're already paying for an EKS cluster than you can run your workers in._

# How it works
Buildkite is one of the few providers that is an AWS-recognized integrator of [AWS EventBridge](https://aws.amazon.com/eventbridge/). EventBridge is a hosted message bus that allows you to attach rules to an event bus that are evaluated on every message sent to the bus. Those rules can do things like trigger Lambda events, which is what we're going to do here! Buildkite supports sending messages to EventBridge for a number of different events -- those are documented [here](https://buildkite.com/docs/integrations/amazon-eventbridge) -- but we're just going to focus on the **Job Scheduled** event.

Whenever EventBridge recieves a message on our Buildkite event bus, it will evaluate a rule that checks to see if the `detail-type` of that event is "Job Scheduled". If it is, we'll have the rule kick off our Lambda function with the payload it receieved.

# Setting up the Kubernetes part
(We're making assumptions that you already have an EKS cluster, you already have an IAM role configured with cluster access that your Lambda function can assume, and you're running your agents in the `buildkite` namespace)

1. Create a secret with your Buildkite token:
```
$ kubectl create secret generic buildkite-agent-token --from-literal token=INSERT-AGENT-TOKEN-HERE --namespace=buildkite 
```
2. Create the `buildkite-env-vars` secret containing the following keys and values: `DOCKER_LOGIN_USER`, `DOCKER_LOGIN_PASSWORD`, `GITHUB_TOKEN`. (`DOCKER_LOGIN_*` env vars are our Docker Hub bot account login, `GITHUB_TOKEN` is a custom env var we use for installing private gems, etc. from GitHub in CI)
3. Create the `buildkite-agent-git-credentials` secret using a `git-credentials` file for a bot account as documented [here](https://git-scm.com/docs/git-credential-store#_storage_format):
```
$ kubectl create secret generic buildkite-agent-git-credentials --from-file=./git-credentials --namespace=buildkite
```

# Setting up the AWS part
1. Create your Lambda function using the payload generated by `make build`. The handler should be `main`, runtime is `go1.x`, timeout `30`, and memory `128mb`. You'll need to setup a few environmental variables in your function, too -- they're documented below.
2. Follow Buildkite's instructions (here)[https://buildkite.com/docs/integrations/amazon-eventbridge#configuring] on setting up AWS EventBridge notifications within the Buildkite and AWS consoles.
3. Create a rule on your `aws.partner/buildkite.com/......` event bus to trigger your Lambda function. For the rule pattern, select `Event pattern` -> `Custom pattern`, and fill in:
```
{
  "account": [
    "<your AWS account number>"
  ],
  "detail-type": [
    "Job Scheduled"
  ]
}
```
In the target section, select `Lambda function` and then select your newly-created function.

# Configuration
The following environmental variables can be configured on your Lambda function for ScaledKite:

- [required] `cluster` - the EKS cluster to authenticate to
- [required] `arn` - the IAM role ARN that Scaledkite should use for EKS cluster access
- [required] `buildkite_token` - your Buildkite agent token
- [optional] `namespace` - the namespace your worker agents will run in
- [optional] `pod_prefix` - the prefix used for created k8s jobs/pods
- [optional] `image` - the `buildkite-agent` docker image to use -- remember that you'll need a custom one with the `pre-exit` hook (see #Quirks)
- [optional] `region` - the region your EKS cluster is in, not required if your Lambda function is running in the same region as your cluster

# Quirks
- **This tool runs Docker in Docker containers as privileged in your cluster. This can be somewhat mitigated by running Buildkite pods on segregated nodes (which we do here via workload selectors).**
- Scaledkite only creates workers for jobs where the `agent_query_rules` are `queue=dynamic`. See the test CI script in `ci/` for an example. (or fork this and edit it to match what you need)
- You probably shouldn't use Scaledkite for steps involving image builds, there's no layer caching support. (We split image builds off into a separate queue at Basecamp)
- Scaledkite relies on a custom `buildkite-agent` Docker image that has a `pre-exit` hook that deletes the `docker-dind` sidecar from the worker pod. Until [proper sidecar support](https://github.com/kubernetes/enhancements/issues/753) lands in Kubernetes, this is one of the better options that doesn't require weird entrypoint signal handling to get our `docker-dind` container to shut down when the `buildkite-agent` container does.
- There's an off-chance that a message to EventBridge isn't delivered and an agent isn't scheduled for a specific task. It could be worth running an agent or two with `BUILDKITE_AGENT_TAGS` set to `queue=dynamic` to pick them up.

# Todo
- Enable Fargate support. It's simple -- just add the annotation to the generated Job config, but we don't use it at Basecamp because we use Docker in CI.
- Move `buildkite_token` to a real secret that isn't a Kubernetes secret.
- Make more things configurable (resource requests, etc.)
- Stop relying on the `pre-exit` hook to stop the `docker-dind` container
- Accept a string of ECR account IDs and regions to authenticate to in the `environment` hook, rather than just assumed we only need us-east-1 in the account the agents are running in.
- Switch to SSH keys for GitHub auth

# Credits
- EKS cluster authentication code from https://github.com/nbrandaleone/eksClient