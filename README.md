# A better DNS

## Scope

This document is a high-level description, essentially a specification,
of a best-of-class DNS for a *single company*.

This specification does *not* cover:

- Implementation details, caching, authorities
- Vendors, brands
- Specific hardware or software
- Global DNS infrastructure


## Why

Over the years, I've worked in many companies as a software developer.  In each company I had to use, configure, deploy-on, and test systems with many hosts on many networks.

One thing I noticed, is that the way DNS is designed from an early days startup to a big scale mega-company, can make a remarkable difference in both:

- quality and security and
- velocity (speed of getting things done).

The best DNS designs I've seen, have made everyone using them more productive, happier & less likely to make mistakes.  On a deeper level, I've found organizations with best-of-breed DNS designs much better positioned to innovate quicker, deploy changes at faster pace, making changes easier to verify/test and generally be better companies all else being equal.

This document will describe what I consider the best practice in DNS for a company.

## DNS specification details

#### Basic functional naming

Assume our domain name is `company.com`.

In this document I describe how does DNS work for employees, partners and users who try to access company hosts and whose DNS is configured to use the company DNS service.

Every host has a function. It can be a mail-service, a personalization engine, a database, or a collection of the above.  As computers become more powerful, smaller, and commoditized they tend to grow in number and the importance of giving them good names which reflect what they do, increases.

For example, suppose we are about to design, implement, develop, test & deploy an email service, it would make sense to use `mail` as our basic-function name.

In order to be more generic, in what follows, I will name the same cluster
of machines `xyz` in all examples rather than picking a name like `mail`.

When we set up the new `xyz` service, both:

    xyz
    xyz.company.com

Should work fine, and resolve to the host which provides our `xyz` service.


#### General vs specific host names

As we grow and get more users, the number of hosts performing the `xyz` function will grow.  For the sake of automating various deployment and diagnosing/testing functions it is a good idea to call the individual boxes:  `xyz1`, `xyz2`, ..., `xyzN`.  We can keep adding boxes and scale out in uniform fashion.

When someone uses the non-specific names `xyz` or `xyz.company.com` both must resolve to one of the boxes implementing the `xyz` service in our company. More details below.

#### High-availability & Cluster naming

We could use round-robin DNS so that `xyz` resolves to any of the specific boxes in a round robin fashion, or use a load-balancer which will take care of additional functions such as:

- Periodic health checks
- Uniform load distribution
- Denial of service mitigation
- Forwarding and redirecting

The important point is that our DNS should make sure that both `xyz` and `xyz2` resolve correctly. `xyz` will resolve to the load-balancer bank for the `xyz` service, while `xyz2` will resolve to a specific machine behind the load-balancer.

Now if `xyz2` happens to crash and burn, users who use the name `xyz` will not be affected. On the other hand, developers who try to diagnose, test or isolate the problem, would always be able to refer to the more specific node, `xyz2`.


#### Hierarchy

As we grow and deploy our service in more regions around the globe, we would start to host the `xyz` service in multiple data-centers around the world.  Say we have one data-center in the east-coast of the US `us-east` and one in the west coast `us-west` we would start having hosts named like: 

    xyz32.us-west.company.com

At this point, we must make sure that a name like:

    # note: missing number
    xyz.us-west.company.com

Works just as well. It should, for example, resolve to the load-balancer fronting the `xyz` service in our `us-west` data-center.

We may also add a CDN (content delivery network) to make sure our users are served in the most efficient manner.  We must make sure that our old names:

    xyz
    xyz.company.com

Still resolve just fine in our DNS, to the data-center, or cluster, or node that is closest to the user making the non-specific request.

In addition, to allow developers, deployers and testers to refer to more specific clusters, we must make sure that the following names work as well from anywhere our DNS is in effect:

    # shorter names (no .company.com) always work as well
    xyz37.us-west
    xyz.us-east

At any point of DNS expansion, we must ensure that our DNS service works in the "least astonishment principle" way.


#### Configuration files

Code should never include specific host-names, but it should always allow overriding such names from the command line, e.g. when we need to test a specific node.
 
Configuration files or databases often have hostnames in them.  In order to prevent errors, and too many variations of the configurations, we must leverage our DNS infrastructure to the max by using the most generic name `xyz` everywhere possible.

#### Reverse DNS

Every hostname served by our DNS *must* support reverse lookups.  We must never be in a situation where we get some malware traffic from (say) 10.23.5.67 and we can't figure out that this IP belongs to `joe-schmoe-laptop`

This shouldn't prevent us from giving fancier names via DNS aliases (CNAMES), but the reverse DNS with a identifiable/actionable target should always exist.

When an unkown IP, like one belonging to some guest on a guest network comes up, its name must be resolved (dynamically) too. A god name would be something like: `guest-<MAC-address>`

#### DHCP and IP classes

Whenever possible, we should make DHCP sticky for at least a year.  I've worked in messed up environments were servers would get different IPs on every reboot making diagnosing and fixing issues a nightmare.

With private IPs in the 10.x.x.x block (or IPv6) there should be no shortage of IPs to assign.

If you use IPv4 internally, use 10.x.x.x whenever possible. Don't
waste time on smaller address spaces that are longer to type and
harder to remember like 192.168.x.x.


#### Split DNS

DNS should behave slightly differently based on where the DNS request comes from.

User desktops reverse DNS names should be visible to internal users only.

If you come from the outside you should not be able to drop the `.company.com` top-level parts of the domain.

#### Testability

One thing I found to be a big drain on resources is the concept of multiple environments with ever so slight differences between them.  The more differences we introduce, the more likely it becomes that something that works in environment X doesn't work in environment Y.

With the above scheme, it is possible to test new code in the same environment without breaking things and without external users being affected by new code.

One way to achieve this is to tell a load-balancer not to direct any requests to a bank of test or early-deployment hosts. Say we decide that `xyz90` to `xyz99` in each sumdomain are always out-of-range for load balancer traffic, we can test on these machines (as long as they don't leave side effects by writing into local databases) by simply referring to a specific out-of-service hostname.

Alternatively, we can have these machines fail the load balancer health-check: e.g. return HTTP 500 whenever the URL path `/are-you-ok` is requested by the load-balancer.


## Summary

A good DNS design makes all the following names possible and resolve to what makes sense the most from the PoV of the requester:

    xyz                     # most generic, closest to user
    xyz3                    # specific node, closest to user
    xyz.<colo>              # generic in data-center <colo>
    xyz3.<colo>             # specific in data-center <colo>
    xyz.company.com         # generic
    xyz3.company.com        # more specific
    xyz.<colo>.company.com  # specific load balancer
    xyz3.<colo>.company.com # most specific (single node)

    

