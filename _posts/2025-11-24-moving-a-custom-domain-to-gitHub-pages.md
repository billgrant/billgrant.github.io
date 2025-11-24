---
title: "Moving a Custom Domain to GitHub Pages: Learning Through Iteration"
date: 2025-11-24
categories:
  - infrastructure
  - learning
tags:
  - dns
  - github-pages
  - route53
  - troubleshooting
---

***To kick off my goal of doing more hands on learning with LLMs. I performed a pretty simple task that I have done in past. Moving my custom domain to a new host. Instead of reading the docs and googling any errors (my normal method). I let Claude.ai guide me through the process. Below is an unedited blog post that claude wrote for me after we finished that describes everything that happened.***

Last night I set up a new GitHub Pages blog using the Minimal Mistakes theme. Today, I wanted to move my custom domain from Route 53 to point to it. I've done similar DNS work before, so I thought this would be straightforward. Spoiler: it wasn't quite that simple.

## The Goal

Point `billgrant.io` (registered in AWS Route 53) to my new GitHub Pages site at `billgrant.github.io`.

## Attempt #1: Getting the Order Wrong

I asked for concise instructions and got a clear set of steps. The problem? I had them backwards.

I tried configuring the custom domain in GitHub Pages **first**, then updating Route 53. GitHub immediately threw an error:

```
www.billgrant.io is improperly configured
Domain's DNS record could not be retrieved. For more information, see documentation (InvalidDNSError).
```

What caught me off guard was that I'd entered `billgrant.io` (no www), but GitHub was checking for `www.billgrant.io`. 

**The fix:** Configure DNS in Route 53 **first**, then add the custom domain in GitHub Pages. This makes sense - GitHub needs to verify you actually own the domain by checking DNS records before accepting it.

## Attempt #2: The "Clever" Shortcut

I set up the required A records pointing `billgrant.io` to GitHub's IPs:
- 185.199.108.153
- 185.199.109.153
- 185.199.110.153
- 185.199.111.153

Then I got clever. Instead of pointing the www CNAME to `billgrant.github.io` as recommended, I pointed it to the apex domain:

```
www.billgrant.io -> billgrant.io
```

My reasoning: if I ever move away from GitHub Pages, I'd only need to update the A records, not both the A records and the CNAME. Single point of maintenance.

It worked! The site loaded at both `billgrant.io` and `www.billgrant.io`.

## The SSL Problem

Then I checked HTTPS. `https://billgrant.io` worked fine, but `https://www.billgrant.io` failed.

The issue: GitHub Pages automatically provisions SSL certificates, but only when subdomains point directly to `yourusername.github.io`. My CNAME setup broke this automation because GitHub couldn't properly identify the www subdomain as belonging to my Pages site.

## The Correct Solution

Changed the CNAME to:
```
www.billgrant.io -> billgrant.github.io
```

After DNS propagation (about 5 minutes), both domains worked with HTTPS enabled.

## What I Learned

**1. Order matters in DNS configuration**
Set up DNS records first, then configure the service (GitHub Pages). Not the other way around.

**2. "Clever" shortcuts often have hidden costs**
My maintenance-optimized CNAME broke SSL. The standard configuration exists for good reasons.

**3. You'll update DNS anyway**
When I eventually migrate away from GitHub Pages (if I do), I'll need to update DNS records regardless:
- New hosting provider → new A records or CNAME
- CDN → their specific DNS requirements
- Custom server → new IP addresses

The www CNAME doesn't actually add maintenance burden.

**4. Always test HTTPS**
HTTP working doesn't mean HTTPS works. Always verify SSL certificates are properly issued.

## The Right Setup

For anyone else doing this, here's the working configuration:

**In Route 53 (do this first):**
- Create 4 A records for your apex domain pointing to GitHub's IPs
- Create a CNAME record: `www.yourdomain.com -> yourusername.github.io`

**In GitHub Pages (after DNS propagates):**
- Settings → Pages → Custom domain
- Enter your domain (e.g., `billgrant.io`)
- Wait for DNS check to pass
- Enable "Enforce HTTPS"

## Using AI for Technical Tasks

This whole process was done conversationally with Claude. What I appreciated:
- Quick, concise answers when I said I had experience
- Caught my mistake about ordering
- Explained *why* my clever solution broke SSL
- Provided the reasoning behind the standard approach

The iterative problem-solving was more valuable than just following documentation. I understood the "why" behind each step because we debugged issues together.

## Next Steps

Now that the infrastructure is set up, I can focus on content. I'm planning posts about:
- Learning Python fundamentals (again, after years away)
- Experimenting with MCP servers
- Building simple AI agents
- Infrastructure as Code patterns

The blog is live at [billgrant.io](https://billgrant.io) - dark mode enabled, because of course it is.
