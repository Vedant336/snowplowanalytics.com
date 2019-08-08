---
layout: post
title: Iglu Scala Client 0.6.0 released
title-short: Iglu Scala Client
tags: [iglu, scala, json, jsonschema]
author: Anton
category: Releases
permalink: /blog/2019/08/09/iglu-scala-client-0.6.0-released/
---

We're tremendously excited to announce the new 0.6.0 release of the Iglu Scala Client, a library in charge of schema resolution and data validation in all Snowplow components, including enrichment jobs and loaders. This release brings enormous amount of API changes we've made in order to facilitate implementation of Snowplow Platform Improvement Proposals, including [new bad rows format][bad-rows-rfc], [Amazon Redshift automigrations][automigrations-rfc] and [deprecation of a batch pipeline][batch-deprecation-rfc].

In the rest of this post we will cover:

1. [API Changes](#api-changes)
2. [Semantic Changes](#semantic-changes)
3. [New Validator](#new-validator)
4. [Acknowledgements](#acknowledgements)
5. [Roadmap and upcoming features](#roadmap)
6. [Getting help](#help)

<!--more-->

<h3 id="api-changes">1. API changes</h3>

Since 0.6.0 Iglu Scala Client exposes a new class called Client which consists of two independent entities: Resolver and Validator. Resolver is responsible for schema resolution, caching and error handling and Validator receives resolved schema with datum user wants to validate and returns the validation report. These entities can be used separately or even re-defined by user, but it is recommended to use Client class as an abstraction for most common use case - validation of self-describing entities.

The Client class defines only one function:

{% highlight scala %}
def check[F[_]: RegistryLookup: Clock: Monad, A](instance: SelfDescribingData[A]): EitherT[F, ClientError, Unit]
{% endhighlight %}

Where:

* F[_] is an abstract effect type, requiring a [tagless final][tagless-final] capabilities RegistryLookup and `cats.effect.Clock` as well as instance of cats.Monad type class. Two most common concrete effect types are cats.effect.IO and Id, all necessary machinery for them provided out-of-box, but user can also define this machinery for ZIO, Monix Task or sophisticated test type
* A is a type of self-describing entity, such as JSON. Since 0.6.0, Iglu Client is based primarily on [circe][circe] Json type, but we're trying to leave it generic whenever possible
* ClientError is a possible unsuccessful outcome, either at resolution or validation step. Unlike ProcessingMessage from pre-0.6.0 it provides type-safe and well-structured information about the failure. This type is widely used in upcoming Snowplow bad rows

From this very short excerpt an astute Scala developer might notice that we replaced several libraries with their modern counterparts:  

* [Circe][circe] is used instead of [Json4s][json4s]. Circe provides much better performance characteristics, does not rely on runtime reflection, provides very clean idiomatic API and remains one of the most popular JSON libraries in Scala ecosystem for last couple of years
* [Cats Effect][cats-effect] is used for managing side-effects, instead of implicit effect management. One can use Iglu Client 0.6.0 without bothering about Cats Effect, but it is highly recommended in async-heavy environments
* [Cats][cats] is used instead of [Scalaz][scalaz] as FP library of choice. Cats is a transitive dependency of Circe and Cats Effect, which makes it a natural choice, since no other dependencies are shipped with Scalaz.
* [networknt json-schema-validator][networknt-validator] is used instead of FGE JSON Validator. One more change library, driving our bad rows effort.  networknt json-schema-validator is actively maintained, provides clean API and shows very impressive results in benchmarks

You can find more usage examples on dedicated [wiki page][iglu-client-docs].

<h3 id="semantic-changes">2. Semantic Changes</h3>

In a batch ETL world, we tried to reduce the load on Iglu Registries by leveraging very simple retry-and-cache algorithm, that was making some configurable amount of attempts before deciding that schema is missing or invalid and caching this failure. The only thing that potentially could reset this cached value is cacheTtl property, that would force the resolver to retry no matter whether cached value is success (in case somebody mutated schema) or failure (in case the registry had a long outage).

This approach does not work for RT-first world anymore. There's no meaningful amount of attempts that resolver needs to make before considering a schema missing or invalid. Streaming application can keep working for many weeks without restart and if during this time, one registry goes down for couple of minutes and resolver will try to resolve a schema it means that until next TTL eviction all data will be invalid. And retries won't help here because they all will happen in a short period of time. 

However, we still need to have certain retry behavior, because registries always can go offline. In a streaming world, the best practice for retries is backoff period. In 0.6.0 the Iglu resolver will attempt to refetch failed schemas with steadily growing period of time between attempts. This period grows from subsecond delays to approximately 20 minutes. What is also very important, these re-attempts will be made only for non-successful responses.

ResolutionError (subtype of `ClientError`) has two properties to reflect the history of attempts: lastAttempt a timestamp of last attempt being made and attempts showing amount of attempts taken so far.


<h3 id="new-validator">3. New Validator</h3>

As it was mentioned before, Iglu Scala Client uses new [JSON Schema validator][networknt-validator] under the hood (which hover can be replaced with any custom one). Despite this validator also targets JSON Schema spec v4, it nevertheless can have incompatibilities with our previous JSON Schema validator. As a result some instances that were considered valid by Iglu Scala Client pre-0.6.0 can now be silently invalidated.

Here's the short list of most widely used Snowplow components we're planning to release with Iglu Client 0.6.0:

* Stream Enrich 0.22.0 (Snowplow R117)
* RDB Shredder 0.15.0 (RDB Loader R31)
* BigQuery Loader 0.2.0

Please, monitor your bad rows produced by above assets.


<h3 id="acknowledgements">4. Acknowledgements</h3>

This is a huge release, overhauling the core part of Snowplow and we were developing and testing it since Fall 2018.
During this time, we received an enormous amount of contributions from outside of core Snowplow Engineering team.
Huge thanks to our Summer 2018 intern [Andrzej Sołtysik][andrzej], [Hacktoberfest][hacktoberfest] 2018 participant [Sajith Appukuttan][saj1th] and our partner from [The Globe and Mail Inc.][globe-and-mail] [Saeed Zareian][saeed].


<h3 id="roadmap">5. Roadmap and upcomming features</h3>

This release is planned to be a last one in 0.x series.
Next release will likely include a relatively small amount of user-facing improvements and will have a 1.0.0 version, marking stability of API.
Since 1.0.0 we're planning to introduce [MiMa-compatibility checks][mima] to our libraries in order to make update process more reliable.


<h3 id="help">6. Getting help</h3>

If you have any questions or run into any problems, please [raise an issue][issues] or get in touch with us through [the usual channels][talk-to-us].




**DISCARDED DRAFTS DELETE AFTER REVIEW**

Iglu Scala Client was initially written in 2014, when Scala ecosystem was drastically different. Many libraries were improved, rewritten or abandoned since then, design approaches evolved and new conventions emerged. Snowplow is among biggest open source Scala users and we want to be a part of the ecosystem, following widely accepted practices and embracing best libraries out there.

One of design approaches emerged since 2014 in Scala ecosystem is [tagless final], a way to write extensible and type-safe DSLs. In a nutshell, tagless final allows developers to abstract over certain _capabilities_, usually related to side-effects and declare functions, depending of them. Callers of such tagless-final function need to provide the capability or depend on it themselves. Different environments can have different requirements, such as ability to execute code in concurrent, distributed or purely functional fashion and end-users could provide different capability-interpreters with these requirements in mind.


[batch-deprecation-rfc]: https://discourse.snowplowanalytics.com/t/rfc-making-the-snowplow-pipeline-real-time-end-to-end-and-deprecating-support-for-batch-processing-modules/3018
[automigrations-rfc]: https://discourse.snowplowanalytics.com/t/redshift-automatic-table-migrations-rfc/2555
[bad-rows-rfc]: https://discourse.snowplowanalytics.com/t/a-new-bad-row-format/2558

[tagless-final]: https://scalac.io/tagless-final-pattern-for-scala-code/
[mima]: https://github.com/lightbend/mima
[iglu-client-docs]: https://github.com/snowplow/iglu/wiki/Scala-client

[andrzej]: https://github.com/asoltysik
[hacktoberfest]: https://hacktoberfest.digitalocean.com/
[saj1th]: https://github.com/saj1th
[saeed]: https://github.com/szareiangm
[globe-and-mail]: https://www.theglobeandmail.com/

[issues]: https://github.com/snowplow/snowplow/iglu-scala-client/issues
[talk-to-us]: https://github.com/snowplow/snowplow/wiki/Talk-to-us