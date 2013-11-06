# Environment Hackery #

Sometimes the assumptions in swift-private-cloud are wrong.  They don't fit the customer environment, or a one-off configuration makes the assumptions invalid (this never happens).

When it does happen, you'll be happy to know that the swift-private-cloud cookbooks are much more configurable than the old swift cookbooks, and significantly more configurable than the nova cookbooks as well.

If you are thinking you need to override something, or change the behavior of a deployed swift cluster, you probably can.  Without having to drop the cluster out of config management.

## Built-in Assumptions ##

Generally, the scope of a swift-private-cloud (as it stands today) is a small cluster with a minimum of 3 storage nodes, 1 proxy node, and 1 management node.  The cookbooks support multiple proxy nodes, but the necessary load balancer is outside the scope of the cookbooks.  As it stands today, multi-proxy configurations must have an external software or hardware load balancer.

The clusters are set up for keystone auth only.  The keystone server can be set up (non-ha) on the management node, or can leverage an external keystone server (for example, one already set up as part of a nova cluster).

Syslog is forwarded to the management node, as is smtp.  NTP is set up to the public ntp pool.  A ring management server is set up on the master, and can distribute rings to the cluster members.

No ring automation is in scope.  Ring pushes, drive additions and removal are all done with operator action and supervision.

## Minimum Environments ##

Two common use cases are standalone swift with keystone, and swift authing against exernal keystone.

### Standalone Swift w/Keystone ###

A minimal environment with built-in keystone looks like this:

    "override_attributes": {
      "swift-private-cloud": {
        "dispersion": {
          "dis_key": "secrete", # dispersion/reporter user
        },
        "network": {
          "management": "10.0.0.0/8" # swift network cidr
          "exnet": "10.1.0.0/8"      # front (lb) network cidr (unused)
        },
        "swift_common": {
          "swift_hash_prefix": "random string",
          "swift_hash_suffix": ""
        },
        "keystone": {
          "auth_password": "secrete",  # service/swift user
          "admin_password": "secrete", # admin/admin user
          "ops_password": "secrete",   # swiftops/swiftops user
          "swift_admin_url": "http://<proxy vip>/v1/AUTH_%(tenant_id)s",
          "swift_public_url": "http://<proxy vip>/v1/AUTH_%(tenant_id)s",
          "swift_internal_url": "http://<proxy vip>/v1/AUTH_%(tenant_id)s"
        }
      }
    }
    
This will install keystone and set up several tenants: 

* `dispersion`: set up with a user named `reporter`.  Used for dispersion reports and cron jobs.
* `service`: set up with a user named `swift` used to for authentication and token verification by the swift proxies.
* `admin`: set up with a user named `admin` as an initial customer tenant.
* `swiftops`: set up with a user named `swiftops` for ops usage.

In the case where there is no load balancer, and a single proxy server, the proxy vip for the swift_url settings is likely port 8080 on an accessible proxy ip.

In the case of an external load balancer, the load balancer can be set up to terminate ssl and the vip can be a proper ssl hostname and port.

### Swift w/External Keystone ###

    "override_attributes": {
      "swift-private-cloud": {
        "dispersion": {
          "dis_key": "secrete", # dispersion/reporter user
        },
        "network": {
          "management": "10.0.0.0/8" # swift network cidr
          "exnet": "10.1.0.0/8"      # front (lb) network cidr (unused)
        },
        "swift_common": {
          "swift_hash_prefix": "random string",
          "swift_hash_suffix": ""
        },
        "keystone": {
          "keystone_admin_url": "http://<keystoneip>:5000/v2.0",
          "auth_password": "secrete",  # service/swift user
          "admin_password": "secrete", # admin/admin user
          "ops_password": "secrete",   # swiftops/swiftops user
        }
      }
    }

The only additionally *required* setting is keystone/auth_uri.  The swift url endpoint need not be specified, as they will not be automatically created, and should be created by hand on the external keystone server.

It's possible that the default tenant and user accounts don't match the actual settings of the external keystone server.  If this is the case, then the tenant and user settings can be set to match required settings:

    "swift-private-cloud": {
      "dispersion": {
        "dis_tenant": "dispersion",
        "dis_user": "reporter"
      },
      "keystone": {
        "ops_user": "swiftops",
        "ops_tenant": "swiftops",
        "auth_user": "swift",
        "auth_tenant": "service",
      }
    }
    
These (along with the password options above) allow for the authentication user, ops user, and dispersion user to be customized to match existing accounts in an external keystone.  They could even be folded into a single tenant/account if desired.

Note that if using an external Keystone server, the accounts will **not** be automatically created.  They must be created by hand using the keystone client, or provided by the customer.

## Tweaking stuff for fun and profit ##

There are a variety of things that can be tweaked in the environment.  A (not complete, but close) list of interesting defaults can be found in the [default attribute file](https://github.com/rcbops-cookbooks/swift-private-cloud/blob/master/attributes/default.rb) of the swift-private-cloud cookbooks.

Some particularly interesting variables:

* rsync connection tuning
  * default["swift-private-cloud"]["rsync"]["config"]["account"]["max connections"] = 8
  * default["swift-private-cloud"]["rsync"]["config"]["container"]["max connections"] = 12
  * default["swift-private-cloud"]["rsync"]["config"]["object"]["max connections"] = 18
* email tuning (cron job alerts)
  * default["swift-private-cloud"]["mailing"] ...
* sysctls
  * default["swift-private-cloud"]["proxy"]["sysctl"]
  * default["swift-private-cloud"]["storage"]["sysctl"]
* keystone region (only for standalone swift)
  * default["swift-private-cloud"]["keystone"]["region"] = "RegionOne"
  



