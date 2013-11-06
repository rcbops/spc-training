% Swift Private Cloud
% William Kelly <william.kelly@rackspace.com>
% November 6th, 2013

## Who is responsible for this travesty?

- Ron Pedde (Dev)
- Chris Laco (Engineer)
- Will Kelly (Engineer)
- Jason Cannavale (Bossman)


##  What is this all about?

- The existing swift cookbook attempted to take on too much of the
  operational aspect of swift and not enough of the configuration.

- Operational training on a new mechanism to deploy and manage swift
using the experience of our cloud files team!


Who is it for
-------------

This training is directed toward the support organization regarding
the technical capabilities of our new cookbooks.  The deployment
mechanisms shown here and the particular configurations we cover may
or may not be what is deemed to be operationally acceptable in the
finished product.



Swift Cookbooks
================


swift-lite
----------

- http://github.com/rcbops-cookbooks/swift-lite

- swift-lite is a really basic cookbook that enables swift configuration
via full specification in node attributes.  It does not make many
decisions about how things should be deployed or what services should
go where or what version of swift to install.

- no hardcoded config

- no baked in assumptions about roles or which recipes should go together

- everything is overridable

swift-private-cloud
-------------------

- http://github.com/rcbops-cookbooks/swift-private-cloud

- swift-private-cloud wraps the swift-lite cookbook to implement a very
opinionated Rackspace configuration.  Currently we support deploying
the swift starter config from the reference architecture
documentation.  This cookbook handles feeding information to
swift-lite based on our deployment considerations, but also provides
significant services outside the realm of swift, such as ntp, syslog
consolidation, mail configuration, snmp configs, and sysctl configs.

- Opinionated recipes, but liberal roles

- Avoids reinventing the wheel where possible, wrapping upstream
  cookbooks rather than making everything from scratch.


Labs
-----

- install_swift.md

- environments.md

- swift-middleware.md
