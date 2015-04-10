---
layout: post
title: "Setting Up Your Private Supermarket Server"
date: 2015-04-07T15:48:34-07:00
categories: chef supermarket
---

*This is an updated version of the previous post from August, 2014: [Getting started with oc-id and Supermarket](https://www.chef.io/blog/2014/08/29/getting-started-with-oc-id-and-supermarket/)*

Chef Server 12 includes [oc-id](https://github.com/chef/oc-id), the OAuth2 service that powers [id.chef.io](https://id.chef.io/).  After upgrading to this release, Chef customers can now run their own Supermarket service behind a firewall.

<!-- more -->

## oc-id setup on your Chef Server:
*You must be logged in to your Chef server via ssh and elevated to an admin user level for the following steps*

1.  Add the following setting to your `/etc/opscode/chef-server.rb` configuration file:
  ```ruby
  oc_id['applications'] = {
    'supermarket' => {
      'redirect_uri' => 'https://supermarket.mycompany.com/auth/chef_oauth2/callback'
    }
  }
  ```

2. run `chef-server-ctl reconfigure`

3. After the reconfigure, you will find the OAuth2 data in `/etc/opscode/oc-id-applications/supermarket.json`
  ```json
  {
    "name": "supermarket",
    "uid": "0bad0f2eb04e935718e081fb71e3b7bb47dc3681c81acb9968a8e1e32451d08b",
    "secret": "17cf1141cc971a10ce307611beda7f4dc6633bb54f1bc98d9f9ca76b9b127879",
    "redirect_uri": "https://supermarket.mycompany.com/auth/chef_oauth2/callback"
  }
  ```

Note the `uid` and `secret` values from this file, you will need them for the next stage.

*You can add as many oc-id applications as you wish to the chef-server.rb configuration, it will create one file per application*

## Running your Private Supermarket server in Test Kitchen
*Note: We will not use the community Supermarket cookbook, because at this time it installs Supermarket from source.  Instead, we will us an Omnibus package to install*

In the spirit of "code as documentation" I've provided a simple cookbook and test-kitchen configuration for testing Supermarket Omnibus packages. These packages are downloaded from https://packagecloud.io/chef/stable

1. Download a copy of the `supermarket-omnibus-cookbook`
```bash
git clone https://github.com/irvingpop/supermarket-omnibus-cookbook.git supermarket-omnibus-cookbook
cd supermarket-omnibus-cookbook
```

2. Create a `.kitchen.local.yml` file, to set your oc-id attributes (as captured in step 3 above)
  ```yaml
  ---
  suites:
    - name: default
      run_list:
        - recipe[supermarket-omnibus-cookbook::default]
      attributes:
        supermarket_omnibus:
          chef_server_url: https://chefserver.mycompany.com
          chef_oauth2_app_id: 0bad0f2eb04e935718e081fb71e3b7bb47dc3681c81acb9968a8e1e32451d08b
          chef_oauth2_secret: 17cf1141cc971a10ce307611beda7f4dc6633bb54f1bc98d9f9ca76b9b127879
          chef_oauth2_verify_ssl: false
  ```

3. Install the `vagrant-hostsupdater` plugin, this will automatically add the names of your machines to your /etc/hosts file. This is important for oauth2, which cares about host names. The `redirect_uri` value you entered in to your oc-id configuration reflects this name.
  ```bash
  vagrant plugin install vagrant-hostsupdater
  ```

4. Start your Supermarket instance and test it
  ```bash
  kitchen converge default-centos-66 && kitchen verify default-centos-66
  ```

5. Go to your your Supermarket server and log in as a Chef user: https://default-centos-66

6. Upon login, you should see:
{% img https://www.getchef.com/blog/wp-content/uploads/2014/08/oc-id5-1024x343.png %}

## Troubleshooting & FAQ

### Incorrect redirect URL
The redirect URL specified in oc-id **MUST** match the hostname of the Supermarket server. Also, you must get the URI correct (/auth/chef_oauth2/callback). If these are not true, you will recieve an error message like:
```
The redirect uri included is not valid.
```

### Supermarket server cannot reach oc-id, throws 500 error during login
The Supermarket server must be able to reach (via https) the specified `chef_server_url` - it does this during OAuth2 negotation. The most common problems are name resolution and firewall rules.

### Where can I find the code to Supermarket?
* Supermarket the rails application is [located here](https://github.com/chef/supermarket)
  * All Supermarket [issues should be reported there](https://github.com/chef/supermarket/issues)
* The code which builds Supermarket into an Omnibus package is [located here](https://github.com/chef/omnibus-supermarket)
  * The cookbook that is run when during `supermarket-ctl reconfigure` is [located within this repo](https://github.com/chef/omnibus-supermarket/tree/master/cookbooks/omnibus-supermarket)
  * You can build your own Omnibus packages by following [the instructions in the README.md](https://github.com/chef/omnibus-supermarket#kitchen-based-build-environment)

### How do I enable rails application debug logging?
There is a known issue with the Supermarket omnibus package that rails messages are not logged. To fix that requires a manual change at the moment. On your supermarket server, edit this file: `/opt/supermarket/embedded/service/supermarket/config/environments/production.rb`, change line 46 (`config.log_level = :warn`) to look like:
```ruby
  config.logger = Logger.new('/var/log/supermarket/rails/rails.log')
  config.logger.level = 'DEBUG'
  config.log_level = :debug
```

Then restart the rails service by running
`supermarket-ctl restart rails`

### Contacting packagecloud fails if I'm behind a proxy
No problem!  Add the following to your `.kitchenl.local.yml` file:
```yaml
---
provisioner:
  name: chef_zero
  solo_rb:
    http_proxy: http://192.168.1.1
    https_proxy: http://192.168.2.2
```

### Test kitchen is slow because it has to download/install the Chef Omnibus client package every time
Here's a few tips to speed it up:

1. Tell test-kitchen to cache the Omnibus installer (put this in your `.kitchen.local.yml` file):
  ```yaml
  ---
  provisioner:
    name: chef_zero
    chef_omnibus_install_options: -d /tmp/vagrant-cache/vagrant_omnibus
  ```
2. Cache yum repos like packagecloud using the vagrant-cachier plugin.  First run `vagrant plugin install vagrant-cachier`, then create a `$VAGRANT_HOME/Vagrantfile` that looks like so:
  ```ruby
  Vagrant.configure("2") do |config|
    config.vm.box = 'some-box'
    if Vagrant.has_plugin?("vagrant-cachier")
      config.cache.scope = :box
      config.cache.enable :chef
      config.cache.enable :apt
      config.cache.enable :yum
      config.cache.enable :gem
    end
  end
  ```
