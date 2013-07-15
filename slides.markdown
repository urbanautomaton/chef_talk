# The Reluctant Chef

## Who am I?

![](images/hat.jpg)

## Who am I?

![](images/desk.jpg)

## What's this about?

![](images/ops1.jpg)

## What's this about?

![](images/ops2.jpg)

## What's this about?

![](images/future.jpg)

## What's this about?

* Introducing config management **gradually**
* Benefits to small teams
* Developers! Developers! Developers!

## Why should I care?

* Our responsibility is for **production** behaviour
* "Works on my machine" just doesn't cut it!
* Our **dependencies** alter the meaning of our code

## Why should I care?

How is your database configured?

```
# /etc/my.cnf

speak_swedish             = no
silently_truncate_strings = no
```

## This sounds boring!

* Ops **can** be like programming
* Playing with VMs is **really cool**
* Maybe we can set up our laptops? 

## Chef overview

Okay, so what does a Chef run actually do?

1. Examine current configuration
2. Construct desired configuration
3. Converge!

## Chef overview

Components:

* **Ohai** - examines existing configuration
* **Chef Server** - aggregates node info, dishes out cookbooks, roles
* **Knife** - manages server objects: cookbooks, nodes, data bags
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
* Libraries
* Recipes

## Workstation cookbook

Let's set up an OSX laptop:

1. Build-essentials
2. Git
3. Package manager
4. Ruby environment
5. MySQL

## Workstation cookbook

```
README.md
attributes/
  - default.rb
metadata.rb
recipes/
  - default.rb
```

## Workstation cookbook

```ruby
# recipes/default.rb
include_recipe "build-essential"
include_recipe "homebrew"
include_recipe "git"
include_recipe "mysql::client"
include_recipe "mysql::server"

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
include_recipe "mysql::client"
include_recipe "mysql::server"

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
# speak_swedish = no
normal['mysql']['tunable']['character-set-server'] = 'utf8mb4'
normal['mysql']['tunable']['collation-server']     = 'utf8mb4_col'
# silently_truncate_strings = no
normal['mysql']['tunable']['sql_mode']             = 'STRICT_TRANS_TABLES'
```

## Workstation cookbook

```ruby
# my.cnf.erb
character-set-server = <%= node['mysql']['character-set-server'] %>
pid-file             = <%= node['mysql']['pid_file'] %>
basedir              = <%= node['mysql']['basedir'] %>
datadir              = <%= node['mysql']['data_dir'] %>
```

## Workstation cookbook

```ruby
# recipes/default.rb
include_recipe "build-essential"
include_recipe "homebrew"
include_recipe "git"
include_recipe "mysql::client"
include_recipe "mysql::server"

package "chruby" do
  action :install
end

package "ruby-install" do
  action :install
end
```

## Chef Resources

Chef's basic unit of abstraction

* **file** - creates a file
* **template** - creates a file with interpolated values
* **package** - installs a software package
* **erl_call** - a connection to a node within a distributed Erlang system
* and many more...

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
```

## Creating a lightweight resource

Then we define its (sole) action:

```ruby
# providers/dotfiles.rb

action :install do
  clone_repo! unless repo_exists?
  update_repo!

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
    cwd "~"
    command "git clone #{new_resource.repo} dotfiles"
  end
end
```

## Creating a lightweight resource

And finally, the helper methods:

```ruby
# providers/repo.rb

def update_repo
  execute "Update dotfiles repo" do
    cwd "~/dotfiles"
    command <<-EOS
      git fetch origin && \
      git checkout -B deploy && \
      git reset --hard origin/#{new_resource.branch}
    EOS
  end
end
```

## Creating a lightweight resource

And finally, the helper methods:

```ruby
# providers/repo.rb

def run_installer!
  execute "Install dotfiles" do
    cwd "~/dotfiles"
    command <<-EOS
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

dotfiles node['workstation']['dotfiles_repo'] do
  action :install
end
```

## From cookbooks to kitchen

Okay, so how do we apply this?

* What recipes get run?
* Where do our cookbooks live?
* Where do our dependencies come from?
* Where do we set machine-specific attributes?

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
roles/
vendor/
  - cookbooks/
```

## The node definition

Primary input to Chef Solo

```javascript
// nodes/workstation.json
{
  "name": "workstation",
  "run_list": [
    "recipe[workstation]"
  ]
}
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
  "attributes": {
    "workstation: {
      "dotfiles_repo": "https://github.com/urbanautomaton/dotfiles"
    }
  }
}
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
  "attributes": {
    "workstation: {
      "dotfiles_repo": "https://github.com/urbanautomaton/dotfiles"
    },
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

## The Chef Run

```
$ chef-solo -c .chef/solo.rb -j nodes/workstation.json
```

## The Chef Run

```ruby
[
  "workstation"
]
```

## The Chef Run

```ruby
{
  "workstation" => [
      "build-essentials",
      "homebrew",
      "git",
      "mysql",
    ]
}
```

## The Chef Run

```ruby
{
  "workstation" => {
      "build-essentials" => [...],
      "homebrew"         => [...],
      "git"              => [...],
      "mysql"            => [...],
    }
}
```

## Ready to go!

* Get your shiny new Macbook
* Set the correct permissions on `/usr/local/`
* Clone your kitchen repo with git
* Bundle install the required gems
* Install a build toolchain so the gems can install
* Uh...
* Bugger.

## The bootstrapping problem

* Chef is a gem
* It requires ruby
* Its dependencies require a build toolchain
* Your cookbooks **might** work with OSX system ruby
* So... what's the point?

## Bait and switch!

* Cost vs. benefit
* How often do you provision your laptop?
* Reusability
* 14 lines of bash?

## Colours to the mast

Chef will not make setting up your laptop fun

## Why should we REALLY care?

* Improve your development tools
* Work with your **real** dependencies
* Stop using development mode!

## Why should we REALLY care?

* Ruby deployment is ghastly
* Abstract the setup of an app

## Why should we REALLY care?

Imagine if we could do this:

```ruby
rails_app "tribesports" do
  hostname "tribesports.com"
  action :create
end
```

## Why should we REALLY care?

Or this:

```
$ vagrant up

$ cap vagrant deploy
```

## Working with Chef

Iterate quickly - virtual machines

## Virtual Machines

![](images/vm_tweet.png)

## Vagrant

![](images/mitchellh.jpg)

## Vagrant

```ruby
# Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box     = "precise64"
  config.vm.box_url = "http://files.vagrantup.com/precise64.box"

  config.vm.network :private_network, :ip => 10.0.0.10

  config.vm.provision "chef_solo" do |chef|
    chef.cookbooks_path = ["cookbooks", "vendor/cookbooks"]
    chef.add_role "database"
    chef.add_role "redis"
    chef.add_role "web"
  end
end
```

## Vagrant

```ruby
# Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box     = "precise64"
  config.vm.box_url = "http://files.vagrantup.com/precise64.box"

  config.vm.network :private_network, :ip => 10.0.0.10

  config.vm.provision "shell" do |s|
    s.path "my_provisioning_script.sh"
  end
end
```

## Working with Vagrant

* Pick a platform
* Build your own binary packages
* Package your own boxes
* Cache packages with vagrant-cachier

## Packaging boxes

1. Take a base box
2. Configure it to a baseline
3. `vagrant package <vm_name>`

## Packaging boxes

![](images/mitchellh.jpg)

## Packaging boxes

packer.io

> Packer is a tool for creating identical machine images for multiple
> platforms from a single source configuration.

## Building binary packages

![](images/jordan_sissel.png)

## Building binary packages

fpm - "effing package management"

> It helps you build packages quickly and easily (Packages like RPM and
> DEB formats).

> Fundamental principle: if fpm is not helping you make packages easily,
> then there is a bug in fpm.

## Building binary packages

1. Script containing your usual compilation steps
2. `$ make install DESTDIR=installdir`
3. `$ fpm -s dir -t deb -n <package_name> -C installdir`

## Building binary packages

Examples at:

* https://github.com/tribesports/packages
* https://github.com/alphagov/packages

## Chef in small steps

One step at a time:

* Configure your user accounts
* Configure sshd and other security
* Install apache
* Set up your deploy directory
* etc. and so forth

## Chef in small steps

Chef Solo gets you a **very** long way

* No live remote node info
* No environments

## Chef in small steps

Provisioning multi-box setups with Chef Solo

```
nodes/
  - db_master.json
  - db_slave.json
  - web1.json
  - web2.json
  - worker.json
```

## Chef in small steps

https://github.com/edelight/chef-solo-search

```
# recipes/db_master.rb
search(:node, "role:web") do |n|
  ip = n['ip_address']
  # poke hole in firewall
  # add database account
end
```

## Working with Chef

## Working with Chef

Is it really programming?

* Honking great global attributes hash
* Spooky action at a distance
* Feels like scripting

## Working with Chef

Increasing use of abstraction

* Resources / definitions
* Client code more explicit
* Reduced coupling
* More composable, extendable

## Working with Chef

Community cookbooks

* Quality varies
* Contributing is a pain
* But so is forking
* Gems like chef-rewind allow tweaking

## Wrapping up

Pros:

* Repeatability
* Testability
* Abstraction / reduce cognitive load
* Control of dependencies
* Better development tools

## Wrapping up

Cons:

* Learning curve
* Bootstrapping can be a pain
* External dependencies - trust the cookbooks?

## Thank you!

Questions?
