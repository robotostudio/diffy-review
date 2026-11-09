# diffy-review

Sends a repository's pull request events to Diffy, Roboto's code review, without a webhook and
without a shared secret. Use it when Roboto's review account, `roboto-pr`, has **Write** on your
repository rather than Admin.

## Use it

1. Ask Roboto to bind the repository for the Action. Until then, Diffy accepts the events and reviews
   nothing.
2. Commit this file on the repository's default branch, as `.github/workflows/diffy-review.yml`:

```yaml
name: Diffy review
on:
  pull_request_target:
    types: [opened, synchronize, ready_for_review]
  issue_comment:
    types: [created]
permissions:
  id-token: write
jobs:
  review:
    if: github.event_name == 'pull_request_target' || (github.event.issue.pull_request && contains(github.event.comment.body, '@roboto-pr'))
    runs-on: ubuntu-latest
    steps:
      - uses: robotostudio/diffy-review@v1
```

## What it does

When a pull request opens, gets new commits or is marked ready, or a comment on one names
`@roboto-pr`, the action:

1. asks GitHub for an OIDC token with the audience `https://eve-review.vercel.app`. GitHub signs it,
   and it names the repository, its owner and the event;
2. posts the event GitHub gave the workflow to `https://eve-review.vercel.app/eve/v1/github`, with the
   token as a bearer.

Diffy checks the token against GitHub's keys, and checks that the repository and the owner it names
are the ones you agreed with Roboto. It needs no secret from you.

The action checks out no code and reads no secret. `id-token: write` is the only permission the
workflow gives it. It needs `bash`, `curl` and `jq`, which GitHub's hosted runners have.

## Why `pull_request_target`

`pull_request_target` and `issue_comment` run the default branch's copy of the workflow. A
`pull_request` run uses the pull request's own copy, which its author can change, so Diffy refuses
it. Pull requests from forks are covered for the same reason. The action never runs the pull
request's code.

## What it does not cover

- A `@roboto-pr` in a comment on a line of the diff. Write it on the pull request's conversation.
- Preview deployments. Diffy does not see them, so a review never opens one.
- A repository or organisation with GitHub Actions turned off.

Each run costs a few seconds of your GitHub Actions time. The `if:` in the file keeps a plain comment
from starting a run.
