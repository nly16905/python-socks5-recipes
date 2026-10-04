# python socks5 proxy: working recipes for requests, aiohttp and httpx, plus auth, DNS leaks and rotation

Most people land here with the same error message on screen: `Missing dependencies for SOCKS support`. The code looks right, the proxy works in curl, but Python refuses to route anything through it. That's because `requests` speaks HTTP proxies natively and SOCKS not at all — SOCKS support comes from a separate package that most tutorials forget to mention in the first line.

Below are the setups that actually run, the two characters that decide whether your DNS leaks, and how to keep a script alive when the IP behind the proxy changes.

## The two-minute version

Install SOCKS support alongside `requests`:

bash
pip install "requests[socks]"


Then point the `proxies` dict at your endpoint. Both keys need the same value, because HTTPS requests through a SOCKS tunnel are still tunneled through the same socket:

python
import requests

proxies = {
    "http": "socks5h://user:password@host:port",
    "https": "socks5h://user:password@host:port",
}

r = requests.get("https://httpbin.org/ip", proxies=proxies, timeout=15)
print(r.text)


If you forget the `[socks]` extra, you get the dependency error at request time, not at import time — which is why it looks like a proxy problem instead of a packaging problem.

## `socks5://` versus `socks5h://`

This is the one detail worth internalizing before anything else.

| Scheme | Who resolves the hostname | When it matters |
| --- | --- | --- |
| `socks5://` | Your machine, before the tunnel opens | Target domains resolve locally without trouble |
| `socks5h://` | The proxy server | You want no local DNS lookups at all |

With `socks5://`, your resolver sees the hostname you're about to fetch. The traffic is still tunneled, but the lookup itself tells anyone watching where you're going. Use `socks5h://` for scraping and geo-targeted work. The only time `socks5://` is the better pick is when your local network resolves names that the proxy's resolver cannot.

## Recipes that work outside a single `get()` call

### Session-level config

Setting proxies once on a `Session` keeps the connection pool alive, which means fewer handshakes per request:

python
import requests

session = requests.Session()
session.proxies = {
    "http": "socks5h://user:password@host:port",
    "https": "socks5h://user:password@host:port",
}
session.headers.update({"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"})

print(session.get("https://httpbin.org/ip", timeout=15).text)
session.close()


### Environment variables

`requests` reads `HTTP_PROXY`, `HTTPS_PROXY` and `ALL_PROXY` from the environment unless you pass `trust_env=False`. That makes this viable without touching the code:

bash
export ALL_PROXY="socks5h://user:password@host:port"


Handy in containers and CI, less handy when you need two different proxies in one process.

### Monkey-patching sockets

If a third-party library builds its own connections and ignores `proxies=`, PySocks can patch socket creation process-wide:

python
import socks, socket, requests

socks.setdefaultproxy(socks.PROXY_TYPE_SOCKS5, "127.0.0.1", 6000)
socket.socket = socks.socksocket

print(requests.get("https://httpbin.org/ip", timeout=15).text)


It works, and it's blunt. Anything in the process that creates a socket now goes through the proxy, including the parts you didn't mean to tunnel.

### Raw SOCKS5 client

When you want the tunnel but not HTTP, `python-socks` exposes the connection directly:

python
import ssl
from python_socks.sync import Proxy

proxy = Proxy.from_url("socks5://user:password@127.0.0.1:1080")
sock = proxy.connect(dest_host="check-host.net", dest_port=443)
sock = ssl.create_default_context().wrap_socket(sock=sock, server_hostname="check-host.net")

sock.sendall(b"GET /ip HTTP/1.1\r\nHost: check-host.net\r\nConnection: close\r\n\r\n")
print(sock.recv(4096))


`python-socks` also ships asyncio, trio and anyio variants, and it's the library that `aiohttp-socks` and `httpx-socks` use under the hood.

### Async with aiohttp

Sync code and concurrency don't mix well, so async scrapers need a connector rather than a plain URL:

python
import asyncio
import aiohttp
from aiohttp_socks import ProxyConnector

async def main():
    connector = ProxyConnector.from_url("socks5://user:password@host:port")
    async with aiohttp.ClientSession(connector=connector) as session:
        async with session.get("https://httpbin.org/ip", timeout=15) as resp:
            print(await resp.text())

asyncio.run(main())


One connector can be shared across a session, which keeps the proxy config in one place instead of scattered across every request. For HTTPX, `httpx-socks` wraps the same python-socks core; recent HTTPX builds also accept a `socks5://` URL in the proxy argument if you install the matching extra, so check which one your installed version expects before rewriting anything.

## When the tool only speaks HTTP

Some libraries and CLIs will accept `http://` proxies and nothing else. Rather than fighting them, bridge the two protocols locally with tinyproxy:


Port 8888
Listen 127.0.0.1
upstream socks5 127.0.0.1:6000


Now `http://127.0.0.1:8888` is an HTTP proxy that forwards up the SOCKS5 chain, and every env-var-based tool works again. `sshuttle` is the heavier alternative when you want the whole machine routed rather than one process, but it needs root on the client.

## Passing credentials without breaking the URL

Proxy usernames are increasingly structured rather than simple words. 9Proxy, for instance, encodes targeting and session behaviour inside the username:


<subaccount>-country-<country_code>-st-<state>-city-<city>-isp-<isp_code>-sst-<minutes>-ssid-<session_id>


That means one account can produce a rotating endpoint, a US sticky endpoint and a Frankfurt sticky endpoint, all from the same host and port:

python
sub = "your_sub_user"
password = "your_password"
HOST, PORT = "your_host", 17521

def proxy_url(country="us", sst=15, ssid=None):
    user = f"{sub}-country-{country}-sst-{sst}"
    if ssid:
        user += f"-ssid-{ssid}"
    return f"socks5h://{user}:{password}@{HOST}:{PORT}"


`ssid` is the part worth understanding. It's optional, but it's what lets ten workers pull ten *different* sticky IPs from an identical configuration — useful when each worker needs its own identity instead of competing for the same session. Drop `sst` and `ssid` entirely and every request gets a fresh IP, which is what you want for high-volume scraping and price checks.

If your password contains `@`, `:` or `/`, URL-encode it before dropping it into the f-string. Otherwise you'll spend twenty minutes debugging an auth failure that's really a parsing failure.

One caveat for the deeper setups: the rotating, dashboard-managed SOCKS5 model works with plain username/password authentication. If you need SOCKS5 to also cover UDP — game traffic, some WebRTC paths, certain WebSocket flows — confirm with the provider first, because many SOCKS5 servers are TCP-only.

## Errors you'll hit, and what they actually mean

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `Missing dependencies for SOCKS support` | PySocks not installed | `pip install "requests[socks]"` |
| Connection refused | Wrong host or port | Verify the proxy is listening (`curl --socks5`) |
| Auth failed | Wrong credentials, or unencoded special characters | Re-check the dashboard, encode the password |
| DNS leak | `socks5://` used where the resolution should be remote | Switch to `socks5h://` |
| Everything is slow | Proxy is geographically far from the target | Pick a closer region, or a closer exit |
| WebSocket fails | SOCKS5 server without UDP support | Use a provider with UDP-capable SOCKS5 |

The DNS one is by far the most common, and the most invisible — the script returns data, the IP looks right in the response body, and the local resolver has already leaked the hostname.

## Where the SOCKS5 endpoint comes from

Free proxy lists are where most scripts start and where most of them die. The endpoints are shared, they're usually already blacklisted, and they vanish mid-run. If a job has to finish, you want residential IPs with real geo-targeting and a rotation model you control.

9Proxy runs residential proxies across 20M+ IPs in 90+ countries with HTTP/HTTPS and SOCKS5 support, targeting down to country, state, city, ZIP and ISP. Two product models sit behind the same SOCKS5 endpoint:

- **Residential by IPs** — fixed IPs with unlimited bandwidth per IP. Unused IPs don't expire, and once forwarded an IP stays live anywhere from a few hours to about 24 hours. Requires the desktop app (Windows/macOS/Linux) for local port forwarding, which hands you `localhost:port` entries plus optional authentication.
- **Residential by GB** — pay per gigabyte, generate unlimited endpoints, no app required. Rotating or sticky sessions by request, 180-day traffic validity (unlimited on Enterprise), and username/password or IP-whitelist auth. This is the model that fits cloud scripts and containers, since nothing has to be installed locally.

There's also a public API for generating proxy lists, rotating IPs, checking wallet balance and managing sub-users, which is useful if you'd rather fetch endpoints at runtime than hardcode them.

👉 [Create a 9Proxy account and generate your first SOCKS5 endpoints](https://bit.ly/9-Proxy)

## What each plan costs

Prices below reflect 9Proxy's current published packages. The IP-based and bundle tiers went up in June 2026; GB-based pricing stayed where it was.

**Residential proxies by IPs** (unlimited bandwidth per IP, IPs don't expire):

| Package | IPs | Price | Billing | Buy |
| --- | --- | --- | --- | --- |
| Entry | 100 IPs | $24 | One-off, non-expiring | [Get the 100 IP package](https://bit.ly/9-Proxy) |
| Small | 500 IPs | $72 | One-off, non-expiring | [Get 500 IPs](https://bit.ly/9-Proxy) |
| Popular | 1,000 + 500 bonus IPs | $126 | One-off, non-expiring | [Get 1,500 IPs](https://bit.ly/9-Proxy) |
| Scale | 2,500 IPs | $210 | One-off, non-expiring | [Get 2,500 IPs](https://bit.ly/9-Proxy) |
| Scale | 5,000 IPs | $360 | One-off, non-expiring | [Get 5,000 IPs](https://bit.ly/9-Proxy) |
| Scale | 15,000 IPs | $720 | One-off, non-expiring | [Get 15,000 IPs](https://bit.ly/9-Proxy) |
| Business | 25,000 IPs | $863 | One-off, non-expiring | [Get 25,000 IPs](https://bit.ly/9-Proxy) |
| Business | 50,000 IPs | $1,438 | One-off, non-expiring | [Get 50,000 IPs](https://bit.ly/9-Proxy) |
| Business | 100,000 IPs | $2,300 | One-off, non-expiring | [Get 100,000 IPs](https://bit.ly/9-Proxy) |
| Business | 200,000 IPs | $4,140 | One-off, non-expiring | [Get 200,000 IPs](https://bit.ly/9-Proxy) |
| Business | 500,000 IPs | $8,625 | One-off, non-expiring | [Get 500,000 IPs](https://bit.ly/9-Proxy) |

**Residential proxies by GB** (unlimited endpoints, rotation, 180-day validity):

| Package | Price | Per GB | Validity | Buy |
| --- | --- | --- | --- | --- |
| 5 GB | $15 | $3.00 | 180 days | [Get 5 GB](https://bit.ly/9-Proxy) |
| 50 + 5 bonus GB | $105 | $2.10 | 180 days | [Get 55 GB](https://bit.ly/9-Proxy) |
| 100 GB | $150 | $1.50 | 180 days | [Get 100 GB](https://bit.ly/9-Proxy) |
| 200 GB | $200 | $1.00 | 180 days | [Get 200 GB](https://bit.ly/9-Proxy) |
| 1,000 GB | $800 | $0.80 | 180 days | [Get 1,000 GB](https://bit.ly/9-Proxy) |
| 2,000 GB | $1,500 | $0.75 | 180 days | [Get 2,000 GB](https://bit.ly/9-Proxy) |
| Larger tiers | — | down to $0.68/GB | 180 days (unlimited on Enterprise) | [Check current GB pricing](https://bit.ly/9-Proxy) |

**Bundle packages** (IPs plus GB traffic in one purchase):

| Bundle | Contents | Price | Buy |
| --- | --- | --- | --- |
| Starter | 100 IPs + 5 GB | $30 | [Get the Starter bundle](https://bit.ly/9-Proxy) |
| Popular | 1,500 IPs + 50 GB | $180 | [Get the Popular bundle](https://bit.ly/9-Proxy) |
| Pro | 5,000 IPs + 500 GB | $720 (list price $860) | [Get the Pro bundle](https://bit.ly/9-Proxy) |

Payment options include cards, Apple Pay, Google Pay, Alipay and crypto via CoinPayments (USDT, BTC, ETH, LTC, DOGE and others). Crypto payments currently come with a +5% IP bonus.

## Which model fits a Python workflow

If your script benefits from unlimited data through a small number of stable identities — long authenticated sessions, heavy per-IP throughput, jobs where you can't predict the bandwidth — the IP-based packages are the cheaper shape. One hundred IPs with no traffic meter goes a long way when each request returns a large payload.

If your script rotates constantly and each request moves a few kilobytes, GB pricing wins. A rotating endpoint per request, no app to install, no port management, and threads don't have to coordinate over a shared port pool. Small scraping jobs frequently burn only a few gigabytes.

A reasonable starting point is the 5 GB tier or the 100 IP package, run your real target through it, and see which unit you're actually consuming before buying volume. Third-party reviews of 9Proxy consistently flag the same thing: there's no clearly advertised free trial, so validation happens with a paid but small package. The company has periodically run free-IP giveaway campaigns and occasionally offers limited trials subject to availability, so it's worth asking support before you buy if you want to test first.

## Two limitations to read before you commit

The refund terms are narrow — credit-based refunds essentially cover IPs that die within about a minute, verified through the Today List. IPs that survive longer but perform badly against your target aren't covered, which is exactly why a small first purchase beats a large one.

Also worth confirming: 9Proxy's IP-based plans are no longer positioned for media streaming under its updated acceptable use policy. If your Python project involves pulling video, check the current terms rather than assuming.

## Quick answers

**Does `requests` support SOCKS5 at all?** Not on its own. Install `requests[socks]` and pass a `socks5://` or `socks5h://` URL in the `proxies` dict.

**Can I use SOCKS5 with Scrapy directly?** Scrapy's built-in proxy middleware is HTTP-oriented. The usual workarounds are a local HTTP-to-SOCKS bridge like tinyproxy, or a third-party middleware — either way, keep one endpoint per spider setting instead of threading credentials through every request.

**Rotating or sticky for scraping?** Rotating for anything anonymous and parallel; sticky when a site ties state to an IP, like a cart or a login session. With 9Proxy's BY-GB model you switch by adding or removing `sst` and `ssid` from the username, no code change beyond the string.
