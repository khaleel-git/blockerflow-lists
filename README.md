# BlockerFlow lists

The block lists that ship inside [BlockerFlow](https://www.blockerflow.net), a content blocker for
Android and Windows, republished here as plain files. You can use them without the app: in Pi-hole,
AdGuard Home, uBlock Origin or a hosts file, or in your own tooling.

There are two lists:

- `pornography/`: about 894,000 adult-site domains, plus the keyword files BlockerFlow's Android app
  uses to catch adult sites that are not on the domain list.
- `web-proxy/`: about 19,000 web-proxy and filter-avoidance domains (sites used to get around a
  blocker).

The files are rebuilt from the signed pack that the app downloads, so what is here is what BlockerFlow
ships. Each domain file is the same list in four formats; pick the one your tool reads.

## Files

Paths are relative to the repository root. `<list>` is `pornography` or `web-proxy`.

| File | Format | Use it for |
|---|---|---|
| `<list>/domains.txt` | One domain per line, sorted. Lines starting with `#` are comments (the header). | Pi-hole and other tools that take a plain domain list. |
| `<list>/hosts.txt` | `0.0.0.0 <domain>` per line, sorted. | A hosts file, Pi-hole, any tool that reads hosts format. |
| `<list>/adblock.txt` | Adblock Plus syntax: `\|\|<domain>^` per line. | AdGuard Home, uBlock Origin. |
| `<list>/domains-with-sources.txt` | `domain\|source_mask` per line, sorted, no header. This is the exact file the app and the signed manifest use. | Checking the signature, or seeing which source listed a domain. |
| `pornography/keywords.txt` | `keyword\|weight\|requiresBoundary` per line. `weight` is a number; `weight >= 100` means high confidence. `requiresBoundary` is `true` or `false` (match whole words only). | Scoring URLs and search terms. |
| `pornography/keywords_intl.txt` | `term\|kind\|lang` per line, `#` comments. `kind` is `strong`, `adult`, `context`, `help` or `cancel`; the header in the file explains each. | Adult keywords in other languages. |
| `pornography/keyword_tiers.txt` | `keyword\|strong` or `keyword\|context`. `strong` blocks on its own; `context` only counts next to another adult term. A keyword not listed is `adult` (two needed). | Tiers for `keywords.txt`. |
| `pornography/adult_hostname_keywords.txt` | One brand root per line, `#` comments. Matched against each dot-separated label of a hostname, also after trailing digits are stripped, so `chaturbate` also matches `chaturbate99.xyz`. | Catching numbered or mirror domains that are not in the domain list. |
| `pornography/page_check_skip.txt` | One host per line, `#` comments. A host or any parent of it matches. | Hosts whose pages the app never reads (large legitimate sites). |
| `manifest.json` | JSON with two fields: `payload` (a string holding the pack version, build time and a path, size and sha256 for each source file) and `signature` (base64, ECDSA P-256 over the `payload` string). | Verifying the lists. See below. |
| `blockerflow_public_key.pem` | The public key for `signature`. | Verifying the lists. |

`source_mask` in `domains-with-sources.txt` is a bitmask of the upstream lists that included the
domain: BLP (Block List Project) = 1, OISD = 2, HaGeZi = 4, StevenBlack = 8. A mask of 5 means both
BLP and HaGeZi listed it. The `web-proxy` list uses the same column; see [NOTICE](NOTICE) for what each
list is built from.

Domain matching: the plain and hosts files list exact names. A hosts file only blocks the names it
lists, not their subdomains. The Adblock syntax (`||domain^`) also blocks subdomains.

## Use the lists

Replace `<list>` with `pornography` or `web-proxy`. Raw URLs have the form:

```
https://raw.githubusercontent.com/khaleel-git/blockerflow-lists/main/<path>
```

For example, `https://raw.githubusercontent.com/khaleel-git/blockerflow-lists/main/pornography/domains.txt`.

**Pi-hole.** In the web interface go to Lists (Adlists in Pi-hole v5), paste
`https://raw.githubusercontent.com/khaleel-git/blockerflow-lists/main/pornography/domains.txt`, add it,
then run `pihole -g` (or Tools > Update Gravity). Add `web-proxy/domains.txt` the same way if you want
that list too. The pornography list has about 894,000 entries, so gravity needs some memory and a
minute to build on small devices.

**AdGuard Home.** Go to Filters > DNS blocklists > Add a blocklist > Add a custom list, and enter
`https://raw.githubusercontent.com/khaleel-git/blockerflow-lists/main/pornography/adblock.txt`.

**uBlock Origin.** Open the dashboard > Filter lists > Custom (at the bottom), paste the same
`adblock.txt` URL, and click Apply changes. This list only blocks network requests to the listed
domains; it does not hide page elements. A list this large slows uBlock Origin's start-up on weak
devices; a DNS filter (Pi-hole, AdGuard Home) is the better fit for the pornography list.

**Hosts file.** Append `hosts.txt` to your hosts file (as an administrator or with `sudo`):

```sh
curl -fsSL https://raw.githubusercontent.com/khaleel-git/blockerflow-lists/main/pornography/hosts.txt \
  | sudo tee -a /etc/hosts > /dev/null
```

On Windows the file is `C:\Windows\System32\drivers\etc\hosts`. Most systems cope with a hosts file of
this size, but Windows' DNS Client service can become slow with several hundred thousand entries; use a
DNS filter there if you see that.

The keyword files are for software that reads them (as BlockerFlow does). No DNS filter or hosts file
uses them.

## Verify the lists

`manifest.json` is signed by BlockerFlow and lists the sha256 of each source file, so you can check
that the files here are the ones BlockerFlow published and were not changed. You need `openssl`, `jq`
and `shasum` (or `sha256sum` on Linux). Run these in a checkout of this repository, or download
`manifest.json`, the public key and the file you want to check into one folder.

1. Check the signature. The signature covers the exact text of the `payload` string, so write it out
   without a trailing newline (`jq -j`, not `jq -r`):

   ```sh
   jq -j .payload manifest.json > payload.txt
   jq -r .signature manifest.json | base64 -d > sig.der
   openssl dgst -sha256 -verify blockerflow_public_key.pem -signature sig.der payload.txt
   ```

   This prints `Verified OK`. Any other output (`Error Verifying Data`) means the manifest or the key
   does not match.

2. Compare a file's hash with the signed one. `domains-with-sources.txt` is the exact file the
   manifest's `pornography/domains.txt` entry describes (`domains.txt` here has a header and no
   masks, so its hash differs). The two hashes must be equal:

   ```sh
   jq -r '.payload | fromjson | .files[] | select(.path == "pornography/domains.txt") | .sha256' manifest.json
   shasum -a 256 pornography/domains-with-sources.txt
   ```

   For `web-proxy`, use `web-proxy/domains.txt` in the first command and
   `web-proxy/domains-with-sources.txt` in the second. The keyword files and
   `adult_hostname_keywords.txt` and `page_check_skip.txt` are published unchanged, so their own sha256
   matches the manifest entry with the same path. `domains.txt`, `hosts.txt` and `adblock.txt` are
   generated from `domains-with-sources.txt`, which is how they are covered.

3. To see the pack version and build time, run `jq -r .payload manifest.json | jq '{version, created}'`.

The public key is also built into the BlockerFlow Android app.

## Licence

The two lists have different licences, because their sources do:

- `pornography/` is GPL-3.0 (full text in [LICENSE](LICENSE)).
- `web-proxy/` is CC BY-SA 4.0 (full text in [LICENSES/CC-BY-SA-4.0.txt](LICENSES/CC-BY-SA-4.0.txt)).
- This README, `manifest.json` and the key are GPL-3.0.

[NOTICE](NOTICE) names every source, its licence, the files built from it, and the copyright notices
that must stay with copies. Read it before redistributing either list.

## Updates

This repository is updated by a script after each stable BlockerFlow list release: it downloads the
signed pack, checks the signature and every file's hash, renders the files above and commits them. Each
commit message names the version, and each release is tagged `v<version>` (for example `v20261105.8`).
The file headers also carry the version and build time. Do not edit the generated files by hand.

The lists come with no warranty and can contain mistakes. To report a wrong entry, open an issue in this
repository. More about the app: https://www.blockerflow.net
