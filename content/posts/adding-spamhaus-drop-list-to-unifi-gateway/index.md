---
title: Adding The Spamhaus DROP List to Unifi Gateway
date: 2024-03-22T16:14:00.000Z
tags: ["howto","unifi","linux","networking","spamhaus","drop"]
author: Adam
---

### Introduction

The [Spamhaus Don't Route Or Peer (DROP) Lists](https://www.spamhaus.org/blocklists/do-not-route-or-peer/) consist of netblocks that are leased or stolen by professional spam or cyber-crime operations, and used for dissemination of malware, trojan downloaders, botnet controllers, or other kinds of malicious activity. i.e. stuff you really don't want to interact with.

I used to consume the DROP list many years ago when my home firewall was Microsoft ISA/TMG (yes, really), but then completely forgot it existed until fairly recently. Having been reminded about it, I looked at options for ingesting it into my Unifi USG firewall and found that there basically aren't any, which means it's once again time to write something horrible.

This should broadly work with anything that uses the Unifi Network Application (née Controller); so USG, UDM, etc. but I've only tested it with a USG because that's all I have access to right now.

### The Hard Part

First up you need to create an address group and firewall rule for this, call them whatever you want, doesn't matter. Add the firewall Drop rule as early as possible in the chain so you don't waste time processing other rules for traffic from IPs you're just going to drop. You'll need to add at least one address block to the address group, your best option is to grab one from the DROP list.

Once the rule and group have been pushed to the gateway, SSH onto it and run `mca-ctrl -t dump-cfg` to dump the running config and look for your address group, it will be under:

```json
"firewall": {
  "group": {
    "address-group": {
```

Followed by a 24-character unique ID; this is what you need for modifying the group. If you don't already have a custom `config.gateway.json` then create one under `<unifi_base>/data/sites/<site_name>` - in my case as it's a docker volume that path is `/var/lib/docker/volumes/unifi-data/_data/data/sites/default/config.gateway.json` - and copy *just* the firewall block elements that contain your address group into it, i.e.

```json
{
  "firewall": {
    "group": {
      "address-group": {
        "65fd8c338aab5250a7f6e24e": {
          "address": [
            "1.10.16.0/20"
            ],
          "description": "customized-Spamhaus DROP List"
        }
      }
    }
  }
}
```

There's some additional information about the `config.gateway.json` on the [Unifi support site](https://help.ui.com/hc/en-us/articles/215458888-UniFi-USG-Advanced-Configuration-Using-config-gateway-json). If you *do* already have a custom `config.gateway.json` then just add the address group to it. Note that changes made via this method *do not* show up in the UI and any changes you *do* make to those objects via the UI will just get overwritten by the settings in the `config.gateway.json`.

Now that we have something in place to override the controller config, we can inject the DROP list IP ranges into it. Now, unfortunately the DROP list is published as a text file, with comments, and that makes things fiddly because we need clean JSON. There *is* a JSON file but it only lists ASNs and Unifi can't provision firewall rules based on ASN.

So, first we download the DROP list from [https://www.spamhaus.org/drop/drop.txt](https://www.spamhaus.org/drop/drop.txt), and then we perform some surgery:

```bash
cut -d';' -f1 "${DROP_TEXT}" | awk 'NF' | head -c -1 | tr -d " " | jq -R -s -c 'split("\n")' > "${DROP_JSON}"
```

In short, remove everything after the `;` on each line, remove empty lines, remove any trailing newlines, trim any whitespace around the addresses, and finally use `jq` to convert the list into a JSON array. We can then use `jq` again to insert this array into our `config.gateway.json`, replacing the existing values:

```bash
GATEWAY_CONFIG=$(jq ".firewall.group[].\"${FIREWALL_GROUP}\".address = input" "${GATEWAY_JSON}" "${DROP_JSON}")
echo "${GATEWAY_CONFIG}" > "${GATEWAY_JSON}"
```

The config is now up to date, *but* the gateway only reads that file when a provisioning operation is triggered, usually when you update the config via the GUI. So we need to find a way to trigger it on-demand.

Thankfully Unifi has a very poorly-documented API that we can leverage. First we authenticate and then we force a provision of the gateway, using the mac address to identify it:

```bash
curl -X POST --data '{"username": "'"${UNIFI_USER}"'", "password": "'"${UNIFI_PASS}"'"}' --header 'Content-Type: application/json' -c "${COOKIE_PATH}" "https://${UNIFI_HOST}/api/login"
curl -X POST -b "${COOKIE_PATH}" --data-binary '{"mac":"'"${UNIFI_MAC}"'","cmd":"force-provision"}' --header "Content-Type: application/json" "https://${UNIFI_HOST}/api/s/${UNIFI_SITE}/cmd/devmgr"
```

If you're running a self-signed certificate on your controller you'll need to pass the `-k` switch to curl for the API calls to ignore the invalid certificate.

There are two critical differences for the UDM's API:

* The login endpoint is `/api/auth/login`
* All API endpoints need to be prefixed with /proxy/network (e.g. `https://${UNIFI_HOST}/proxy/network/api/s/${UNIFI_SITE}/self`)

### Testing Things

This is all very well and good, but if you accidentally push broken config to your controller you could give yourself serious issues, so you probably want some safeguards in place. There are a few things we can check; for a start we can add a `set -e` to the script to terminate on a non-zero exit code from any of the commands. We can also sanity check the content of the DROP list with something like:

```bash
if ! grep -E -q "([0-9]{1,3}[\.]){3}[0-9]{1,3}\/[0-9]{1,2}" "${DROP_TEXT}"; then
    exit 1
fi
```

i.e. does it contain at least on IP range in CIDR format? Yes, this is an overly simple regex and would match "invalid" IPs like `843.184.674.12/1`, but that's not something we're trying to validate in this case, we just want to know if there's something CIDR-like in the returned data. You can also use `jq` to confirm that the file is valid JSON before you overwrite the existing config file:

```bash
if ! jq -e . >/dev/null 2>&1 <<<"${GATEWAY_CONFIG}"; then
    exit 1
fi
```

You can obviously go further and add error logging output and further checks if you feel the need.

### Tying It All Together

```bash
#!/bin/bash
set -e

UNIFI_USER="${1}"
UNIFI_PASS="${2}"
UNIFI_MAC=""
UNIFI_HOST=""
UNIFI_SITE=""
FIREWALL_GROUP=""
DROP_URL="https://www.spamhaus.org/drop/drop.txt"
DROP_TEXT="/tmp/drop_$(date -I).txt"
DROP_JSON="/tmp/drop_$(date -I).json"
GATEWAY_JSON="/appdata/unifi/data/sites/default/config.gateway.json"
COOKIE_PATH=$(mktemp)

# Fetch DROP List
curl -so "${DROP_TEXT}" "${DROP_URL}"

if ! grep -E -q "([0-9]{1,3}[\.]){3}[0-9]{1,3}\/[0-9]{1,2}" "${DROP_TEXT}"; then
    exit 1
fi

# Convert DROP list to JSON array
cut -d';' -f1 "${DROP_TEXT}" | awk 'NF' | head -c -1 | tr -d " " | jq -R -s -c 'split("\n")' > "${DROP_JSON}"

# Insert drop list into firewall group address list
GATEWAY_CONFIG=$(jq ".firewall.group[].\"${FIREWALL_GROUP}\".address = input" "${GATEWAY_JSON}" "${DROP_JSON}")

if ! jq -e . >/dev/null 2>&1 <<<"${GATEWAY_CONFIG}"; then
    exit 1
fi

echo "${GATEWAY_CONFIG}" > "${GATEWAY_JSON}"

# Login to controller
curl -X POST --data '{"username": "'"${UNIFI_USER}"'", "password": "'"${UNIFI_PASS}"'"}' --header 'Content-Type: application/json' -c "${COOKIE_PATH}" "https://${UNIFI_HOST}/api/login"

# Force provision of gateway
curl -X POST -b "${COOKIE_PATH}" --data-binary '{"mac":"'"${UNIFI_MAC}"'","cmd":"force-provision"}' --header "Content-Type: application/json" "https://${UNIFI_HOST}/api/s/${UNIFI_SITE}/cmd/devmgr"

# Clean up
rm \
  "${COOKIE_PATH}" \
  "${DROP_TEXT}" \
  "${DROP_JSON}"
```

Now we can set up a cron job to run the script on whatever frequency we want. The DROP list changes quite slowly so there's no need to update more than once per hour, in fact once per day is more than enough in most cases. If you try and pull more often than once an hour you'll probably get blocked by Spamhaus, so don't do that.

Make sure you run the script at least once by hand to make sure everything works as expected before you set it up to run automatically; you don't want your gateway going down in the middle of the night due to a typo.
