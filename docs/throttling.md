
## The mobile network throttling preset

This is the standard recommendation for mobile throttling:

- Latency: 150ms
- Throughput: 1.6Mbps down / 750 Kbps up.
- Packet loss: none.

These exact figures are used as [Lighthouse's throttling default](https://github.com/GoogleChrome/lighthouse/blob/8f500e00243e07ef0a80b39334bedcc8ddc8d3d0/lighthouse-core/config/constants.js#L19-L26) and represent roughly the bottom 25% of 4G connections and top 25% of 3G connections. They are identical to the [WebPageTest "Mobile 3G - Fast" preset](https://github.com/WPO-Foundation/webpagetest/blob/master/www/settings/connectivity.ini.sample) and, due to a lower latency, slightly faster for some pages than the [WebPageTest "4G" preset](https://github.com/WPO-Foundation/webpagetest/blob/master/www/settings/connectivity.ini.sample).

## Throttling basics

1. Simulated throttling, which Lighthouse uses by default, is computed throttling _after a trace has been recorded_ which makes it very fast and deterministic. However, due to the imperfect nature of predicting alternate execution paths, there is inherent inaccuracy that is summarized in this doc: [Lighthouse Metric Variability and Accuracy](https://docs.google.com/document/d/1BqtL-nG53rxWOI5RO0pItSRPowZVnYJ_gBEQCJ5EeUE/edit). The TLDR: while it's roughly as accurate or better than DevTools throttling for most sites, it suffers from edge cases and a deep investigation to performance should use _Packet-level_ throttling tools.
1. The DevTools network throttling, which Lighthouse uses for `throttlingMethod='devtools'`, is implemented within Chrome at the _request-level_. As a result, it has some downsides that are now summarized in this doc: [Network Throttling & Chrome - status](https://docs.google.com/document/d/1TwWLaLAfnBfbk5_ZzpGXegPapCIfyzT4MWuZgspKUAQ/edit). The TLDR: while it's a [decent approximation](https://docs.google.com/document/d/1uS9SH1KpVH31JAmf-iIZ61VazwAF9MrCVwETshBC4UQ/edit), it's not a sufficient model of a slow connection. The [multipliers used in Lighthouse](https://github.com/GoogleChrome/lighthouse/blob/3be483287a530fb560c843b7299ef9cfe91ce1cc/lighthouse-core/lib/emulation.js#L33-L39) attempt to correct for the differences.
1. _Proxy-level_ throttling tools do not affect UDP data, so they're not recommended.
1. _Packet-level_ throttling tools are able to make the most accurate network simulation.

## How do I get high-quality packet-level throttling?

If you want to use more accurate throttling, read on.

This Performance Calendar article, [Testing with Realistic Networking Conditions](https://calendar.perfplanet.com/2016/testing-with-realistic-networking-conditions/), has a good explanation of packet-level traffic shaping (which applies across TCP/UDP/ICMP) and recommendations.

### Using `comcast` for network throttling

The `comcast` Go package appears to be the most usable Mac/Linux commandline app for managing your network connection. Important to note: it changes your **entire** machine's network interface. Also, **`comcast` requires `sudo`** (as all packet-level shapers do).

**Windows?** As of today, there is no single cross-platform tool for throttling. But there are two recommended Windows-specific network shaping utilities: [WinShaper](https://calendar.perfplanet.com/2016/testing-with-realistic-networking-conditions/#introducing_winshaper) and [Clumsy](http://jagt.github.io/clumsy/).

### `comcast` set up

```sh
# Install with go
go get github.com/tylertreat/comcast
# Ensure your $GOPATH/bin is in your $PATH (https://github.com/golang/go/wiki/GOPATH)

# To use the recommended throttling values:
comcast --latency=150 --target-bw=1600 --dry-run

# To disable throttling
# comcast --stop
```

Currently, `comcast` will also throttle the websocket port that Lighthouse uses to connect to Chrome. This isn't a big problem but mostly means that receiving the trace from the browser takes significantly more time. Also, `comcast` [doesn't support](https://github.com/tylertreat/comcast/issues/17) a separate uplink throughput.

### Using Lighthouse with `comcast`

```sh
# Enable system traffic throttling
comcast --latency=150 --target-bw=1600

# Run Lighthouse with its own throttling disabled
lighthouse --throttling.requestLatencyMs=0 --throttling.downloadThroughputKbps=0 --throttling.uploadThroughputKbps=0 # ...

# Disable the traffic throttling once you see "Retrieving trace"
comcast --stop
```
