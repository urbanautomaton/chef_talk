# The Reluctant Chef

somehting something something

## Who am I?

![](images/hat.jpg)

## Who am I?

![](images/future.jpg)

## Who am I?

![](images/desk.jpg)

## What's this about?

![](images/ops1.jpg)

## What's this about?

![](images/ops2.jpg)

## What's this about?

* Noddy guide to Chef
* Introducing config management **gradually**
* Tools/advice to **simplify the process**
* Developers! Developers! Developers!

## Why should I care?

* Our code is only meaningful when **embodied**
* Our **dependencies** alter the meaning of our code
* Our responsibility is for **production** performance
* "Works on my machine" just doesn't cut it!

## This sounds boring!

* Ops **can** be like programming
* Playing with VMs is **really cool**
* Maybe we can set up our laptops? 

## This sounds boring!

![](images/vm_tweet.png)

## Chef overview

Okay, so what does a Chef run actually do?

1. Examine current configuration
2. Construct desired configuration
3. Converge!

## Chef overview

Components:

* **Ohai** - examines existing configuration
* **Knife** - manages server objects: cookbooks, nodes, data bags
* **Chef Server** - aggregates node information, dishes out cookbooks
* **Chef Client** - receives info. from server, applies changes
* **Chef Solo** - standalone client, uses only local info

## Chef overview

![](images/chef_server.png)

## Chef overview

![](images/chef_server_shaded.png)

## Chef Solo

![](images/chef_solo.png)

## What's in a cookbook?

The gems of the Chef world:

* Attributes
* Files & Templates
* Custom resources / definitions
* Recipes

## Workstation cookbook

1. Build-essentials (gcc etc.)
2. Git
3. Package manager (homebrew)
4. Ruby environment (switcher + install manager)
5. MySQL

## Workstation cookbook

```ruby
# recipes/default.rb
include_recipe "build-essential"
include_recipe "homebrew"
include_recipe "git"
include_recipe "mysql"

package "chruby" do
  action :install
end

package "ruby-install" do
  action :install
end
```

## Workstation cookbook

Cookbook metadata - c.f. gemspec

```ruby
# metadata.rb
name             'workstation'
description      'Sets up an OSX workstation with basic web dev stack'
long_description IO.read(File.join(File.dirname(__FILE__), 'README.md'))
version          '0.1.0'

supports 'mac_os_x'

depends "build-essential",  "~> 1.4.0"
depends "git",              "~> 2.5.0"
depends "homebrew",         "~> 1.3.0"
depends "mysql",            "~> 2.1.0"
```

## Workstation cookbook

```ruby
# recipes/default.rb
include_recipe "build-essential"
include_recipe "homebrew"
include_recipe "git"
include_recipe "mysql"

package "chruby" do
  action :install
end

package "ruby-install" do
  action :install
end
```

## Workstation cookbook

We want to customise some MySQL configuration details:

```ruby
# attributes/default.rb
normal['mysql']['tunable']['character-set-server'] = 'utf8mb4'
normal['mysql']['tunable']['collation-server']     = 'utf8mb4_col'
normal['mysql']['tunable']['log_bin']              = false
normal['mysql']['tunable']['sql_mode']             = 'STRICT_TRANS_TABLES'
```

## Workstation cookbook

```ruby
# recipes/default.rb
include_recipe "build-essential"
include_recipe "homebrew"
include_recipe "git"
include_recipe "mysql"

package "chruby" do
  action :install
end

package "ruby-install" do
  action :install
end
```

## Chef Resources

Chef's main unit of abstraction

* **file** - creates a file
* **template** - creates a file with interpolated values
* **package** - installs a software package
* **erl_call** - a connection to a node within a distributed Erlang system

## Creating a lightweight resource

Let's create a resource that installs a user's dotfiles. It'll be used
like this:

```ruby
workstation_dotfiles 'https://github.com/urbanautomaton/dotfiles' do
  branch    "master"
  installer "install.sh"
  action    :install
end
```

## Creating a lightweight resource

```ruby
# resources/dotfiles.rb

actions :install

default_action :install

attribute :repo,      :kind_of => String, :name_attribute => true
attribute :branch,    :kind_of => String, :default => "master"
attribute :installer, :kind_of => String, :default => "install.sh"

attr_accessor :exists
```

## Creating a lightweight resource

Then we define its (sole) action:

```ruby
# providers/dotfiles.rb

action :install do
  if !repo_exists?
    clone_repo!
  else
    update_repo!
  end

  if dotfiles_installed?
    Chef::Log.info("Dotfiles up to date - skipping install.")
  else
    run_installer!
  end
end
```

## Creating a lightweight resource

And finally, the helper methods:

```ruby
# providers/repo.rb

def clone_repo
  execute "Clone dotfiles repo" do
    command <<-EOS
      cd ~ && \
      git clone #{new_resource.repo} dotfiles && \
      cd dotfiles && \
      git reset --hard origin/#{new_resource.branch}
    EOS
  end
end
```

## Creating a lightweight resource

And finally, the helper methods:

```ruby
# providers/repo.rb

def update_repo
  execute "Update dotfiles repo" do
    command <<-EOS
      cd ~/dotfiles && \
      git fetch origin && \
      git checkout #{new_resource.branch} && \
      git reset --hard origin/#{new_resource.branch}
    EOS
  end
end
```

## Creating a lightweight resource

And finally, the helper methods:

```ruby
# providers/repo.rb

def run_installer
  execute "Install dotfiles" do
    command <<-EOS
      cd ~/dotfiles && \
      ./#{new_resource.installer} && \
      git rev-parse HEAD > .installed_version
    EOS
  end
end
```

## Creating a lightweight resource

And finally, the helper methods:

```ruby
# providers/repo.rb

def repo_exists?
  ::File.exists?(::File.expand_path("~/dotfiles"))
end

def dotfiles_installed?
  installed_cmd = <<-EOS
    cd ~/dotfiles && \
    if [ -f ".installed_version" ]; then
      cat .installed_version;
    fi
  EOS
  latest_cmd = "cd ~/dotfiles && git rev-parse HEAD"
  `#{installed_cmd}`.strip == `#{latest_cmd}`.strip
end
```

## Back to our cookbook

```ruby
#
# Cookbook Name:: workstation
# Recipe:: default

include_recipe "build-essential"
include_recipe "homebrew"
include_recipe "git"
include_recipe "mysql"

package "chruby"
package "ruby-install"

dotfiles "https://github.com/urbanautomaton/dotfiles"
```

## From cookbooks to kitchen

Okay, so how do we apply this?

* What recipes get run?
* Where do our cookbooks live?
* Where do our dependencies come from?

## The kitchen repo

```
Gemfile
Berksfile
.chef/
  - knife.rb
  - solo.rb
certificates/
cookbooks/
  - workstation/
data_bags/
nodes/
  - workstation.json
roles/
vendor/
  - cookbooks/
    - git/
    - mysql/
    - # etc.
```

## The node definition

Primary input to Chef Solo

```javascript
// nodes/workstation.json
{
  "name": "workstation",
  "run_list": [
    "recipe[workstation]"
  ],
  "attributes" {
    "mysql": {
      "tunable": {
        "innodb_buffer_pool_size": "512M"
      }
    }
  }
}
```

## Dependency management

## Dependency management

Before:

```
$ knife cookbook site install <cookbook> [<version>]
```

[unspeakable git violence ensues]

## Dependency management

```
* 7f7f786 - (cookbook-site-imported-database-1.3.12, chef-vendor-database)
*   e099780 - Merge branch 'chef-vendor-xfs'
|\
| * 2d13d2c - (cookbook-site-imported-xfs-1.1.0, chef-vendor-xfs)
* |   bce817e - Merge branch 'chef-vendor-aws'
|\ \
| * | 4bde629 - (cookbook-site-imported-aws-0.100.6, chef-vendor-aws)
* | |   99a07a1 - Merge branch 'chef-vendor-apt'
|\ \ \
| * | | fc650a9 - (cookbook-site-imported-apt-1.9.2, chef-vendor-apt)
* | | |   8865913 - Merge branch 'chef-vendor-postgresql'
|\ \ \ \
| * | | | d782483 - (cookbook-site-imported-postgresql-2.4.0, chef-vendor-postgresql)
```

## Dependency management

After: Berkshelf / Librarian

```ruby
# Berksfile
site :opscode

cookbook "build-essential"
cookbook "git"
cookbook "homebrew"
cookbook "mysql"
```

```
$ berks install --path=vendor/cookbooks
```

## Ready to go!

* Get your shiny new Macbook
* Set the correct permissions on `/usr/local/`
* Clone your kitchen repo with git
* Bundle install the required gems
* Uh...
* Bugger.

## The bootstrapping problem

* Chef is a gem
* It requires ruby
* Its dependencies require a build toolchain
* So... what's the point?

## Bait and switch!

* Cost vs. benefit
* How often do you provision your laptop?
* Reusability
* 14 lines of bash?

## Colours to the mast

* Chef will not make setting up your laptop fun
* Chef will not eliminate the germs that may cause bad breath
* Chef **will** put you in the driver's seat

## Why should we REALLY care?

* Improve your development tools
* Stop using development mode!


