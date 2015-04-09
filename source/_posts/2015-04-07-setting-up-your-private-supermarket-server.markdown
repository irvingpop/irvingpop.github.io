---
layout: post
title: "Setting Up Your Private Supermarket Server"
date: 2015-04-07T15:48:34-07:00
categories: chef supermarket
---

*Note: this is an updated version of the previous post from August, 2014: [Getting started with oc-id and Supermarket](https://www.chef.io/blog/2014/08/29/getting-started-with-oc-id-and-supermarket/)*

Chef Server 12 includes [oc-id](https://github.com/chef/oc-id), the OAuth2 service that powers [id.chef.io](https://id.chef.io/).  After upgrading to this release, Chef customers can now run their own Supermarket service behind a firewall.

## oc-id setup on your Chef Server:
*NOTE:*

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

## Testing your Private Supermarket server in Test Kitchen
*Note: We will not use the community Supermarket cookbook, because at this time it installs Supermarket from source.  Instead, we will follow the newer install method which uses an Omnibus package*

1. s

