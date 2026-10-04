# axios proxy: Why HTTPS requests ignore the built-in option, and how to fix it with agents, SOCKS5, and rotation

The first clue is usually silence. You add a `proxy` object to your axios call, run it against an echo endpoint, print the response — and see your own IP. No exception, no warning, just a request that took the direct route while your code believed it went through a proxy.

That outcome is common enough that it gets reported as a bug, patched with a workaround, and reported again on the next Node release. The workaround has been stable for years, and it takes about four lines. What changes between projects is everything downstream: authentication, rotation, and the IPs sitting behind it all.

## What axios actually does with a proxy config

Axios accepts a `proxy` object with `host`, `port`, `protocol`, and optional `auth`. It also resolves `HTTP_PROXY`, `HTTPS_PROXY`, and `NO_PROXY` through its `proxy-from-env` dependency, so environment variables work in axios 1.x without extra packages. In a recent Node version with the built-in env-proxy mode enabled, axios hands proxy resolution to Node instead.

For plain `http://` targets, that machinery usually behaves. For `https://` targets it has to build a CONNECT tunnel first, and that's where setups diverge. Some axios and Node combinations send the request to the proxy as if the proxy were the origin server. Others appear to ignore the proxy entirely. Version-specific quirks have been reported repeatedly across the axios issue tracker and Stack Overflow, and the symptoms rarely look like a proxy problem: a successful response containing your real IP, an unexplained `ECONNRESET`, or a target that suddenly starts rate-limiting your home connection.

Worth being precise here, because the internet is not. Current axios 1.x does tunnel HTTPS through a plain HTTP proxy in ordinary cases, and many people run the built-in `proxy` option without ever seeing trouble. The failing configurations are the ones that generate search results, because the people they work for don't post anything. So treat this as a reliability problem rather than a universal law: the built-in option's behaviour depends on your axios version, your Node version, and whether anything else already claimed the connection. An explicit agent behaves the same way everywhere.

## The fix: `httpsAgent` plus `proxy: false`

bash

npm install https-proxy-agent



js

import axios from "axios";

import { HttpsProxyAgent } from "https-proxy-agent";

const agent = new HttpsProxyAgent("http://user:pass@203.0.113.7:8080");

const res = await axios.get("https://httpbin.org/ip", {

httpsAgent: agent,

proxy: false, // stop axios's own proxy handling

timeout: 15_000,

});

console.log(res.data);



Both extra properties matter. `httpsAgent` gives you the CONNECT tunnel and answers the proxy's authentication challenge on your behalf, which is exactly the part the built-in option is least reliable at. `proxy: false` turns off axios's own proxy resolution so it doesn't read the environment or apply its own logic on top of the agent you just passed. Skip it and you're back to two systems trying to own one request.

`timeout` isn't decoration either. A proxy exit that stops responding mid-request will keep a connection open until the socket gives up, and if you're running this inside a queue or a worker, that stall propagates. Fifteen seconds is a reasonable starting point for scraping and API polling; adjust to your target's real latency rather than to your patience.

To verify routing, hit an endpoint that echoes the caller's IP — `https://httpbin.org/ip` or `https://api.ipify.org?format=json`. If the response shows your proxy's address, the tunnel is working. If it shows your own, something else is deciding routing.

## One proxy for a whole module

If every request in a module should leave through the same exit, `axios.create()` keeps the config in one place instead of repeating it per call:

js

const client = axios.create({

httpsAgent: agent,

proxy: false,

timeout: 15_000,

});



The trade-off is scope: instance defaults apply to the instance, not per request. That's fine for a single-proxy script and useless the moment you want each request to leave from a different IP. For that, see the rotation section below.

## Environment variables and the double-config trap

Axios reading `HTTPS_PROXY` is convenient until something else also reads it. Node gained a native environment proxy mode in v24.0.0, backported to v22.21.0, enabled with `NODE_USE_ENV_PROXY=1` or the `--use-env-proxy` flag. If that mode is on and you're also passing a proxy config to axios, two layers are making routing decisions, and the loser is whichever one you were relying on.

Pick one owner and stay consistent:

- **Let Node route.** Enable the flag, don't pass a `proxy` config to axios, and control everything with environment variables.

- **Let axios route.** Leave the flag off and rely on `HTTP_PROXY` / `HTTPS_PROXY`.

- **Take control.** Set `proxy: false` and supply your own agent. This is the option you want as soon as rotation or per-request routing enters the picture.

One more check before you blame the code: `NO_PROXY` matching. A pattern that's broader than you intended will quietly send some hosts direct while everything else tunnels, which produces a debugging session where half your requests look proxied and half don't.

## SOCKS5: install the agent, set both keys

Axios has no native SOCKS support. Setting `protocol: "socks"` in the built-in proxy object throws an assertion — `ERR_ASSERTION: protocol mismatch`, where axios expected `http:` and got your SOCKS scheme. Not a helpful error message, but a definitive one.

bash

npm install socks-proxy-agent



js

import axios from "axios";

import { SocksProxyAgent } from "socks-proxy-agent";

const agent = new SocksProxyAgent("socks5h://user:pass@gate.example.com:1080");

const res = await axios.get("https://httpbin.org/ip", {

httpAgent: agent, // http:// targets

httpsAgent: agent, // https:// targets

proxy: false,

timeout: 15_000,

});



Axios picks the agent by the target's scheme, so pass the same instance to both keys or requests to one scheme will bypass the proxy while the other routes correctly. And mind the scheme prefix: `socks5://` resolves DNS locally, `socks5h://` sends the hostname through the proxy. If internal hostnames fail to resolve or you'd rather your resolver didn't see every target, the `h` is the one you want.

## Credentials, 407, and characters that break URLs

With agents, credentials live in the proxy URL as `user:pass@host:port`. If your password contains `@`, `:`, `/`, or `#`, percent-encode it before building the string, or the URL parses into something the proxy rejects:

js

const proxyUrl = `http://${user}:${encodeURIComponent(pass)}@host:8080`;



A wrong or missing password comes back as HTTP 407, which is genuinely useful information: it confirms the proxy answered and the problem is your credentials. Axios surfaces this clearly — `error.response.status === 407` — while undici's built-in fetch reports the same situation as a generic `fetch failed` with an unhelpful cause. If you're debugging auth between two clients, axios gives you a much shorter path to the answer.

## Rotation: a pool, an interceptor, and exactly one retry

Randomly picking a proxy at the call site works until you have more than one call site. The version that survives real load keeps the pool in one place:

js

import axios, { AxiosError, type InternalAxiosRequestConfig } from "axios";

import { HttpsProxyAgent } from "https-proxy-agent";

class ProxyPool {

private agents: HttpsProxyAgent<string>[];

private index = 0;

constructor(urls: string[]) {

this.agents = urls.map((u) => new HttpsProxyAgent(u));

}

next() {

const agent = this.agents[this.index];

this.index = (this.index + 1) % this.agents.length;

return agent;

}

}

const pool = new ProxyPool([

"http://user:pass@us-gate.example.com:8080",

"http://user:pass@de-gate.example.com:8080",

]);

const client = axios.create({ timeout: 15_000 });

client.interceptors.request.use((config: InternalAxiosRequestConfig) => {

config.proxy = false;

config.httpsAgent = pool.next();

return config;

});



Two details bite people here. Request interceptors run last-in-first-out, so if you also have interceptors adding auth headers or logging, their order relative to this one matters more than you'd expect. And retries need a boundary: an unconditional retry against a large pool turns one bad exit into a cascade, and retrying a `POST` can duplicate a side effect you specifically didn't want twice.

js

type RetryableConfig = InternalAxiosRequestConfig & { __proxyRetryCount?: number };

client.interceptors.response.use(undefined, async (error: AxiosError) => {

const config = error.config as RetryableConfig | undefined;

if (!config) throw error;

const method = String(config.method ?? "get").toUpperCase();

config.__proxyRetryCount ??= 0;

if (method !== "GET" || config.__proxyRetryCount >= 1) throw error;

config.__proxyRetryCount += 1;

config.httpsAgent = pool.next();

return client.request(config);

});



One retry, `GET` only, then fail loudly. If your workload needs retries on writes, make it explicit per endpoint rather than global.

## When it breaks, the error tells you where

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `ECONNREFUSED` | Wrong host/port, or the proxy is down | Confirm outside axios first: `curl -x http://host:port https://target` |
| `407 Proxy Authentication Required` | Credentials missing, wrong, or unencoded | Put `user:pass` in the agent URL, percent-encode special characters |
| Response contains your real IP | `NO_PROXY` matched, `proxy: false` with no agent, or two proxy systems active | Decide which layer owns routing, then verify with an IP echo endpoint |
| `ETIMEDOUT` | A slow or dead exit, or a timeout shorter than the target's real latency | Measure, then replace the exit or raise the timeout deliberately |
| `ECONNRESET` mid-response | The proxy or origin closed the connection | Log which exit failed, retry only replay-safe requests |
| `ERR_ASSERTION: protocol mismatch` | SOCKS scheme passed to axios's built-in `proxy` option | Use `socks-proxy-agent` instead |
| `ERR_TLS_CERT_ALTNAME_INVALID` | Wrong agent for the protocol, or a self-signed cert | Match the agent to the scheme; keep `rejectUnauthorized: false` out of production |

## Where the actual IPs come from

Everything above assumes you have a proxy endpoint worth pointing at. Free proxy lists will pass a test once and fail within the hour, and datacenter IPs are recognized as hosting ranges by most targets with serious anti-bot logic. Residential IPs are registered to consumer ISPs, which is why they behave like ordinary visitors at the IP reputation layer.

That's the point where the choice of provider starts affecting whether your axios code works at all. 9Proxy runs a residential network of 20M+ IPs across 90+ countries with HTTP(S) and SOCKS5 endpoints, and city, state, ZIP, and ISP-level targeting on its bandwidth plans. 👉 Set up a 9Proxy account with the invite code already applied

### Two purchase models, and only one of them suits a headless Node process

**Residential by GB.** You buy traffic, not IPs. You generate unlimited endpoints from the dashboard, authenticate with a username/password or an IP whitelist, and choose sticky or rotating sessions per endpoint. Traffic is valid for 180 days, and forever on Enterprise plans. No desktop app is involved, which means the endpoint strings drop straight into the `HttpsProxyAgent` or `SocksProxyAgent` URL you're already building. Sticky mode is what you want for anything involving a login state across several axios calls; rotating mode is what you want for per-request exits.

**Residential by IPs.** You buy a fixed number of IPs with unlimited bandwidth while they're active. Unused IPs never expire, and each IP lasts anywhere from a few hours to about 24 hours. The catch is architectural: this model requires the 9Proxy desktop app, which does local port forwarding and optional proxy authentication, and supports auto-rotation on selected ports. That's convenient for a browser or a tool without native proxy settings. It's awkward for a containerized axios worker, because now your traffic depends on an app running next to it.

If your code runs on a VPS or in a container, the GB model is the one that fits. 👉 Start with a small GB package and test it against your real targets

Two smaller things worth knowing before you top up. If an IP fails within the first 60 seconds of activation, you can swap it at no charge — a policy most providers don't offer, since they treat a dead connection as a consumed resource. And proxies you accessed in the previous 24 hours can be reused at no extra cost, which trims waste during testing.

Sign-up on 9Proxy runs on invite codes rather than open self-serve registration, which is why the link below includes one. 👉 Generate proxy endpoints in the dashboard after you register

### Current published packages

Plans are balance-based, so most purchases happen inside the dashboard after you register rather than on a per-plan checkout page. Prices below are the current published rates.

| Type | Package | What you get | Price | Validity | Sign up |
| --- | --- | --- | --- | --- | --- |
| GB-based | 5 GB | 5 GB traffic, unlimited endpoints | $15 ($3.00/GB) | 180 days | [ Get 5 GB](https://bit.ly/9-Proxy) |
| GB-based | 50 GB + 5 GB bonus | 55 GB total | $105 ($2.10/GB) | 180 days | [ Get 50 + 5 GB](https://bit.ly/9-Proxy) |
| GB-based | 100 GB | 100 GB traffic | $150 ($1.50/GB) | 180 days | [ Get 100 GB](https://bit.ly/9-Proxy) |
| GB-based | 200 GB | 200 GB traffic | $200 ($1.00/GB) | 180 days | [ Get 200 GB](https://bit.ly/9-Proxy) |
| GB-based | 1,000 GB | 1,000 GB traffic | $800 ($0.80/GB) | 180 days | [ Get 1,000 GB](https://bit.ly/9-Proxy) |
| GB-based | 2,000 GB | 2,000 GB traffic | $1,500 ($0.75/GB) | 180 days | [ Get 2,000 GB](https://bit.ly/9-Proxy) |
| Enterprise GB | 3,000 GB | 3,000 GB, team mode | $2,160 ($0.72/GB) | No expiry | [ Get Enterprise 3,000 GB](https://bit.ly/9-Proxy) |
| Enterprise GB | 6,000 GB | 6,000 GB, team mode | $4,200 ($0.70/GB) | No expiry | [ Get Enterprise 6,000 GB](https://bit.ly/9-Proxy) |
| Enterprise GB | 10,000 GB | 10,000 GB, team mode | $6,800 ($0.68/GB) | No expiry | [ Get Enterprise 10,000 GB](https://bit.ly/9-Proxy) |
| Bundle | Starter | 100 IPs + 5 GB | $30 | 180 days on traffic | [ Get the Starter bundle](https://bit.ly/9-Proxy) |
| Bundle | Popular | 1,500 IPs + 50 GB | $180 | 180 days on traffic | [ Get the Popular bundle](https://bit.ly/9-Proxy) |
| Bundle | Pro | 5,000 IPs + 500 GB | $720 | 180 days on traffic | [ Get the Pro bundle](https://bit.ly/9-Proxy) |
| IP-based | 100 IPs | 100 IPs, unlimited bandwidth per IP | $24 (≈$0.24/IP) | Unused IPs don't expire | [ Get 100 IPs](https://bit.ly/9-Proxy) |
| IP-based | 500 IPs | 500 IPs, unlimited bandwidth per IP | $72 (≈$0.14/IP) | Unused IPs don't expire | [ Get 500 IPs](https://bit.ly/9-Proxy) |
| IP-based | 1,000 IPs + 500 bonus | 1,500 IPs total | $126 | Unused IPs don't expire | [ Get 1,000 + 500 IPs](https://bit.ly/9-Proxy) |
| IP-based | 100,000 IPs | Wholesale tier | $2,300 | Unused IPs don't expire | [ Get 100,000 IPs](https://bit.ly/9-Proxy) |
| IP-based | 500,000 IPs | Wholesale tier | $8,625 | Unused IPs don't expire | [ Get 500,000 IPs](https://bit.ly/9-Proxy) |

IP-based plans require the 9Proxy desktop app; GB-based and bundle plans work directly from the dashboard.

## Matching a plan to an axios workload

The billing model matters more than the headline rate, because it decides what your code can afford to do.

**Rotating per-request scraping with small payloads.** GB-based, starting small. Every request gets a fresh exit, bandwidth per request is a few hundred kilobytes, and 5 GB goes further than the number suggests. 👉 Check the current GB and bundle prices before you pick a tier

**Sticky sessions with logins or multi-step forms.** Sticky mode on a GB plan, or an IP-based package if you need the same address for a long run. Hold one exit across the whole flow instead of rotating mid-session, or you'll be logged out by your own code.

**Bandwidth-heavy jobs — media, downloads, large JSON dumps.** IP-based. Unlimited bandwidth per IP means a 2 GB download costs the same as a 2 KB request. On GB billing the same job burns through your balance.

**Teams and agencies splitting traffic across people.** Enterprise. It adds unlimited data validity, one owner plus up to five members, per-member traffic controls, and activity logs, with shared bandwidth that doesn't expire inside the team.

One practical note: don't guess the sticky-session format. The dashboard's proxy generator hands you the exact endpoint strings with your chosen geotargeting and session mode baked in, exportable as `.txt` or `.csv` with code samples in several languages. Copy those into your agent URL rather than assembling the username yourself.

## Before you commit

Third-party resellers reported service interruptions during 2026, with GB-based access returning before IP-based stock, and their write-ups are understandably framed around the alternatives they sell. Treat that as a signal about budget proxy economics in general rather than a verified verdict: hosting a large residential network at $0.68/GB is a hard business, and any provider in that price band carries availability risk.

So structure your code for it. Keep the proxy list in a pool, not in a constant, and keep a second provider's endpoints in that same pool so a bad week costs you an error rate rather than a pipeline. Start with a small top-up, run your own targets for a week, and measure success rate, latency, and stickiness before you buy a five-figure IP package.

---

**Does `proxy: false` disable proxying?** It disables axios's automatic proxy resolution — the environment variables and the built-in `proxy` object. Your agent still routes the request. That's the point: one owner for routing.

**Why does my request show my real IP?** Either no agent is attached, `NO_PROXY` matched the host, or axios and Node are both handling proxy resolution and the wrong one won. Attach an agent, set `proxy: false`, and test against an IP echo endpoint.

**Do I need both `httpAgent` and `httpsAgent`?** For SOCKS, yes — pass the same instance to both. For HTTP proxies, `httpsAgent` covers HTTPS targets; add `httpAgent` if you also call plain HTTP URLs.

**Can I rotate IPs without rewriting my call sites?** Yes. Put rotation in a request interceptor and bounded retries in a response interceptor, as above. Then every call using that instance rotates automatically.
