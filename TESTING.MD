Cookbook TESTING doc
====================

**Note**: This document outlines the process for testing that we are working towards for all chef-cookbooks repos.  Not all cookbooks currently ship with the necessary testing framework. This document does not apply to any repository that is missing the Rakefile. We appreciate the patience as we roll out first class testing to all our repositories.

Testing Prerequisites
---------------------
Chef cookbooks require either a working ChefDK installation set as your system's default ruby or Ruby 2.0+ with bundler installed. Using ChefDK provides a consistent Ruby install, and is the suggested method. ChefDK can be downloaded at https://downloads.chef.io/chef-dk/

To ensure all gems are updated to their latest releases run `bundle install; bundle update` before running any Rake testing tasks.

Integration testing relies on both Hashicorp's [Vagrant](https://www.vagrantup.com/downloads.html) and Oracle's [Virtualbox](https://www.virtualbox.org/wiki/Downloads), which must be installed first.

Rakefile
--------
The Rakefile ships with a number of tasks, each of which can be ran
individually, or in groups. Typing "rake" by itself will perform style
checks with Rubocop and Foodcritic, ChefSpec with rspec, and
integration with Test Kitchen using the Vagrant driver by
default.Alternatively, integration tests can be ran with Test Kitchen
cloud drivers.

```
$ rake -T
rake integration:cloud        # Run Test Kitchen with cloud plugins
rake integration:vagrant      # Run Test Kitchen with Vagrant
rake maintainers:generate     # Generate MarkDown version of MAINTAINERS file
rake spec                     # Run ChefSpec examples
rake style                    # Run all style checks
rake style:chef               # Run Chef style checks
rake style:ruby               # Run Ruby style checks
rake style:ruby:auto_correct  # Auto-correct RuboCop offenses
rake travis                   # Run all tests on Travis
```

Style Testing
-------------
Ruby style tests can be performed by Rubocop by issuing either
```
bundle exec rubocop
```
or
```
rake style:ruby
```

Chef style/correctness tests can be performed with Foodcritic by issuing either
```
bundle exec foodcritic
```
or
```
rake style:chef
```

Spec Testing
-------------
Unit testing is done by running Rspec examples. Rspec will test any
libraries, then test recipes using ChefSpec. This works by compiling a
recipe (but not converging it), and allowing the user to make
assertions about the resource_collection.

Integration Testing
-------------------
Integration testing is performed by Test Kitchen. Test Kitchen will
use either the Vagrant driver or various cloud drivers to instantiate
machines and apply cookbooks. After a successful converge, tests are
uploaded and ran out of band of Chef. Tests should be designed to
ensure that a recipe has accomplished its goal.

Integration Testing using Vagrant
---------------------------------
Integration tests can be performed on a local workstation using
Virtualbox or VMWare. Detailed instructions for setting this up can be
found at the [Bento](https://github.com/chef/bento) project web site.

Integration tests using Vagrant can be performed with either
```
bundle exec kitchen test
```
or
```
rake integration:vagrant
```

Integration Testing using Cloud providers
-----------------------------------------
Integration tests can be performed on cloud providers using
Test Kitchen plugins. If the cookbook ships with a ```.kitchen.cloud.yml```
that references environmental variables present in the shell that
```kitchen test``` is ran from. These usually contain authentication
tokens for driving IaaS APIs, as well as the paths to ssh private keys
needed for Test Kitchen log into them after they've been created.

Examples of environment variables being set in ```~/.bash_profile```:
```
# digital_ocean
export DIGITAL_OCEAN_CLIENT_ID='your_bits_here'
export DIGITAL_OCEAN_API_KEY='your_bits_here'
export DIGITAL_OCEAN_SSH_KEY_IDS='your_bits_here'

# aws
export AWS_ACCESS_KEY_ID='your_bits_here'
export AWS_SECRET_ACCESS_KEY='your_bits_here'
export AWS_KEYPAIR_NAME='your_bits_here'

# joyent
export SDC_CLI_ACCOUNT='your_bits_here'
export SDC_CLI_IDENTITY='your_bits_here'
export SDC_CLI_KEY_ID='your_bits_here'
```

Integration tests using cloud drivers can be performed with either
```
export KITCHEN_YAML=.kitchen.cloud.yml
bundle exec kitchen test
```
or
```
rake integration:cloud
```

Digital Ocean Hint
------------------
At the time of this writing, you cannot find the numerical values
needed for your SSH_KEY_IDS from the GUI. Instead, you will need to
access the API from the command line.

    curl -L 'https://api.digitalocean.com/ssh_keys/?client_id=your_bits_here&api_key=your_bits_here'

Words about .travis.yml
-----------------------
In order for Travis to perform integration tests on public cloud
providers, two major things need to happen. First, the environment
variables referenced by ```.kitchen.cloud.yml``` need to be made
available. Second, the private half of the ssh keys needed to log into
machines need to be dropped off on the machine.

The first part is straight forward. The travis gem can encrypt
environment variables against the public key on the Travis repository
and add them to the .travis.yml.

```
gem install travis
travis encrypt AWS_ACCESS_KEY_ID='your_bits_here' --add
travis encrypt AWS_SECRET_ACCESS_'your_bits_here' --add
travis encrypt AWS_KEYPAIR_NAME='your_bits_here' --add
travis encrypt EC2_SSH_KEY_PATH='~/.ssh/id_ec2.pem' --add

travis encrypt DIGITAL_OCEAN_CLIENT_ID='your_bits_here' --add
travis encrypt DIGITAL_OCEAN_API_KEY='your_bits_here' --add
travis encrypt DIGITAL_OCEAN_SSH_KEY_IDS='your_bits_here' --add
travis encrypt DIGITAL_OCEAN_SSH_KEY_PATH='~/.ssh/id_do.pem' --add
```

The second part is a little more complicated. Travis ENV variables are
restricted to 90 bytes, and will not fit an entire SSH key. This can
be worked around by breaking them up into 90 byte chunks, stashing
them into ENV variables, then digging them out in the
```before_install``` section of .travis.yml

Here is an AWK script to do the encoding.
```
base64 ~/.ssh/travisci_cook_digitalocean.pem | \
awk '{
  j=0;
  for( i=1; i<length; i=i+90 ) {
    system("travis encrypt DO_KEY_CHUNK_" j "=" substr($0, i, 90) " --add");
    j++;
  }
}'

base64 ~/.ssh/travisci_cook_ec2.pem | \
awk '{
  j=0;
  for( i=1; i<length; i=i+90 ) {
    system("travis encrypt EC2_KEY_CHUNK_" j "=" substr($0, i, 90)" --add");
    j++;
  }
}'
```

Then in .travis.yml:
```
before_install:
- echo -n $DO_KEY_CHUNK_{0..30} >> ~/.ssh/id_do.base64
- cat ~/.ssh/id_do.base64 | tr -d ' ' | base64 --decode >  ~/.ssh/id_do.pem
  - echo -n $EC2_KEY_CHUNK_{0..30} >> ~/.ssh/id_ec2.base64
  - cat ~/.ssh/id_ec2.base64 | tr -d ' ' | base64 --decode > ~/.ssh/id_ec2.pem
```

