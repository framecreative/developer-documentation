---
title: Server Access & Firewalls
parent: Hosting
order: 1
---
# Firewalls and Server Access
Frame servers and applications typically have between 1-3 levels of protection applied to them. Understanding these levels of protection can be important for configuration, along with troubleshooting blocked access.

## Web Application Firewalls

Cloudflare can do some filtering and blocking at the CDN / DNS level. Typically this doesn't show up much, but checking if the domain in questions is under cloudflare is useful when troubleshooting wierd connection behaviour. We can bypass Cloudflare using the setting on the DNS page. It's also worth checking for Page Rules as a cause of wierd behaviour.

## Data Centre Level Firewall
Vultr servers have access to Vultr's firewall. Our typical web server is within the Web Server firwall group - this blocks all incoming connections / ports with the exception of the following
 - `port 80` For http based web connections
 - `port 443` for https based web connections
 - `port 22` for SSH connections

 If you're trying to access the server via a non-standard port or protocal, this could be reason the connection is failing.

 ## Fail2Ban Server Level Firewall
 All ServerPilot configured servers come with Fail2Ban installed by default. We do not configure specific rules within this service, so typically any entries within F2B come from blocked SSH connections. If you make too many unsuccessful attempts at connecting via SSH, the IP will be temporarily banned. This will usually reset within 15 minutes. 

If you need to regain immediate access, please see the troubleshooting below.

## Troubleshooting

### Removing Fail2Ban SSH / IP Tables IP lockouts

If you enter the wrong credentials too many times, or try to login with a username that doesn't exist, the serverpilot Frame servers will generally add you to a Fail2Ban block list for around 15 minutes - more repeated attempts will extend the block or get the IP you're trying to access the server from permanently banned.

As we all use the same IP address at Frame, this isn't great! The issue usually presents itself as

```bash
# THE_HOST represents the host we're trying to connect to
ssh: connect to host ${THE_HOST}.frame.hosting port 22: Connection refused
````

**Step 1 : Change IP Address**

To login and fix the problem, first we need to get a new, unbanned, IP. Common ways to do this are

- Tether your phone to your computer and access the internet via this hotspot
- Use the Frame VPN via [Open VPN](https://openvpn.net/), utilising the credentials in 1Password

**Step 2: Login as the `frame` user**

The frame user is the `sudo` enabled user on the server, so we'll need to be logged in as `frame` in order to `sudo` or switch to the `root` user.

**Step 3: Check IP Tables**

We can see the active ban rules using the following command. `iptables` is a single command tool, it utilises flags to do different things. In this example we're passing the "list" flag with `-L` to list the current rules.

`sudo iptables -L --line-numbers`

Example Output

**Step 4: Remove specific rules, or flush the whole chain**

If we're able to find the blocked IP in the list output by the command above then we can either remove that rule specifically using it's chain and line number, or flush the whole chain to remove all rules from that chain. 

_A chain is really the same as a group - it usually refers to the type of block or it's origin. Common chains on our server are INPUT, OUPUT and f2b-sshd_
{: .d-inline-block .p-4 .bg-grey-lt-300}

To remove a specific rule by it's chain and line number

```bash
# -D flag is for "delete"
# Will remove line item #3 in the f2b-sshd chain
sudo iptables -D f2b-sshd 3
````

To flush all rules in a chain

```bash
# -F flag is for "flush"
# Will flush all rules in the Fail2Ban SSH Chain, this is usually the one responsible for SSH lockouts
sudo iptables -F f2b-sshd
```

**Step 5: Check Access**

You should be able to login again after this has been compelted.

**I can't find my IP**

The initial F2B rules are often just 10 or 15 minute bans - so in some cases, the rule may have expired and purged itself by the time you login and check on it! If you can't see a rule that corresponds to [your current IP](https://whatismyipaddress.com/) then it's best to just try connecting again via SSH. If this issue persists, it's likely something else. 