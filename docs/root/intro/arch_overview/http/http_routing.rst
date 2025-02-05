.. _arch_overview_http_routing:

HTTP routing
============

Envoy includes an HTTP :ref:`router filter <config_http_filters_router>` which can be installed to
perform advanced routing tasks. This is useful both for handling edge traffic (traditional reverse
proxy request handling) as well as for building a service to service Envoy mesh (typically via
routing on the host/authority HTTP header to reach a particular upstream service cluster). Envoy
also has the ability to be configured as forward proxy. In the forward proxy configuration, mesh
clients can participate by appropriately configuring their http proxy to be an Envoy. At a high
level the router takes an incoming HTTP request, matches it to an upstream cluster, acquires a
:ref:`connection pool <arch_overview_conn_pool>` to a host in the upstream cluster, and forwards the
request. The router filter supports the following features:

* Virtual hosts that map domains/authorities to a set of routing rules.
* Prefix and exact path matching rules (both :ref:`case sensitive <envoy_v3_api_field_config.route.v3.RouteMatch.case_sensitive>` and case insensitive).
* :ref:`Regex path matching <envoy_v3_api_field_config.route.v3.RouteMatch.safe_regex>` rules.
* :ref:`TLS redirection <envoy_v3_api_field_config.route.v3.VirtualHost.require_tls>` at the virtual host
  level.
* :ref:`Path <envoy_v3_api_field_config.route.v3.RedirectAction.path_redirect>`/:ref:`host
  <envoy_v3_api_field_config.route.v3.RedirectAction.host_redirect>` redirection at the route level.
* :ref:`Direct (non-proxied) HTTP responses <arch_overview_http_routing_direct_response>`
  at the route level.
* :ref:`Explicit host rewriting <envoy_v3_api_field_extensions.filters.http.aws_request_signing.v3.AwsRequestSigning.host_rewrite>`.
* :ref:`Automatic host rewriting <envoy_v3_api_field_config.route.v3.RouteAction.auto_host_rewrite>` based on
  the DNS name of the selected upstream host.
* :ref:`Prefix rewriting <envoy_v3_api_field_config.route.v3.RedirectAction.prefix_rewrite>`.
* :ref:`Path rewriting using a regular expression and capture groups <envoy_v3_api_field_config.route.v3.RouteAction.regex_rewrite>`.
* :ref:`Request retries <arch_overview_http_routing_retry>` specified either via HTTP header or via
  route configuration.
* Request timeout specified either via :ref:`HTTP
  header <config_http_filters_router_headers_consumed>` or via :ref:`route configuration
  <envoy_v3_api_field_config.route.v3.RouteAction.timeout>`.
* :ref:`Request hedging <arch_overview_http_routing_hedging>` for retries in response to a request (per try) timeout.
* Traffic shifting from one upstream cluster to another via :ref:`runtime values
  <envoy_v3_api_field_config.route.v3.RouteMatch.runtime_fraction>` (see :ref:`traffic shifting/splitting
  <config_http_conn_man_route_table_traffic_splitting>`).
* Traffic splitting across multiple upstream clusters using :ref:`weight/percentage-based routing
  <envoy_v3_api_field_config.route.v3.RouteAction.weighted_clusters>` (see :ref:`traffic shifting/splitting
  <config_http_conn_man_route_table_traffic_splitting_split>`).
* Arbitrary header matching :ref:`routing rules <envoy_v3_api_msg_config.route.v3.HeaderMatcher>`.
* Virtual cluster specifications. A virtual cluster is specified at the virtual host level and is
  used by Envoy to generate additional statistics on top of the standard cluster level ones. Virtual
  clusters can use regex matching.
* :ref:`Priority <arch_overview_http_routing_priority>` based routing.
* :ref:`Hash policy <envoy_v3_api_field_config.route.v3.RouteAction.hash_policy>` based routing.
* :ref:`Absolute urls <envoy_v3_api_field_extensions.filters.network.http_connection_manager.v3.HttpConnectionManager.http_protocol_options>` are supported for non-tls forward proxies.

.. _arch_overview_http_routing_route_scope:

Route Scope
-----------

Scoped routing enables Envoy to put constraints on search space of domains and route rules.
A :ref:`Route Scope <envoy_v3_api_msg_config.route.v3.scopedrouteconfiguration>` associates a key with a :ref:`route table <arch_overview_http_routing_route_table>`.
For each request, a scope key is computed dynamically by the HTTP connection manager to pick the :ref:`route table <envoy_v3_api_msg_config.route.v3.routeconfiguration>`.
RouteConfiguration associated with scope can be loaded on demand with :ref:`v3 API reference <envoy_v3_api_msg_extensions.filters.http.on_demand.v3.OnDemand>` configured and on demand filed in protobuf set to true.

The Scoped RDS (SRDS) API contains a set of :ref:`Scopes <envoy_v3_api_msg_config.route.v3.ScopedRouteConfiguration>` resources, each defining independent routing configuration,
along with a :ref:`ScopeKeyBuilder <envoy_v3_api_msg_extensions.filters.network.http_connection_manager.v3.ScopedRoutes.ScopeKeyBuilder>`
defining the key construction algorithm used by Envoy to look up the scope corresponding to each request.

For example, for the following scoped route configuration, Envoy will look into the "addr" header value, split the header value by ";" first, and use the first value for key 'x-foo-key' as the scope key.
If the "addr" header value is "foo=1;x-foo-key=127.0.0.1;x-bar-key=1.1.1.1", then "127.0.0.1" will be computed as the scope key to look up for corresponding route configuration.

.. code-block:: yaml

  name: scope_by_addr
  fragments:
    - header_value_extractor:
        name: Addr
        element_separator: ;
        element:
          key: x-foo-key
          separator: =

.. _arch_overview_http_routing_route_table:

For a key to match a :ref:`ScopedRouteConfiguration<envoy_v3_api_msg_config.route.v3.ScopedRouteConfiguration>`, the number of fragments in the computed key has to match that of
the :ref:`ScopedRouteConfiguration<envoy_v3_api_msg_config.route.v3.ScopedRouteConfiguration>`.
Then fragments are matched in order. A missing fragment(treated as NULL) in the built key makes the request unable to match any scope,
i.e. no route entry can be found for the request.

Route table
-----------

The :ref:`configuration <config_http_conn_man>` for the HTTP connection manager owns the :ref:`route
table <envoy_v3_api_msg_config.route.v3.RouteConfiguration>` that is used by all configured HTTP filters. Although the
router filter is the primary consumer of the route table, other filters also have access in case
they want to make decisions based on the ultimate destination of the request. For example, the built
in rate limit filter consults the route table to determine whether the global rate limit service
should be called based on the route. The connection manager makes sure that all calls to acquire a
route are stable for a particular request, even if the decision involves randomness (e.g. in the
case of a runtime configuration route rule).

.. _arch_overview_http_routing_retry:

Retry semantics
---------------

Envoy allows retries to be configured both in the :ref:`route configuration
<envoy_v3_api_field_config.route.v3.RouteAction.retry_policy>` as well as for specific requests via :ref:`request
headers <config_http_filters_router_headers_consumed>`. The following configurations are possible:

* **Maximum number of retries**: Envoy will continue to retry any number of times. The intervals between
  retries are decided either by an exponential backoff algorithm (the default), or based on feedback
  from the upstream server via headers (if present). Additionally, *all retries are contained within the
  overall request timeout*. This avoids long request times due to a large number of retries.
* **Retry conditions**: Envoy can retry on different types of conditions depending on application
  requirements. For example, network failure, all 5xx response codes, idempotent 4xx response codes,
  etc.
* **Retry budgets**: Envoy can limit the proportion of active requests via :ref:`retry budgets <envoy_v3_api_field_config.cluster.v3.circuitbreakers.thresholds.retry_budget>` that can be retries to
  prevent their contribution to large increases in traffic volume.
* **Host selection retry plugins**: Envoy can be configured to apply additional logic to the host
  selection logic when selecting hosts for retries. Specifying a
  :ref:`retry host predicate <envoy_v3_api_field_config.route.v3.RetryPolicy.retry_host_predicate>`
  allows for reattempting host selection when certain hosts are selected (e.g. when an already
  attempted host is selected), while a
  :ref:`retry priority <envoy_v3_api_field_config.route.v3.RetryPolicy.retry_priority>` can be
  configured to adjust the priority load used when selecting a priority for retries.

Note that Envoy retries requests when :ref:`x-envoy-overloaded
<config_http_filters_router_x-envoy-overloaded_set>` is present. It is recommended to either configure
:ref:`retry budgets (preferred) <envoy_v3_api_field_config.cluster.v3.circuitbreakers.thresholds.retry_budget>` or set
:ref:`maximum active retries circuit breaker <arch_overview_circuit_break>` to an appropriate value to avoid retry storms.

.. _arch_overview_http_routing_hedging:

Request Hedging
---------------

Envoy supports request hedging which can be enabled by specifying a :ref:`hedge
policy <envoy_v3_api_msg_config.route.v3.HedgePolicy>`. This means that Envoy will race
multiple simultaneous upstream requests and return the response associated with
the first acceptable response headers to the downstream. The retry policy is
used to determine whether a response should be returned or whether more
responses should be awaited.

Currently hedging can only be performed in response to a request timeout. This
means that a retry request will be issued without canceling the initial
timed-out request and a late response will be awaited. The first "good"
response according to retry policy will be returned downstream.

The implementation ensures that the same upstream request is not retried twice.
This might otherwise occur if a request times out and then results in a 5xx
response, creating two retriable events.

.. _arch_overview_http_routing_priority:

Priority routing
----------------

Envoy supports priority routing at the :ref:`route <envoy_v3_api_msg_config.route.v3.Route>` level.
The current priority implementation uses different :ref:`connection pool <arch_overview_conn_pool>`
and :ref:`circuit breaking <config_cluster_manager_cluster_circuit_breakers>` settings for each
priority level. This means that even for HTTP/2 requests, two physical connections will be used to
an upstream host. In the future Envoy will likely support true HTTP/2 priority over a single
connection.

The currently supported priorities are *default* and *high*.

.. _arch_overview_http_routing_direct_response:

Direct responses
----------------

Envoy supports the sending of "direct" responses. These are preconfigured HTTP responses
that do not require proxying to an upstream server.

There are two ways to specify a direct response in a Route:

* Set the :ref:`direct_response <envoy_v3_api_field_config.route.v3.Route.direct_response>` field.
  This works for all HTTP response statuses.
* Set the :ref:`redirect <envoy_v3_api_field_config.route.v3.Route.redirect>` field. This works for
  redirect response statuses only, but it simplifies the setting of the *Location* header.

A direct response has an HTTP status code and an optional body. The Route configuration
can specify the response body inline or specify the pathname of a file containing the
body. If the Route configuration specifies a file pathname, Envoy will read the file
upon configuration load and cache the contents.

.. attention::

   If a response body is specified, by default it is limited to 4KB in size, regardless of
   whether it is provided inline or in a file. Envoy currently holds the entirety of the
   body in memory, so the 4KB default is intended to keep the proxy's memory footprint
   from growing too large. However, if required, this limit can be changed through setting
   the :ref:`max_direct_response_body_size_bytes
   <envoy_v3_api_field_config.route.v3.RouteConfiguration.max_direct_response_body_size_bytes>`
   field.

If **response_headers_to_add** has been set for the Route or the enclosing Virtual Host,
Envoy will include the specified headers in the direct HTTP response.

Routing Via Generic Matching
----------------------------

Envoy recently added support for utilizing a :ref:`generic match tree <arch_overview_matching_api>` to
specify the route table. This is a more expressive matching engine than the original one, allowing
for sublinear matching on arbitrary headers (unlike the original matching engine which could only
do this for :authority in some cases).

To use the generic matching tree, specify a matcher on a virtual host with a RouteAction action:

.. code-block:: yaml

  matcher:
    "@type": type.googleapis.com/xds.type.matcher.v3.Matcher
    matcher_tree:
      input:
        name: request-headers
        typed_config:
          "@type": type.googleapis.com/envoy.type.matcher.v3.HttpRequestHeaderMatchInput
          header_name: :path
      exact_match_map:
        map:
          "/new_endpoint/foo":
            action:
              name: route
              typed_config:
                "@type": type.googleapis.com/envoy.config.route.v3.Route
                match:
                  prefix: /
                route:
                  cluster: cluster_foo
                request_headers_to_add:
                - header:
                    key: x-route-header
                    value: new-value
          "/new_endpoint/bar":
            action:
              name: route
              typed_config:
                "@type": type.googleapis.com/envoy.config.route.v3.Route
                match:
                  prefix: /
                route:
                  cluster: cluster_bar
                request_headers_to_add:
                - header:
                    key: x-route-header
                    value: new-value

This allows resolving the same Route proto message used for the `routes`-based routing using the additional
matching flexibility provided by the generic matching framework.

Note that the resulting Route also specifies a match criteria. This must be satisfied in addition to resolving
the route in order to achieve a route match. When path rewrites are used, the matched path will only depend on
the match criteria of the resolved Route. Path matching done during the match tree traversal does not contribute
to path rewrites.

The only inputs supported are request headers (via `envoy.type.matcher.v3.HttpRequestHeaderMatchInput`). See
the docs for the :ref:`matching API <arch_overview_matching_api>` for more information about the API as a whole.
