# Templating Language

Consul Template parses files authored in the [Go Template][text-template]
format. If you are not familiar with the syntax, please read Go's documentation
and examples. In addition to the Go-provided template functions, Consul Template
provides the following functions:

- [API Functions](#api-functions)
  - [caLeaf](#caleaf)
  - [caRoot](#caroot)
  - [connect](#connect)
  - [datacenters](#datacenters)
  - [file](#file)
  - [key](#key)
  - [keyExists](#keyexists)
  - [keyOrDefault](#keyordefault)
  - [ls](#ls)
  - [safeLs](#safels)
  - [node](#node)
  - [nodes](#nodes)
  - [secret](#secret)
  - [secrets](#secrets)
  - [service](#service)
  - [services](#services)
  - [tree](#tree)
  - [safeTree](#safetree)
- [Scratch](#scratch)
  - [scratch.Key](#scratchkey)
  - [scratch.Get](#scratchget)
  - [scratch.Set](#scratchset)
  - [scratch.SetX](#scratchsetx)
  - [scratch.MapSet](#scratchmapset)
  - [scratch.MapSetX](#scratchmapsetx)
  - [scratch.MapValues](#scratchmapvalues)
- [Helper Functions](#helper-functions)
  - [base64Decode](#base64decode)
  - [base64Encode](#base64encode)
  - [base64URLDecode](#base64urldecode)
  - [base64URLEncode](#base64urlencode)
  - [byKey](#bykey)
  - [byTag](#bytag)
  - [byMeta](#bymeta)
  - [contains](#contains)
  - [containsAll](#containsall)
  - [containsAny](#containsany)
  - [containsNone](#containsnone)
  - [containsNotAll](#containsnotall)
  - [env](#env)
  - [executeTemplate](#executetemplate)
  - [explode](#explode)
  - [explodeMap](#explodemap)
  - [indent](#indent)
  - [in](#in)
  - [loop](#loop)
  - [join](#join)
  - [trimSpace](#trimspace)
  - [parseBool](#parsebool)
  - [parseFloat](#parsefloat)
  - [parseInt](#parseint)
  - [parseJSON](#parsejson)
  - [parseUint](#parseuint)
  - [parseYAML](#parseyaml)
  - [plugin](#plugin)
  - [regexMatch](#regexmatch)
  - [regexReplaceAll](#regexreplaceall)
  - [replaceAll](#replaceall)
  - [sha256Hex](#sha256hex)
  - [md5sum](#md5sum)
  - [split](#split)
  - [timestamp](#timestamp)
  - [toJSON](#tojson)
  - [toJSONPretty](#tojsonpretty)
  - [toLower](#tolower)
  - [toTitle](#totitle)
  - [toTOML](#totoml)
  - [toUpper](#toupper)
  - [toYAML](#toyaml)
  - [sockaddr](#sockaddr)
- [Math Functions](#math-functions)
  - [add](#add)
  - [subtract](#subtract)
  - [multiply](#multiply)
  - [divide](#divide)
  - [modulo](#modulo)
  - [minimum](#minimum)
  - [maximum](#maximum)

## API Functions

API functions interact with remote API calls, communicating with external
services like [Consul][consul] and [Vault][vault].

### `caLeaf`

Query [Consul][consul] for the leaf certificate representing a single service.

```liquid
{{ caLeaf "<NAME>" }}
```

For example:
```liquid
{{ with caLeaf "proxy" }}{{ .CertPEM }}{{ end }}
```

renders
```text
-----BEGIN CERTIFICATE-----
MIICizCCAjGgAwIBAgIBCDAKBggqhkjOPQQDAjAWMRQwEgYDVQQDEwtDb25zdWwg
...
lXcQzfKlIYeFWvcAv4cA4W258gTtqaFRDRJ2i720eQ==
-----END CERTIFICATE-----
```

The two most useful fields are `.CertPEM` and `.PrivateKeyPEM`. For a complete
list of available fields, see consul's documentation on
[LeafCert](https://godoc.org/github.com/hashicorp/consul/api#LeafCert).

### `caRoot`

Query [Consul][consul] for all [connect][connect] trusted certificate authority
(CA) root certificates.

```liquid
{{ caRoots }}
```

For example:
```liquid
{{ range caRoots }}{{ .RootCertPEM }}{{ end }}
```

renders
```text
-----BEGIN CERTIFICATE-----
MIICWDCCAf+gAwIBAgIBBzAKBggqhkjOPQQDAjAWMRQwEgYDVQQDEwtDb25zdWwg
...
bcA+Su3r8qSRppTlc6D0UOYOWc1ykQKQOK7mIg==
-----END CERTIFICATE-----

```

The most useful field is `.RootCertPEM`. For a complete list of available
fields, see consul's documentation on
[CARootList](https://godoc.org/github.com/hashicorp/consul/api#CARootList).


### `connect`

Query [Consul][consul] for [connect][connect]-capable services based on their
health.

```liquid
{{ connect "<TAG>.<NAME>@<DATACENTER>~<NEAR>|<FILTER>" }}
```

Syntax is exactly the same as for the [service](#service) function below.


```liquid
{{ range connect "web" }}
server {{ .Name }} {{ .Address }}:{{ .Port }}{{ end }}
```

renders the IP addresses of all _healthy_ nodes with a logical
[connect][connect]-capable service named "web":

```text
server web01 10.5.2.45:21000
server web02 10.2.6.61:21000
```


### `datacenters`

Query [Consul][consul] for all datacenters in its catalog.

```liquid
{{ datacenters }}
```

For example:

```liquid
{{ range datacenters }}
{{ . }}{{ end }}
```

renders

```text
dc1
dc2
```

An optional boolean can be specified which instructs Consul Template to ignore
datacenters which are inaccessible or do not have a current leader. Enabling
this option requires an O(N+1) operation and therefore is not recommended in
environments where performance is a factor.

```liquid
// Ignores datacenters which are inaccessible
{{ datacenters true }}
```

### `file`

Read and output the contents of a local file on disk. If the file cannot be
read, an error will occur. When the file changes, Consul Template will pick up
the change and re-render the template.

```liquid
{{ file "<PATH>" }}
```

For example:

```liquid
{{ file "/path/to/my/file" }}
```

renders

```text
file contents
```

This does not process nested templates. See
[`executeTemplate`](#executeTemplate) for a way to render nested templates.

### `key`

Query [Consul][consul] for the value at the given key path. If the key does not
exist, Consul Template will block rendering until the key is present. To avoid
blocking, use `keyOrDefault` or `keyExists`.

```liquid
{{ key "<PATH>@<DATACENTER>" }}
```

The `<DATACENTER>` attribute is optional; if omitted, the local datacenter is
used.

For example:

```liquid
{{ key "service/redis/maxconns" }}
```

renders

```text
15
```

### `keyExists`

Query [Consul][consul] for the value at the given key path. If the key exists,
this will return true, false otherwise. Unlike `key`, this function will not
block if the key does not exist. This is useful for controlling flow.

```liquid
{{ keyExists "<PATH>@<DATACENTER>" }}
```

The `<DATACENTER>` attribute is optional; if omitted, the local datacenter is
used.

For example:

```liquid
{{ if keyExists "app/beta_active" }}
  # ...
{{ else }}
  # ...
{{ end }}
```

### `keyOrDefault`

Query [Consul][consul] for the value at the given key path. If the key does not
exist, the default value will be used instead. Unlike `key`, this function will
not block if the key does not exist.

```liquid
{{ keyOrDefault "<PATH>@<DATACENTER>" "<DEFAULT>" }}
```

The `<DATACENTER>` attribute is optional; if omitted, the local datacenter is
used.

For example:

```liquid
{{ keyOrDefault "service/redis/maxconns" "5" }}
```

renders

```text
5
```

Note that Consul Template uses a [multi-phase
execution](#multi-phase-execution). During the first phase of evaluation, Consul
Template will have no data from Consul and thus will _always_ fall back to the
default value. Subsequent reads from Consul will pull in the real value from
Consul (if the key exists) on the next template pass. This is important because
it means that Consul Template will never "block" the rendering of a template due
to a missing key from a `keyOrDefault`. Even if the key exists, if Consul has
not yet returned data for the key, the default value will be used instead.

### `ls`

Query [Consul][consul] for all top-level kv pairs at the given key path.

```liquid
{{ ls "<PATH>@<DATACENTER>" }}
```

The `<DATACENTER>` attribute is optional; if omitted, the local datacenter is
used.

For example:

```liquid
{{ range ls "service/redis" }}
{{ .Key }}:{{ .Value }}{{ end }}
```

renders

```text
maxconns:15
minconns:5
```

### `safeLs`

Same as [ls](#ls), but refuse to render template, if the KV prefix query return blank/empty data.

This is especially useful, for rendering mission critical files, that are being populated by consul-template.

For example:

```text
/root/.ssh/authorized_keys
/etc/sysconfig/iptables
```

Using `safeLs` on empty prefixes will result in template output not being rendered at all.

To learn how `safeLs` was born see [CT-1131](https://github.com/hashicorp/consul-template/issues/1131) [C-3975](https://github.com/hashicorp/consul/issues/3975) and [CR-82](https://github.com/hashicorp/consul-replicate/issues/82).

### `node`

Query [Consul][consul] for a node in the catalog.

```liquid
{{node "<NAME>@<DATACENTER>"}}
```

The `<NAME>` attribute is optional; if omitted, the local agent node is used.

The `<DATACENTER>` attribute is optional; if omitted, the local datacenter is
used.

For example:

```liquid
{{ with node }}
{{ .Node.Address }}{{ end }}
```

renders

```text
10.5.2.6
```

To query a different node:

```liquid
{{ with node "node1@dc2" }}
{{ .Node.Address }}{{ end }}
```

renders

```text
10.4.2.6
```

To access map data such as `TaggedAddresses` or `Meta`, use
[Go's text/template][text-template] map indexing.

### `nodes`

Query [Consul][consul] for all nodes in the catalog.

```liquid
{{ nodes "@<DATACENTER>~<NEAR>" }}
```

The `<DATACENTER>` attribute is optional; if omitted, the local datacenter is
used.

The `<NEAR>` attribute is optional; if omitted, results are specified in lexical
order. If provided a node name, results are ordered by shortest round-trip time
to the provided node. If provided `_agent`, results are ordered by shortest
round-trip time to the local agent.

For example:

```liquid
{{ range nodes }}
{{ .Address }}{{ end }}
```

renders

```text
10.4.2.13
10.46.2.5
```

To query a different data center and order by shortest trip time to ourselves:

```liquid
{{ range nodes "@dc2~_agent" }}
{{ .Address }}{{ end }}
```

To access map data such as `TaggedAddresses` or `Meta`, use
[Go's text/template][text-template] map indexing.

### `secret`

Query [Vault][vault] for the secret at the given path.

```liquid
{{ secret "<PATH>" "<DATA>" }}
```

The `<DATA>` attribute is optional; if omitted, the request will be a `vault
read` (HTTP GET) request. If provided, the request will be a `vault write` (HTTP
PUT/POST) request.

For example:

```liquid
{{ with secret "secret/passwords" }}
{{ .Data.wifi }}{{ end }}
```

renders

```text
FORWARDSoneword
```

To access a versioned secret value (for the K/V version 2 backend):

```liquid
{{ with secret "secret/passwords?version=1" }}
{{ .Data.data.wifi }}{{ end }}
```

When omitting the `?version` parameter, the latest version of the secret will be
fetched. Note the nested `.Data.data` syntax when referencing the secret value.
For more information about using the K/V v2 backend, see the
[Vault Documentation](https://www.vaultproject.io/docs/secrets/kv/kv-v2.html).

When using Vault versions 0.10.0/0.10.1, the secret path will have to be prefixed
with "data", i.e. `secret/data/passwords` for the example above. This is not
necessary for Vault versions after 0.10.1, as consul-template will detect the KV
backend version being used. The version 2 KV backend did not exist prior to 0.10.0,
so these are the only affected versions.

An example using write to generate PKI certificates:

```liquid
{{ with secret "pki/issue/my-domain-dot-com" "common_name=foo.example.com" }}
{{ .Data.certificate }}{{ end }}
```

The parameters must be `key=value` pairs, and each pair must be its own argument
to the function:

Please always consider the security implications of having the contents of a
secret in plain-text on disk. If an attacker is able to get access to the file,
they will have access to plain-text secrets.

Please note that Vault does not support blocking queries. As a result, Consul
Template will not immediately reload in the event a secret is changed as it
does with Consul's key-value store. Consul Template will renew the secret with
Vault's [Renewer API](https://godoc.org/github.com/hashicorp/vault/api#Renewer).
The Renew API tries to use most of the time the secret is good, renewing at
around 90% of the lease time (as set by Vault).

Also consider enabling `error_on_missing_key` when working with templates that
will interact with Vault. By default, Consul Template uses Go's templating
language. When accessing a struct field or map key that does not exist, it
defaults to printing `<no value>`. This may not be the desired behavior,
especially when working with passwords or other data. As such, it is recommended
you set:

```hcl
template {
  error_on_missing_key = true
}
```

You can also guard against empty values using `if` or `with` blocks.

```liquid
{{ with secret "secret/foo"}}
{{ if .Data.password }}
password = "{{ .Data.password }}"
{{ end }}
{{ end }}
```

### `secrets`

Query [Vault][vault] for the list of secrets at the given path. Not all
endpoints support listing.

```liquid
{{ secrets "<PATH>" }}
```

For example:

```liquid
{{ range secrets "secret/" }}
{{ . }}{{ end }}
```

renders

```text
bar
foo
zip
```

To iterate and list over every secret in the generic secret backend in Vault:

```liquid
{{ range secrets "secret/" }}
{{ with secret (printf "secret/%s" .) }}{{ range $k, $v := .Data }}
{{ $k }}: {{ $v }}
{{ end }}{{ end }}{{ end }}
```

You should probably never do this.

Please also note that Vault does not support
blocking queries. To understand the implications, please read the note at the
end of the `secret` function.

### `service`

Query [Consul][consul] for services based on their health.

```liquid
{{ service "<TAG>.<NAME>@<DATACENTER>~<NEAR>|<FILTER>" }}
```

The `<TAG>` attribute is optional; if omitted, all nodes will be queried.

The `<DATACENTER>` attribute is optional; if omitted, the local datacenter is
used.

The `<NEAR>` attribute is optional; if omitted, results are specified in lexical
order. If provided a node name, results are ordered by shortest round-trip time
to the provided node. If provided `_agent`, results are ordered by shortest
round-trip time to the local agent.

The `<FILTER>` attribute is optional; if omitted, only health services are
returned. Providing a filter allows for client-side filtering of services.

For example:

The example above is querying Consul for healthy "web" services, in the "east-aws" data center. The tag and data center attributes are optional. To query all nodes of the "web" service (regardless of tag) for the current data center:

```liquid
{{ range service "web" }}
server {{ .Name }} {{ .Address }}:{{ .Port }}{{ end }}
```

renders the IP addresses of all _healthy_ nodes with a logical service named
"web":

```text
server web01 10.5.2.45:2492
server web02 10.2.6.61:2904
```

To access map data such as `NodeTaggedAddresses` or `NodeMeta`, use
[Go's text/template][text-template] map indexing.

By default only healthy services are returned. To list all services, pass the
"any" filter:

```liquid
{{ service "web|any" }}
```

This will return all services registered to the agent, regardless of their
status.

To filter services by a specific set of healths, specify a comma-separated list
of health statuses:

```liquid
{{ service "web|passing,warning" }}
```

This will returns services which are deemed "passing" or "warning" according to
their node and service-level checks defined in Consul. Please note that the
comma implies an "or", not an "and".

**Note:** Due to the use of dot `.` to delimit TAG, the `service` command will
not recognize service names containing dots.

**Note:** There is an architectural difference between the following:

```liquid
{{ service "web" }}
{{ service "web|passing" }}
```

The former will return all services which Consul considers "healthy" and
passing. The latter will return all services registered with the Consul agent
and perform client-side filtering. As a general rule, do not use the "passing"
argument alone if you want only healthy services - simply omit the second
argument instead.


### `services`

Query [Consul][consul] for all services in the catalog.

```liquid
{{ services "@<DATACENTER>" }}
```

The `<DATACENTER>` attribute is optional; if omitted, the local datacenter is
used.

For example:

```liquid
{{ range services }}
{{ .Name }}: {{ .Tags | join "," }}{{ end }}
```

renders

```text
node01 tag1,tag2,tag3
```

### `tree`

Query [Consul][consul] for all kv pairs at the given key path.

```liquid
{{ tree "<PATH>@<DATACENTER>" }}
```

The `<DATACENTER>` attribute is optional; if omitted, the local datacenter is
used.

For example:

```liquid
{{ range tree "service/redis" }}
{{ .Key }}:{{ .Value }}{{ end }}
```
renders

```text
minconns 2
maxconns 12
nested/config/value "value"
```

Unlike `ls`, `tree` returns **all** keys under the prefix, just like the Unix
`tree` command.

### `safeTree`

Same as [tree](#tree), but refuse to render template, if the KV prefix query return blank/empty data.

This is especially useful, for rendering mission critical files, that are being populated by consul-template.

For example:

```text
/root/.ssh/authorized_keys
/etc/sysconfig/iptables
```

Using `safeTree` on empty prefixes will result in template output not being rendered at all.

To learn how `safeTree` was born see [CT-1131](https://github.com/hashicorp/consul-template/issues/1131) [C-3975](https://github.com/hashicorp/consul/issues/3975) and [CR-82](https://github.com/hashicorp/consul-replicate/issues/82).

---

## Scratch

The scratchpad (or "scratch" for short) is available within the context of a
template to store temporary data or computations. Scratch data is not shared
between templates and is not cached between invocations.

### `scratch.Key`

Returns a boolean if data exists in the scratchpad at the named key. Even if the
data at that key is "nil", this still returns true.

```liquid
{{ scratch.Key "foo" }}
```

### `scratch.Get`

Returns the value in the scratchpad at the named key. If the data does not
exist, this will return "nil".

```liquid
{{ scratch.Get "foo" }}
```

### `scratch.Set`

Saves the given value at the given key. If data already exists at that key, it
is overwritten.

```liquid
{{ scratch.Set "foo" "bar" }}
```

### `scratch.SetX`

This behaves exactly the same as `Set`, but does not overwrite if the value
already exists.

```liquid
{{ scratch.SetX "foo" "bar" }}
```

### `scratch.MapSet`

Saves a value in a named key in the map. If data already exists at that key, it
is overwritten.

```liquid
{{ scratch.MapSet "vars" "foo" "bar" }}
```

### `scratch.MapSetX`

This behaves exactly the same as `MapSet`, but does not overwrite if the value
already exists.

```liquid
{{ scratch.MapSetX "vars" "foo" "bar" }}
```

### `scratch.MapValues`

Returns a sorted list (by key) of all values in the named map.

```liquid
{{ scratch.MapValues "vars" }}
```

---

## Helper Functions

Unlike API functions, helper functions do not query remote services. These
functions are useful for parsing data, formatting data, performing math, etc.

### `base64Decode`

Accepts a base64-encoded string and returns the decoded result, or an error if
the given string is not a valid base64 string.

```liquid
{{ base64Decode "aGVsbG8=" }}
```

renders

```text
hello
```

### `base64Encode`

Accepts a string and returns a base64-encoded string.

```liquid
{{ base64Encode "hello" }}
```

renders

```text
aGVsbG8=
```

### `base64URLDecode`

Accepts a base64-encoded URL-safe string and returns the decoded result, or an
error if the given string is not a valid base64 URL-safe string.

```liquid
{{ base64URLDecode "aGVsbG8=" }}
```

renders

```text
hello
```

### `base64URLEncode`

Accepts a string and returns a base-64 encoded URL-safe string.

```liquid
{{ base64Encode "hello" }}
```

renders

```text
aGVsbG8=
```

### `byKey`

Accepts a list of pairs returned from a [`tree`](#tree) call and creates a map that groups pairs by their top-level directory.

For example:

```text
groups/elasticsearch/es1
groups/elasticsearch/es2
groups/elasticsearch/es3
services/elasticsearch/check_elasticsearch
services/elasticsearch/check_indexes
```

with the following template

```liquid
{{ range $key, $pairs := tree "groups" | byKey }}{{ $key }}:
{{ range $pair := $pairs }}  {{ .Key }}={{ .Value }}
{{ end }}{{ end }}
```

renders

```text
elasticsearch:
  es1=1
  es2=1
  es3=1
```

Note that the top-most key is stripped from the Key value. Keys that have no
prefix after stripping are removed from the list.

The resulting pairs are keyed as a map, so it is possible to look up a single
value by key:

```liquid
{{ $weights := tree "weights" }}
{{ range service "release.web" }}
  {{ $weight := or (index $weights .Node) 100 }}
  server {{ .Node }} {{ .Address }}:{{ .Port }} weight {{ $weight }}{{ end }}
```

### `byTag`

Takes the list of services returned by the [`service`](#service) or
[`services`](#services) function and creates a map that groups services by tag.

```liquid
{{ range $tag, $services := service "web" | byTag }}{{ $tag }}
{{ range $services }} server {{ .Name }} {{ .Address }}:{{ .Port }}
{{ end }}{{ end }}
```

### `byMeta`

Takes a list of services returned by [`service`](#service) and returns a map
that groups services by ServiceMeta values. Multiple service meta keys can be
passed as a comma separated string. `|int` can be added to a meta key to
convert numbers from service meta values to padded numbers in `printf "%05d" %
value` format (useful for sorting as Go Template sorts maps by keys).

**Example**:

If we have the following services registered in Consul:

```json
{
  "Services": [
     {
       "ID": "redis-dev-1",
       "Name": "redis",
       "ServiceMeta": {
         "environment": "dev",
         "shard_number": "1"
       },
       ...
     },
     {
       "ID": "redis-prod-1",
       "Name": "redis",
       "ServiceMeta": {
         "environment": "prod",
         "shard_number": "1"
       },
       ...
     },
     {
       "ID": "redis-prod-2",
       "Name": "redis",
       "ServiceMeta": {
         "environment": "prod",
         "shard_number": "2",
       },
       ...
     }
   ]
}
```

```liquid
{{ service "redis|any" | byMeta "environment,shard_number|int" | toJSON }}
```

The code above will produce a map of services grouped by meta:

```json
{
  "dev_00001": [
    {
      "ID": "redis-dev-1",
      ...
    }
  ],
  "prod_00001": [
    {
      "ID": "redis-prod-1",
      ...
    }
  ],
  "prod_00002": [
    {
      "ID": "redis-prod-2",
      ...
    }
  ]
}
```

### `contains`

Determines if a needle is within an iterable element.

```liquid
{{ if .Tags | contains "production" }}
# ...
{{ end }}
```

### `containsAll`

Returns `true` if all needles are within an iterable element, or `false`
otherwise. Returns `true` if the list of needles is empty.

```liquid
{{ if containsAll $requiredTags .Tags }}
# ...
{{ end }}
```

### `containsAny`

Returns `true` if any needle is within an iterable element, or `false`
otherwise. Returns `false` if the list of needles is empty.

```liquid
{{ if containsAny $acceptableTags .Tags }}
# ...
{{ end }}
```

### `containsNone`

Returns `true` if no needles are within an iterable element, or `false`
otherwise. Returns `true` if the list of needles is empty.

```liquid
{{ if containsNone $forbiddenTags .Tags }}
# ...
{{ end }}
```

### `containsNotAll`

Returns `true` if some needle is not within an iterable element, or `false`
otherwise. Returns `false` if the list of needles is empty.

```liquid
{{ if containsNotAll $excludingTags .Tags }}
# ...
{{ end }}
```

### `env`

Reads the given environment variable accessible to the current process.

```liquid
{{ env "CLUSTER_ID" }}
```

This function can be chained to manipulate the output:

```liquid
{{ env "CLUSTER_ID" | toLower }}
```

Reads the given environment variable and if it does not exist or is blank use a default value, ex `12345`.

```liquid
{{ or (env "CLUSTER_ID") "12345" }}
```

### `executeTemplate`

Executes and returns a defined template.

```liquid
{{ define "custom" }}my custom template{{ end }}

This is my other template:
{{ executeTemplate "custom" }}

And I can call it multiple times:
{{ executeTemplate "custom" }}

Even with a new context:
{{ executeTemplate "custom" 42 }}

Or save it to a variable:
{{ $var := executeTemplate "custom" }}
```

### `explode`

Takes the result from a `tree` or `ls` call and converts it into a deeply-nested
map for parsing/traversing.

```liquid
{{ tree "config" | explode }}
```

Note: You will lose any metadata about the key-pair after it has been exploded.
You can also access deeply nested values:

```liquid
{{ with tree "config" | explode }}
{{ .a.b.c }}{{ end }}
```

You will need to have a reasonable format about your data in Consul. Please see
[Go's text/template package][text-template] for more information.


### `explodeMap`

Takes the value of a map and converts it into a deeply-nested map for parsing/traversing,
using the same logic as `explode`.

```liquid
{{ scratch.MapSet "example", "foo/bar", "a" }}
{{ scratch.MapSet "example", "foo/baz", "b" }}
{{ scratch.Get "example" | explodeMap | toYAML }}
```

### `indent`

Indents a block of text by prefixing N number of spaces per line.

```liquid
{{ tree "foo" | explode | toYAML | indent 4 }}
```

### `in`

Determines if a needle is within an iterable element.

```liquid
{{ if in .Tags "production" }}
# ...
{{ end }}
```

### `loop`

Accepts varying parameters and differs its behavior based on those parameters.

If `loop` is given one integer, it will return a goroutine that begins at zero
and loops up to but not including the given integer:

```liquid
{{ range loop 5 }}
# Comment{{end}}
```

If given two integers, this function will return a goroutine that begins at
the first integer and loops up to but not including the second integer:

```liquid
{{ range $i := loop 5 8 }}
stanza-{{ $i }}{{ end }}
```

which would render:

```text
stanza-5
stanza-6
stanza-7
```

Note: It is not possible to get the index and the element since the function
returns a goroutine, not a slice. In other words, the following is **not
valid**:

```liquid
# Will NOT work!
{{ range $i, $e := loop 5 8 }}
# ...{{ end }}
```

### `join`

Takes the given list of strings as a pipe and joins them on the provided string:

```liquid
{{ $items | join "," }}
```

### `trimSpace`

Takes the provided input and trims all whitespace, tabs and newlines:

```liquid
{{ file "/etc/ec2_version" | trimSpace }}
```

### `parseBool`

Takes the given string and parses it as a boolean:

```liquid
{{ "true" | parseBool }}
```

This can be combined with a key and a conditional check, for example:

```liquid
{{ if key "feature/enabled" | parseBool }}{{ end }}
```

### `parseFloat`

Takes the given string and parses it as a base-10 float64:

```liquid
{{ "1.2" | parseFloat }}
```

### `parseInt`

Takes the given string and parses it as a base-10 int64:

```liquid
{{ "1" | parseInt }}
```

This can be combined with other helpers, for example:

```liquid
{{ range $i := loop key "config/pool_size" | parseInt }}
# ...{{ end }}
```

### `parseJSON`

Takes the given input (usually the value from a key) and parses the result as
JSON:

```liquid
{{ with $d := key "user/info" | parseJSON }}{{ $d.name }}{{ end }}
```

Note: Consul Template evaluates the template multiple times, and on the first
evaluation the value of the key will be empty (because no data has been loaded
yet). This means that templates must guard against empty responses.

### `parseUint`

Takes the given string and parses it as a base-10 int64:

```liquid
{{ "1" | parseUint }}
```

### `parseYAML`

Takes the given input (usually the value from a key) and parses the result as
YAML:

```liquid
{{ with $d := key "user/info" | parseYAML }}{{ $d.name }}{{ end }}
```

Note: The same caveats that apply to `parseJSON` apply to `parseYAML`.

### `plugin`

Takes the name of a plugin and optional payload and executes a Consul Template
plugin.

```liquid
{{ plugin "my-plugin" }}
```

The plugin can take an arbitrary number of string arguments, and can be the
target of a pipeline that produces strings as well. This is most commonly
combined with a JSON filter for customization:

```liquid
{{ tree "foo" | explode | toJSON | plugin "my-plugin" }}
```

Please see the [plugins](#plugins) section for more information about plugins.

### `regexMatch`

Takes the argument as a regular expression and will return `true` if it matches
on the given string, or `false` otherwise.

```liquid
{{ if "foo.bar" | regexMatch "foo([.a-z]+)" }}
# ...
{{ else }}
# ...
{{ end }}
```

### `regexReplaceAll`

Takes the argument as a regular expression and replaces all occurrences of the
regex with the given string. As in go, you can use variables like $1 to refer to
subexpressions in the replacement string.

```liquid
{{ "foo.bar" | regexReplaceAll "foo([.a-z]+)" "$1" }}
```

### `replaceAll`

Takes the argument as a string and replaces all occurrences of the given string
with the given string.

```liquid
{{ "foo.bar" | replaceAll "." "_" }}
```

This function can be chained with other functions as well:

```liquid
{{ service "web" }}{{ .Name | replaceAll ":" "_" }}{{ end }}
```

### `sha256Hex`

Takes the argument as a string and compute the sha256_hex value

```liquid
{{ "bladibla" | sha256Hex }}
```

### `md5sum`

Takes a string input as an argument, and returns the hex-encoded md5 hash of the input.

```liquid
{{ "myString" | md5 }}
```

### `split`

Splits the given string on the provided separator:

```liquid
{{ "foo\nbar\n" | split "\n" }}
```

This can be combined with chained and piped with other functions:

```liquid
{{ key "foo" | toUpper | split "\n" | join "," }}
```

### `timestamp`

Returns the current timestamp as a string (UTC). If no arguments are given, the
result is the current RFC3339 timestamp:

```liquid
{{ timestamp }} // e.g. 1970-01-01T00:00:00Z
```

If the optional parameter is given, it is used to format the timestamp. The
magic reference date **Mon Jan 2 15:04:05 -0700 MST 2006** can be used to format
the date as required:

```liquid
{{ timestamp "2006-01-02" }} // e.g. 1970-01-01
```

See [Go's `time.Format`](http://golang.org/pkg/time/#Time.Format) for more
information.

As a special case, if the optional parameter is `"unix"`, the unix timestamp in
seconds is returned as a string.

```liquid
{{ timestamp "unix" }} // e.g. 0
```

### `toJSON`

Takes the result from a `tree` or `ls` call and converts it into a JSON object.

```liquid
{{ tree "config" | explode | toJSON }}
```

renders

```javascript
{"admin":{"port":"1234"},"maxconns":"5","minconns":"2"}
```

Note: Consul stores all KV data as strings. Thus true is "true", 1 is "1", etc.

### `toJSONPretty`

Takes the result from a `tree` or `ls` call and converts it into a
pretty-printed JSON object, indented by two spaces.

```liquid
{{ tree "config" | explode | toJSONPretty }}
```

renders

```javascript
{
  "admin": {
    "port": "1234"
  },
  "maxconns": "5",
  "minconns": "2",
}
```

Note: Consul stores all KV data as strings. Thus true is "true", 1 is "1", etc.

### `toLower`

Takes the argument as a string and converts it to lowercase.

```liquid
{{ key "user/name" | toLower }}
```

See [Go's `strings.ToLower`](http://golang.org/pkg/strings/#ToLower) for more
information.

### `toTitle`

Takes the argument as a string and converts it to titlecase.

```liquid
{{ key "user/name" | toTitle }}
```

See [Go's `strings.Title`](http://golang.org/pkg/strings/#Title) for more
information.

### `toTOML`

Takes the result from a `tree` or `ls` call and converts it into a TOML object.

```liquid
{{ tree "config" | explode | toTOML }}
```

renders

```toml
maxconns = "5"
minconns = "2"

[admin]
  port = "1134"
```

Note: Consul stores all KV data as strings. Thus true is "true", 1 is "1", etc.

### `toUpper`

Takes the argument as a string and converts it to uppercase.

```liquid
{{ key "user/name" | toUpper }}
```

See [Go's `strings.ToUpper`](http://golang.org/pkg/strings/#ToUpper) for more
information.

### `toYAML`

Takes the result from a `tree` or `ls` call and converts it into a
pretty-printed YAML object, indented by two spaces.

```liquid
{{ tree "config" | explode | toYAML }}
```

renders

```yaml
admin:
  port: "1234"
maxconns: "5"
minconns: "2"
```

Note: Consul stores all KV data as strings. Thus true is "true", 1 is "1", etc.

### `sockaddr`

Takes a quote-escaped template string as an argument and passes it on to
[hashicorp/go-sockaddr](https://github.com/hashicorp/go-sockaddr) templating engine.

```liquid
{{ sockaddr "GetPrivateIP" }}
```

See [hashicorp/go-sockaddr documentation](https://godoc.org/github.com/hashicorp/go-sockaddr)
for more information.

---

## Math Functions

The following functions are available on floats and integer values.

### `add`

Returns the sum of the two values.

```liquid
{{ add 1 2 }} // 3
```

This can also be used with a pipe function.

```liquid
{{ 1 | add 2 }} // 3
```

### `subtract`

Returns the difference of the second value from the first.

```liquid
{{ subtract 2 5 }} // 3
```

This can also be used with a pipe function.

```liquid
{{ 5 | subtract 2 }} // 3
```

Please take careful note of the order of arguments.

### `multiply`

Returns the product of the two values.

```liquid
{{ multiply 2 2 }} // 4
```

This can also be used with a pipe function.

```liquid
{{ 2 | multiply 2 }} // 4
```

### `divide`

Returns the division of the second value from the first.

```liquid
{{ divide 2 10 }} // 5
```

This can also be used with a pipe function.

```liquid
{{ 10 | divide 2 }} // 5
```

Please take careful note of the order or arguments.

### `modulo`

Returns the modulo of the second value from the first.

```liquid
{{ modulo 2 5 }} // 1
```

This can also be used with a pipe function.

```liquid
{{ 5 | modulo 2 }} // 1
```

Please take careful note of the order of arguments.

### `minimum`

Returns the minimum of the two values.

```liquid
{{ minimum 2 5 }} // 2
```

This can also be used with a pipe function.

```liquid
{{ 5 | minimum 2 }} // 2
```

### `maximum`

Returns the maximum of the two values.

```liquid
{{ maximum 2 5 }} // 2
```

This can also be used with a pipe function.

```liquid
{{ 5 | maximum 2 }} // 2
```
