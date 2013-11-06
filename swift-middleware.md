# Installing Custom Middleware #

Sometimes, middleware needs to be added or configured differently than
the assumptions made in the swift-private-cloud cookbook.

Fortunately, there are provisions for modifying to configurations to
achieve different goals.

As an example, let's install Swift's "Static Web" middleware, as
described in the [OpenStack Object Storage
manual](http://docs.openstack.org/api/openstack-object-storage/1.0/content/Create_Static_Website-dle4000.html).

According to this documentation, the proxy server configuration must
be changed to include the following sections:

    [DEFAULT]
    ...

    [pipeline:main]
    pipeline = healthcheck cache tempauth staticweb proxy-server

    ...

    [filter:staticweb]
    use = egg:swift#staticweb
    # Seconds to cache container x-container-meta-web-* header values.
    # cache_timeout = 300
    # You can override the default log routing for this filter here:
    # set log_name = staticweb
    # set log_facility = LOG_LOCAL0
    # set log_level = INFO
    # set access_log_name = staticweb
    # set access_log_facility = LOG_LOCAL0
    # set access_log_level = INFO
    # set log_headers = False

In general, only two things need to be done. A middleware stanza must
be added for staticweb, and it must be added to the application
pipeline. Simple enough.

## Swift Config Files ##

In prior cookbooks, configuration files were only able to be templated
to the degree that the chef templates were wired to allow it. With the
new swift cookbooks, however, all values in all the major config files
are able to be customized.

The config files themselves are stored as a big json blob in chef
itself. These json blobs are serialized into the config files on each
run.

For example, a section of the account server config file might look
like this:

    [DEFAULT]
    backlog = 4096
    bind_ip = 0.0.0.0
    bind_port = 6002
    workers = 6

    [filter:healthcheck]
    use = egg:swift#healthcheck

    [app:account-server]
    use = egg:swift#account

    [pipeline:main]
    pipeline = healthcheck account-server

The json blob that corresponds to this config snippet is at
`node['swift-private-cloud']['account']['config']`, and looks like
this:

    {
      "DEFAULT": {
        "backlog": 4096,
        "bind_ip": "0.0.0.0",
    	 "bind_port": 6002,
    	 "workers": 6
      },
      "filter:healthcheck": {
        "use": "egg:swift#healthcheck"
      },
      "pipeline:main": {
        "pipeline": "healthcheck account-server"
      }
    }

The configuration doesn't have to be specified as a json blob,
however. Individual settings can be overridden in the environment with
an attribute like this:

    "override_attributes": {
      "swift-private-cloud": {
        "account": {
          "config": {
            "DEFAULT": {
              "workers": 12
            }
          }
        }
      }
      ... other config values omitted ...
    }

Which will leave the default configuration intact, with the exception
of the single override on default workers.

In a like manner, the following configuration files can be overriden
at the following environment locations:

| Config File | Attribute Location |
|-------------|--------------------|
| /etc/swift/object-server.conf | node['swift-private-cloud']['object']['config'] |
| /etc/swift/container-server.conf | node['swift-private-cloud']['container']['config'] |
| /etc/swift/account-server.conf | node['swift-private-cloud']['account']['config'] |
| /etc/swift/proxy-server.conf | node['swift-private-cloud']['proxy']['config'] |
| /etc/swift/object-expirer.conf | node['swift-private-cloud']['object-expirer']['config'] |

## Adding Static Web Middleware ##

So, given the configuration information above, the task of adding the
staticweb middleware becomes a problem of defining the config file
overrides necessary to cause the correct config to be dropped on the
proxy servers.

Since this is a modification to proxy-server.conf, we must modify the
json blob in `node['swift-private-cloud']['proxy']['config']`. We need
to add a section called `"filter:staticweb"` with the key/value pair
(`"use"` `"egg:swift#staticweb"`).

This can be accomplished with an override attribute in the environment:

    "override_attributes": {
      "swift-private-cloud": {
        "proxy": {
          "config": {
            "filter:staticweb": {
              "use": "egg:swift#staticweb"
            }
          }
        }
      }
    }

In addition, the pipeline must be changed to add the staticweb
middleware. The documentation is a little bit off, as what we really
want to do is add the staticweb middleware into the existing proxy
pipeline. Changing the pipeline to what is documented in the OpenStack
docs would result in losing a number of important pipeline
middleware. Looking at the [default config
values](http://github.com/rcbops-cookbooks/swift-private-cloud/blob/master/attributes/default.rb#L152)
for the proxy server, we can see that the default pipeline
(`node['swift-private-cloud']['proxy']['config']['pipeline:main']['pipeline']`)
is `"catch_errors proxy-logging healthcheck cache ratelimit authtoken
keystoneauth proxy-logging proxy-server"`.

We'll just add the staticweb module right before proxy-server, as it
was in the documentation:

    "override_attributes": {
      "swift-private-cloud": {
        "proxy": {
          "config": {
            "pipeline:main": {
              "pipeline": "catch_errors proxy-logging healthcheck cache ratelimit authtoken keystoneauth proxy-logging staticweb proxy-server"
            }
          }
        }
      }
    }

The combined environment might look something like this:

    "override_attributes": {
      "swift-private-cloud": {
        "proxy": {
          "config": {
            "filter:staticweb": {
              "use": "egg:swift#staticweb"
            },
            "pipeline:main": {
              "pipeline": "catch_errors proxy-logging healthcheck cache ratelimit authtoken keystoneauth proxy-logging staticweb proxy-server"
            }
          }
        }
      }
    }

## Try It ##

1.  Edit the environment to add the environment suggested above
2.  Run chef-client on the proxy
3.  Observe that the config file has been changed (`/etc/swift/proxy-server.conf`)
4.  Make a web site:
  1.  ssh into the proxy server (as root?)
  2.  `source .openrc`
  3.  `echo "<html><h1>Hi</h1></html>" > index.html`
  4.  `swift upload web index.html`
  5.  `swift post -r '.r:*' web`
  6.  `swift post -m 'web-index:index.html' web`
  7.  view at http://`<proxy-ip>`:8080/v1/`<account>`/web (account looks like `AUTH_<hex digits>`, and can be found with `swift stat`)

## Summary ##

Customizing your swifts.  YOU CAN DO IT!
