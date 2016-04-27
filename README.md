# Cookbook Promotion Across Environments

Creating Chef Environments to match your Infrastructure Environments allows
you to speak the same language with other groups within your organization and
lessens the overhead of managing constraints by keeping the number of
Chef Environments to a bare minimum.

A typical organization might have Infrastructure Environments like:

- Dev
- Staging
- Prod

A Chef Environment can be represented via json like this:
```
{
    "name": "Staging",
    "description": "This is the Staging pre-production environment.",
    "cookbook_versions": {
        "acme-finance-web": "= 1.0.2",
        "acme-marketing-web": "= 0.1.2",
        "acme-sales-pos": "= 0.1.1"
    },
    "json_class": "Chef::Environment",
    "chef_type": "environment",
    "default_attributes": {
    },
    "override_attributes": {
    }
}
```

As you can see above Chef allows you to constrain cookbook versions in an
Environment.  Any nodes in that Environment will only be able to run these
cookbook versions.

For more on Versioning see: http://semver.org

## Opt-in Will Scale Easier
It may be tempting to use the Chef Environment to constrain every cookbook
that exists.  However, this will lead to version dependency nightmares.

For example, if Team A has Cookbook "team-a", and Team B has Cookbook "team-b"
and both of them depend upon a shared Library Cookbook: "library-z" which happens
to be constrained in an Environment with an equality constraint; any new Release
of "library-z" will need to be coordinated across both Team A and Team B.  This
strategy will simply not scale.

A much easier approach to managing this is to allow for a "Opt-in" style where
Team A and Team B are able to move to newer versions of "library-z" only when
they are ready and after thorough testing.

This requires an approach of only managing version constraints for individual
Team cookbooks in Chef Environments by carving out a uniquely named "Role" for
them, version Pinning that "Role Cookbook" in the Environment and allowing the
teams to manage all dependencies within their Role Cookbook. This allows Teams
more autonomy to move at varying speeds promoting artifacts across Chef
Environments without fear of a suprise unresolvable dependency constraint
imposed by another Team or Group when they promote into a new Environment.
It significantly lessons the burdens on the Chef Administrators / DevOps
champions who do NOT have "police" cookbook promotion, allowing the whole 
Organization to begin autonomously developing with Chef.

An organization's pipeline should be able to "promote" these Role Cookbooks from
one environment to the next after each dependent component passes comprehensive
automated unit, integration, regression, smoke and acceptance/functional tests
of the whole stack.  Promotion entails the Pipeline adding or updating the Chef
Environment constraint for the Role Cookbooks in an automated fashion.  Perhaps
using a tool like Knife Spork: https://github.com/jonlives/knife-spork

Individual users should not have the permissions to manipulate Chef Environments
directly. Rather, changes should be initiated through version control and pushed
to the Chef Server via automation in the Pipeline.

To implement this, it is important to understand the Cookbook types.

# Cookbook Archetypes
Let's look at the Cookbook types that fundamentally enable this to happen safely
and with ease.

## The Library Cookbook

Most basic build block. These are meant to be reusable and depended upon by other
Cookbooks to provide functionality.

Library Cookbooks provide things like:
 - Adding LWRPs that abstract common functionality
 - Including Libraries that add Ruby modules/classes for any depending cookbooks

Generally these do not have attributes since there is nothing to configure
and often do not include any recipes.

Library cookbooks may depend on other library cookbooks or application cookbooks.
They never depend on a Role Cookbook and they never depend on a Wrapper cookbook.

An example: https://github.com/chef-cookbooks/chef-ingredient

## The Application Cookbook

These cookbooks are a level above Library cookbooks. They always contain at least
one recipe (the default recipe) to install a particular piece of software sharing
the same name as the cookbook. If the application the cookbook manages contains
multiple components then each one is broken up into it's own recipe and the recipe
is named after the component it will install. Things are broken up in this way so
you could install various components spread across a number of nodes within an
environment.

These cookbooks almost always contain a set of attributes which act as the runtime
configuration for the cookbook. These attributes can do something like setting a
port number or even describing the desired state of a service.

These cookbooks are always named after the application they manage.

Application cookbooks may depend on Library Cookbooks and other Application
Cookbooks. They never depend on Role Cookbooks. They never depend on a Wrapper
or Base Cookbook unless they are intended to be internal to your organization
and will never be distributed to the Chef Community Site.

Every Application cookbook should live in it's own version control repository.

## The Wrapper Cookbook

This is the lightest Cookbook out of all the known Cookbook patterns. It does a
very simple job of depending on an Application Cookbook and then exposing a
recipe for each recipe found in the Application Cookbook that it is wrapping. In
these recipes a single call to include_recipe "{wrapped-cookbook}::{wrapped-recipe}"
will be found along with a number of node.set[] functions which override the
default values of the wrapped Cookbook.

Wrapper cookbooks depend on Application Cookbooks only. They do not depend on
other Wrapper Cookbooks, Library Cookbooks, or Role Cookbooks.

These cookbooks follow the naming convention {organization}-{wrapped_cookbook}
or even sometimes {application}-{wrapped_cookbook}.

Every Wrapper cookbook should live in it's own version control repository.

## The Base Cookbook

Each organization should have one of these. This cookbook does the job of setting
the MOTD on your machines or creating users and setting them up with zsh instead
of bash. This cookbook can become a sort of "junk drawer" so you should be
careful when adding to it. Things should only be added here when it doesn't make
sense to place them in another spot.

This is another cookbook pattern that you don't see out in the wild because it's
specific to your organization and shouldn't be shared with anyone else.

Base Cookbooks may depend on Library Cookbooks, Application Cookbooks, or Wrapper
Cookbooks. They never depend on an Role Cookbook.

These cookbooks follow the naming convention {organization}-base.
Every Base cookbook should live in it's own version control repository.

Example metatdata.rb of a Base Cookbook:
```
name             'acme-base-linux'
maintainer       'Acme Co., Inc'
maintainer_email 'devops@acme.com'
license          '# Copyright (c) 2016 Acme Co., Inc, All Rights Reserved.'
description      'Installs/Configures acme-base-linux'
long_description IO.read(File.join(File.dirname(__FILE__), 'README.md'))

version          '2.0.4'

source_url 'https://github.com/acme/acme-base-linux'
issues_url 'https://github.com/acme/acme-base-linux/issues'

depends 'chef-client', '= 4.3.3'
depends 'chef-handler-profiler', '= 1.0.1'
depends 'chef-splunk', '= 1.8.0'
depends 'datadog', '= 2.1.0'
depends 'datadog_support', '= 2.6.0'
depends 'firewall', '= 2.3.0'
depends 'ntp', '= 1.7.0'
depends 'openssh', '= 1.5.2'
depends 'opsmatic', '= 0.1.26'
depends 'opsmatic_support', '= 0.1.7'
depends 'selinux', '= 0.8.1'
depends 'splunk_support', '= 0.6.0'
depends 'sysctl', '= 0.6.4'
depends 'sysctl_support', '= 1.2.0'
depends 'tmpwatch', '= 2.0.0'
depends 'yum-gd', '= 0.7.1'
depends 'os-hardening', '= 1.2.1'
```
Example Default Recipe of a Base Cookbook:
```
raise if node['platform'] == 'windows'

include_recipe 'yum-gd' if node['platform_family'] == 'rhel'
include_recipe 'chef-handler-profiler'
include_recipe 'chef-client::config'
include_recipe 'chef-client::service' if node['chef_client']['service']['enabled'] == true
include_recipe 'chef-client::delete_validation'

if node['acme-selinux']['enabled'] == false
  include_recipe 'selinux::permissive'
else
  include_recipe 'selinux::enforcing'
end
...
```
## The Role Cookbook
This is the piece that ties the release process of your development cycle
together and allows you to release software that is easy to install and to configure.

A Role Cookbook should not contain any logic of it's own - it should simply
include_recipe on the Application Cookbooks necessary.  One exception is that it
can include logic about which nodes should get which services, and any
orchestration that has to happen.  The version of the Role Cookbook will be
pinned in the Chef Environment.

Again, the Role Cookbook has metadata.rb depends entries (with version pins) for
each cookbook that you will want to converge on the node and will have
include_recipes entries in the default recipe for each dependent recipe component
required.

Be sure to include the "base" cookbook if you have one.

Example metatdata.rb of a Role Cookbook:
```
name             'acme-finance-web'
maintainer       'Acme Finance'
maintainer_email 'finance-devs@acme.com'
license          'All rights reserved'
description      'Role cookbook for finance-web application'
long_description IO.read(File.join(File.dirname(__FILE__), 'README.md'))
version          '1.0.2'

depends 'acme-base-linux', '~> 1.1.0'
depends 'finance-cookbook', '= 2.3.1'
```
Example Default Recipe of the Role Cookbook:
```
include_recipe 'acme-base-linux'
include_recipe 'finance-application::keys'
include_recipe 'finance-application::web'
```

Sometimes a Chef Role may be created to house the Role Cookbook.  However, it
should NOT contain any attribute overrides and anything other than one
run_list entry: the Role Cookbook.  This is important because Chef Roles are not
versioned.  Changing a Chef Role immediately affects ALL nodes across all
environments that have the Role applied. Instead, allow the Role Cookbook to version
the run_list and dependency pins and perhaps override attributes.

Example Chef Role:
```
{
  "name": "acme-finance-web",
  "description": "Installs and configures all components for a Finance Web server.",
  "json_class": "Chef::Role",
  "default_attributes": {
  },
  "override_attributes": {
  },
  "chef_type": "role",
  "run_list": [
    "recipe[acme-finance-web]"
  ],
  "env_run_lists": {
  }
}
```

Finally, the Chef Environment Files with Role Cookbook version pins:
```
{
    "name": "Dev",
    "description": "This is the Dev.  The latest and greatest versions.",
    "cookbook_versions": {
    },
    "json_class": "Chef::Environment",
    "chef_type": "environment",
    "default_attributes": {
    },
    "override_attributes": {
    }
}
```
```
{
    "name": "Staging",
    "description": "This is the Staging pre-production environment.",
    "cookbook_versions": {
        "acme-finance-web": "= 1.4.1",
        "acme-marketing-web": "= 1.1.8",
        "acme-sales-pos": "= 0.10.9"
    },
    "json_class": "Chef::Environment",
    "chef_type": "environment",
    "default_attributes": {
    },
    "override_attributes": {
    }
}
```
```
{
    "name": "Prod",
    "description": "This is the Production environment.",
    "cookbook_versions": {
        "acme-finance-web": "= 1.0.1",
        "acme-marketing-web": "= 0.1.0",
        "acme-sales-pos": "= 0.1.0"
    },
    "json_class": "Chef::Environment",
    "chef_type": "environment",
    "default_attributes": {
    },
    "override_attributes": {
    }
}
```

Notes: In other articles and discussion circles you may sometimes hear the words "Role Cookbooks"
and "Environment Cookbooks" used interchangeably.

Much of this was cribbed from:
- http://blog.vialstudios.com/the-environment-cookbook-pattern/
- https://www.chef.io/blog/2013/11/19/chef-roles-arent-evil/
