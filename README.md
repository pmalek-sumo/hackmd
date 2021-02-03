# Adding telegraf agent to opentelemetry-collector-contrib

## The problem

While working on adding telegraf receiver to
[opentelemetry-collector-contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib)
one might notice that the dependencies pulled in by telegraf and
opentelemetry-collector-contrib have a couple of conflicts.

Adding the following dependecy in opentelemetry-collector-contrib

```
require github.com/influxdata/telegraf v1.17.1
```

results in the following build errors:

```
$ make otelcontribcol-unstable
GO111MODULE=on CGO_ENABLED=0 go build -o ./bin/otelcontribcol_unstable_darwin_amd64 \
                -ldflags "-X github.com/open-telemetry/opentelemetry-collector-contrib/internal/version.GitHash=63b2f339 -X github.com/open-telemetry/opentelemetry-collector-contrib/internal/version.Version=v0.19.0 -X go.opentelemetry.io/collector/internal/version.BuildType=release" -tags enable_unstable ./cmd/otelcontribcol
go: finding module for package github.com/prometheus/prometheus/discovery/install
go: finding module for package github.com/Azure/azure-sdk-for-go/arm/compute
go: finding module for package github.com/Azure/azure-sdk-for-go/arm/network
../../../.gvm/pkgsets/go1.15.7/global/pkg/mod/github.com/prometheus/prometheus@v2.5.0+incompatible/discovery/azure/azure.go:24:2: module github.com/Azure/azure-sdk-for-go@latest found (v51.0.0+incompatible), but does not contain package github.com/Azure/azure-sdk-for-go/arm/compute
../../../.gvm/pkgsets/go1.15.7/global/pkg/mod/github.com/prometheus/prometheus@v2.5.0+incompatible/discovery/azure/azure.go:25:2: module github.com/Azure/azure-sdk-for-go@latest found (v51.0.0+incompatible), but does not contain package github.com/Azure/azure-sdk-for-go/arm/network
../../../.gvm/pkgsets/go1.15.7/global/pkg/mod/github.com/prometheus/prometheus@v2.5.0+incompatible/discovery/consul/consul.go:27:2: ambiguous import: found package github.com/hashicorp/consul/api in multiple modules:
        github.com/hashicorp/consul v1.2.1 (/Users/pmalek/.gvm/pkgsets/go1.15.7/global/pkg/mod/github.com/hashicorp/consul@v1.2.1/api)
        github.com/hashicorp/consul/api v1.7.0 (/Users/pmalek/.gvm/pkgsets/go1.15.7/global/pkg/mod/github.com/hashicorp/consul/api@v1.7.0)
../../../.gvm/pkgsets/go1.15.7/global/pkg/mod/go.opentelemetry.io/collector@v0.19.0/receiver/prometheusreceiver/factory.go:22:2: module github.com/prometheus/prometheus@latest found (v2.5.0+incompatible), but does not contain package github.com/prometheus/prometheus/discovery/install
make: *** [otelcontribcol-unstable] Error 1
```

---

## Import issues

### Prometheus

Because of introduction of Go modules in Prometheus `v2.6`
[commit][1] and not following Go semver convention (every module version past
`v1` should have a `v<num>` suffix) and [no intention on changing this][2], requiring Prometheus package results in adding `github.com/prometheus/prometheus@v2.5.0` as
a dependency since that was the last version without support for Go modules
(hence no Go module requirements mentioned above).

This can be overcome with requiring a [particular SHA of a release commit][3] in telegraf
like so:

```
go get github.com/prometheus/prometheus@e83ef207b6c2398919b69cd87d2693cfc2fb4127
```

This particular version is [a v2.21.0 release commit][4] which doesn't conflict with
`00f16d1ac3a4` version required by otc-contrib - pointing to [`v2.22.1` commit][5].

[1]: https://github.com/prometheus/prometheus/commit/a516bc2160b86c652d7ebb7d2df0fc27ca328f8b
[2]: https://github.com/prometheus/prometheus/issues/8417#issuecomment-769042914
[3]: https://github.com/prometheus/prometheus/issues/7991#issuecomment-701298893
[4]: https://github.com/prometheus/prometheus/commit/e83ef207b6c2398919b69cd87d2693cfc2fb4127
[5]: https://github.com/prometheus/prometheus/commit/00f16d1ac3a4c94561e5133b821d8e4d9ef78ec2

## Consul

When the above is resolved (to an extent) in telegraf repository we get another problem
with `github.com/hashicorp/consul/api`:

```
$ make telegraf
go build -mod=mod -ldflags " -X main.commit=8907e61a -X main.branch=HEAD -X main.goos=darwin -X main.goarch=amd64 -X main.version=1.17.1" ./cmd/telegraf
plugins/inputs/consul/consul.go:7:2: ambiguous import: found package github.com/hashicorp/consul/api in multiple modules:
        github.com/hashicorp/consul v1.2.1 (/Users/pmalek/.gvm/pkgsets/go1.15.7/global/pkg/mod/github.com/hashicorp/consul@v1.2.1/api)
        github.com/hashicorp/consul/api v1.6.0 (/Users/pmalek/.gvm/pkgsets/go1.15.7/global/pkg/mod/github.com/hashicorp/consul/api@v1.6.0)
```

This is the result of introducing a separate Go modules in
[`github.com/hashicorp/consul/api`][1] which before was a package inside
`github.com/hashicorp/consul` module but now it's available under a different
import (module) path.

To resolve this we can request a higher version that contains 
`github.com/hashicorp/consul/api` and doesn break telegraf compilation e.g. `v1.6.0`

```
go get github.com/hashicorp/consul@v1.6.0
```



[1]: https://github.com/hashicorp/consul/tree/master/api