# Application Healthchecks

## Summary

We currently use application healthchecks for three things:

- For load balancing, to determine if an instance is ready to serve
  requests.
- For continuous deployments, to determine (along with other smoke
  tests) if a release can be automatically pushed to production.
- For alerting, so that 2ndline know if an instance has a problem
  which needs manual intervention to fix.

We implement healthchecks using the [`GovukHealthcheck`][] module in
[govuk_app_config][].

Currently, a healthcheck response is always served with an HTTP status
of 200, and indicates in the body whether the instance's status is
"ok", "warning", or "critical".  However, AWS load balancers use the
HTTP status, not the response body, to determine whether an instance
is healthy.  So we will continue to send requests to an instance which
is in a "warning" or "critical" state.

This RFC proposes that we standardise on our healthchecks returning a
200 or a 500 status code, which has some further ramifications on
monitoring and alerting (discussed below).

A full list of healthchecks which will need updating is given at the
end of the document.

## Problem

We overload the meaning of "healthcheck", and cover both *application
health* as well as *instance health* with our `/healthcheck`
endpoints.  This introduces confusion, and means that we cannot
currently use healthchecks in load balancing.

Here are some examples of healthchecks which are suitable for load
balancing and for post-deployment checks:

- [`GovukHealthcheck::ActiveRecord`][]
- [`GovukHealthcheck::Mongoid`][]
- [`GovukHealthcheck::RailsCache`][]
- [`GovukHealthcheck::Redis`][]
- [`GovukHealthcheck::SidekiqRedis`][]

If an app can't talk to a backing service it relies on, then it
probably can't be trusted to handle any requests.  One of these
healthchecks failing could indicate a networking or a configuration
issue.

Here are some examples of healthchecks which are unsuitable for load
balancing or for post-deployment checks:

- [`GovukHealthcheck::SidekiqQueueCheck`][], this is an abstract check
  which other checks extend.
  [`GovukHealthcheck::SidekiqQueueLatencyCheck`][] is an instance of
  this check.
- [`GovukHealthcheck::ThresholdCheck`][], this is another abstract
  check which other checks extend.
  [`GovukHealthcheck::SidekiqRetrySizeCheck`][] is an instance of this
  check.
- [`Healthcheck::ApiTokens`][] in [signon][], which reports that an
  API token needs rotating.

If one of these healthchecks fail, the app instances themselves are
probably fine.  The problem is elsewhere.

Two of these indicate capacity problems outside of the instance, and
the third is an alert that a manual maintenance procedure needs to be
performed some time in the next two months.  They all share the
property that if one instance reports a failure, all will.  And so if
we use these healthchecks as part of load balancing rules, all running
instances of the app will be taken down simultaneously.

Even if we don't make our healthchecks suitable for load balancing, as
GOV.UK moves towards continuous deployments, we will end up in
situations where an automatic deployment is aborted because of
something unrelated to the change being deployed.  That's not good.

A healthcheck should check that the instance is running and isn't
completely broken.  That's all.

## Proposal

My proposal comes down to three parts:

1. Remove the "warning" state of `GovukHealthcheck`, so a healthcheck
   result is either "ok" or "critical".
2. Make `GovukHealthcheck` serve an HTTP status of 500 for a
   "critical" result.
3. Add a separate healthcheck for liveness, to use with the ECS
   container lifecycle.

But we can't make this change right now, we have a migration path to
follow first.

### Remove the "warning" state

There are four possible semantics we could assign the "warning" state:

| # | State    | Allows automatic deployments? | Allows requests to be sent to the instance? |
| - | -------- | ----------------------------- | ------------------------------------------- |
|   | ok       | yes | yes |
| 1 | warning  | yes | yes |
| 2 | warning  | yes | no  |
| 3 | warning  | no  | yes |
| 4 | warning  | no  | no  |
|   | critical | no  | no  |

- If we go for option 1, then "warning" is the same as "ok".
- If we go for option 2, then "warning" will let us deploy unusable releases.
- If we go for option 3, then "warning" will block deployments.
- If we go for option 4, then "warning" is the same as "critical".

The most sensible option is 3, which is our current behaviour.

But does it really gain us anything over having separate alerts for
the specific condition we need to know about?

By removing the "warning" state, we will remove some spurious alerts
which don't add value, and add more specific alerts for ones which do.

### Make `GovukHealthcheck` return a 500 status for critical failure

This is a requirement for `GovukHealthcheck` to be usable with load
balancers.

### Have separate healthchecks for liveness and readiness

If `GovukHealthcheck` checks whether an instance is ready to handle
requests, and takes it out of the load balancer target group if not,
then it is a *readiness* check.

There is also value in having a separate *liveness* check, which
simply checks if the instance is running and reachable, and doesn't
check any of its dependencies:

```ruby
get "/healthcheck", to: proc { [200, {}, []] }
get "/healthcheck/ready", to: GovukHealthcheck.rack_response(...)
```

When we have replatformed to ECS, this healthcheck can be used to
determine if a container needs to be automatically restarted, for
example if the process has crashed.

In a sense we already have both liveness and readiness healthchecks:
the liveness part comes from `GovukHealthcheck` always returning an
HTTP status of 200, and the readiness part comes from reading the JSON
response body.  But our load balancers cannot read the response body,
so in practice we can't make use of our readiness check.

### Migration path

We can't start serving non-"ok" healthchecks with an HTTP status of
500 right now, as this will take down entire apps due to unsuitable
healthchecks.  We will need to do this in stages:

1. List all the healthchecks which need changing, because they don't
   indicate a critical failure of the app (done, see the appendix)
2. For each such healthcheck:
   - remove it if it's not adding value, or
   - add a separate alert if it is.
3. Once all healthchecks are updated, change `GovukHealthcheck` to
   serve the appropriate HTTP status code.

## Appendix: Healthchecks which need changing

The following healthchecks need updating before `GovukHealthcheck` can
serve an HTTP status of 500:

| App or Gem | Check | Reason |
| ---------- | ----- | ------ |
| [govuk_app_config][]  | [`GovukHealthcheck::SidekiqQueueCheck`][]           | checks for a capacity issue |
| [govuk_app_config][]  | [`GovukHealthcheck::SidekiqQueueLatencyCheck`][]    | checks for a capacity issue |
| [govuk_app_config][]  | [`GovukHealthcheck::SidekiqRetrySizeCheck`][]       | checks for a capacity issue |
| [govuk_app_config][]  | [`GovukHealthcheck::ThresholdCheck`][]              | checks for a capacity issue |
| [content-publisher][] | [`Healthcheck::GovernmentDataCheck`][]              | a failure doesn't seem to totally impair the instance |
| [finder-frontend][]   | [`Healthcheck::RegistriesCache`][]                  | a failure doesn't seem to totally impair the instance |
| [publisher][]         | [`Healthcheck::ScheduledPublishing`][]              | checks for a capacity issue |
| [publishing-api][]    | [`Healthcheck::QueueLatency`][]                     | checks for a capacity issue |
| [search-api][]        | [`Healthcheck::ElasticsearchIndexDiskspaceCheck`][] | checks for a capacity issue |
| [search-api][]        | [`Healthcheck::RerankerHealthcheck`][]              | checks for an error which is gracefully handled |
| [search-api][]        | [`Healthcheck::SidekiqQueueLatenciesCheck`][]       | checks for a capacity issue |
| [signon][]            | [`Healthcheck::ApiTokens`][]                        | checks for an upcoming maintenance task |

Notes:

- [asset-manager][] isn't using `GovukHealthcheck`, but it has [an equivalent implementation][].
- [content-data-admin][] has a nonstandard healthcheck which reports an error to Sentry.
- [finder-frontend][] has a `/healthcheck` and a `/healthcheck.json` which do different things.
- Various apps don't have any healthcheck endpoint at all.
- Various apps have a healthcheck endpoint which is just `proc { [200, {}, []] }`.

[`GovukHealthcheck`]: https://github.com/alphagov/govuk_app_config/blob/b5f76dd7920ccee294c6e862336265980c9eb323/lib/govuk_app_config/govuk_healthcheck.rb
[`GovukHealthcheck::ActiveRecord`]: https://github.com/alphagov/govuk_app_config/blob/b5f76dd7920ccee294c6e862336265980c9eb323/lib/govuk_app_config/govuk_healthcheck/active_record.rb
[`GovukHealthcheck::Mongoid`]: https://github.com/alphagov/govuk_app_config/blob/b5f76dd7920ccee294c6e862336265980c9eb323/lib/govuk_app_config/govuk_healthcheck/mongoid.rb
[`GovukHealthcheck::RailsCache`]: https://github.com/alphagov/govuk_app_config/blob/b5f76dd7920ccee294c6e862336265980c9eb323/lib/govuk_app_config/govuk_healthcheck/rails_cache.rb
[`GovukHealthcheck::Redis`]: https://github.com/alphagov/govuk_app_config/blob/b5f76dd7920ccee294c6e862336265980c9eb323/lib/govuk_app_config/govuk_healthcheck/redis.rb
[`GovukHealthcheck::SidekiqQueueCheck`]: https://github.com/alphagov/govuk_app_config/blob/b5f76dd7920ccee294c6e862336265980c9eb323/lib/govuk_app_config/govuk_healthcheck/sidekiq_queue_check.rb
[`GovukHealthcheck::SidekiqQueueLatencyCheck`]: https://github.com/alphagov/govuk_app_config/blob/b5f76dd7920ccee294c6e862336265980c9eb323/lib/govuk_app_config/govuk_healthcheck/sidekiq_queue_latency_check.rb
[`GovukHealthcheck::SidekiqRedis`]: https://github.com/alphagov/govuk_app_config/blob/b5f76dd7920ccee294c6e862336265980c9eb323/lib/govuk_app_config/govuk_healthcheck/sidekiq_redis.rb
[`GovukHealthcheck::SidekiqRetrySizeCheck`]: https://github.com/alphagov/govuk_app_config/blob/b5f76dd7920ccee294c6e862336265980c9eb323/lib/govuk_app_config/govuk_healthcheck/sidekiq_retry_size_check.rb
[`GovukHealthcheck::ThresholdCheck`]: https://github.com/alphagov/govuk_app_config/blob/b5f76dd7920ccee294c6e862336265980c9eb323/lib/govuk_app_config/govuk_healthcheck/threshold_check.rb
[`Healthcheck::ApiTokens`]: https://github.com/alphagov/signon/blob/694b123062218bf87e457b4b0f36d76c2fe3045d/lib/healthcheck/api_tokens.rb
[`Healthcheck::GovernmentDataCheck`]: https://github.com/alphagov/content-publisher/blob/c1b0d0e1bb05413dd2102ec1eadb485344890c7f/lib/healthcheck/government_data_check.rb
[`Healthcheck::ScheduledPublishing`]: https://github.com/alphagov/publisher/blob/2f63ea01c6b504b0a50657f5d3bbbbd3c6542257/app/models/healthcheck/scheduled_publishing.rb
[`Healthcheck::QueueLatency`]: https://github.com/alphagov/publishing-api/blob/c0c69997d1a1c63bdea3581655dd4c3146281443/app/models/healthcheck/queue_latency.rb
[`Healthcheck::SidekiqQueueLatenciesCheck`]: https://github.com/alphagov/search-api/blob/339220141d28d3496af85ab5e40c3fe457c4051d/lib/healthcheck/sidekiq_queue_latencies_check.rb
[`Healthcheck::RerankerHealthcheck`]: https://github.com/alphagov/search-api/blob/339220141d28d3496af85ab5e40c3fe457c4051d/lib/healthcheck/reranker_healthcheck.rb
[`Healthcheck::ElasticsearchIndexDiskspaceCheck`]: https://github.com/alphagov/search-api/blob/339220141d28d3496af85ab5e40c3fe457c4051d/lib/healthcheck/elasticsearch_index_diskspace_check.rb
[`Healthcheck::RegistriesCache`]: https://github.com/alphagov/finder-frontend/blob/60d0fb2d10e503e33c4e4c3da86731d4700b532f/app/lib/healthchecks/registries_cache.rb

[govuk_app_config]: https://github.com/alphagov/govuk_app_config
[asset-manager]: https://github.com/alphagov/asset-manager
[content-data-admin]: https://github.com/alphagov/content-data-admin
[content-publisher]: https://github.com/alphagov/content-publisher
[finder-frontend]: https://github.com/alphagov/finder-frontend
[publisher]: https://github.com/alphagov/publisher
[publishing-api]: https://github.com/alphagov/publishing-api
[search-api]: https://github.com/alphagov/search-api
[signon]: https://github.com/alphagov/signon

[an equivalent implementation]: https://github.com/alphagov/asset-manager/blob/e8b25f21a72948117c7dcb084491af26b949805a/app/controllers/healthcheck_controller.rb
