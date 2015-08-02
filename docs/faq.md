Frequently Asked Questions
==========================

### Can I create machines without provisioning them?

To create machines without provisioning, you can use the `:setup` action.  That will get chef-client installed and the client/node created, but will not actually run it.  `:allocate` goes even further: it will ask Amazon to spin up the machine but will not wait for it to be spun up (you can provision it later with Chef Provisioning still, don't worry).  `:ready` is in between and will spin up the machine but will not install chef-client on it.

If you want to create the machines without *any* provisioning at all--without even waiting for them to be provisioned--you can use the :allocate action.  All that will do is send the request to AWS to spin up the machines and then let you continue.  If you do this:

```ruby
machine 'x' do
  action :allocate
end

<do other recipe stuff>

machine 'x' do
  recipe 'blah'
end
```

Then everything will still work: the second machine declaration will "catch" the machine and provision it, so to speak.  Yay idempotency!

Even more interesting in combination with machine_batch:

```ruby
machine_batch do
  machines 'a', 'b', 'c'
  action :allocate
  # NOTE: you could have also written out each machine as a full machine delcaration here a la machine 'x' do ... end
end

... do other stuff ...

machine_batch do
  machine 'a' do
    recipe 'a'
  end
  machine 'b' do
    recipe 'b'
  end
  ...
end
```

This will get Amazon spinning up all your machines at once, at the beginning, and then your recipe can do other things ... then when you're done with other things, the second machine_batch declaration will "catch" the machines and get them provisioned.

### I need to be sure *all* my machines have registered themselves and updated before I install my actual app.  How?

A very common, primitive, and effective orchestration technique with Chef Provisioning is to fully :converge the machines, but not run any recipes (or just run some base recipes) so that the whole data center is registered with the Chef server, ohai'd, and updated before the real application install begins.  This lets you get the benefits of parallelism at the beginning--set up *all* of your servers at once--and lets you converge the machines later in whatever order you want.

In fact, often you won't have to do any ordering at all if you do this, since Chef now knows all the machines' IPs and they can connect to each other.  Typically, as long as you can put the IPs of the other machines in config files, applications will wait until the other machines spin up.

It looks something like this:

```ruby
machine_batch do
  [ 'db', 'web1', 'web2', 'web3' ].each do |name|
    machine name do
      recipe 'base' # Don't converge machine-specific stuff yet, but let's get apt updated and stuff in parallel
    end
  end
end

# Converge the database first
machine 'db' do
  recipe 'mysql'
end

# Now converge the web machines
machine_batch do
  [ 'web1', 'web2', 'web3' ].each do |name|
    machine name do
      recipe 'apache2'
    end
  end
end
```

### Where does machine information get stored?

When you first provision a machine with Provisioning (like, when you allocate it), Chef Provisioning creates a **Chef node** for the machine with a special attribute `{ "chef_provisioning": { "location": { ... }}}`. Inside there is a hash of information identifying the server, including a `driver_url`.  The driver_url is the same thing you specify in `CHEF_DRIVER` and `with_driver`.  Different drivers will store different information; the AWS driver, for instance, stores the instance ID (`i-19834b 13`).

If you are running against a Chef server, this node lives on the Chef server and anyone else with permissions can see it.  If you are running in local mode, the nodegets saved to <chef_repo_path>/nodes/node_name.json (which you can look at on the hard drive).  If you haven't set anything, `<chef_repo_path>` will generally be the current directory.

When a second chef-client run goes off, the machine resource looks up the node and sees the instance ID already in it.  It uses the `driver_url` to load the AWS driver and your credentials, and checks if the instance is powered up and if ssh is available.

### I've been using custom bootstrap files with `knife bootstrap`.  Does Chef Provisioning support this?

Short answer: no.  Provisioning uses a different mechanism to register the machine with Chef and set up chef-client on it.

However, Chef Provisioning has a lot of capabilities that were not available with `knife bootstrap` and in many cases makes it unnecessary.  A few examples:

1. Many custom bootstraps exist to get secrets and other files up to the machine.  In Chef Provisioning, you can do that like this:

```ruby
machine 'foo' do
  file '/remote/path.txt', '/local/path.txt'
end
```

2. Other times, bootstrap files are used to do "pre-provisioning" recipes that set things up that need to be there before the main recipes run.  With Chef Provisioning, you can simply use the machine resource twice to get separate chef-client runs:

```ruby
machine 'foo' do
  recipe 'base' # Install the base stuff on the machine, connect it to AD, etc.
end
machine 'foo' do
  recipe 'my_web_app' # Install everything else
end
```

This will converge twice: the first converge will run the `base` cookbook and the second will run `my_web_app`.

### I used a rather insecure `admins` group workaround recommended in the early days of Provisioning, can I reverse that?

The Provisioning README had a neglected section giving bad security advice for a long while. Specifically, it suggested if you were using Hosted or Enterprise Chef, that you would need to add `clients` group to the `admins` group. To find out for sure whether you used that advice, you can check pretty easily by running:

```
knife show /groups/admins.json | grep clients
```

If you get no output, you're good. If you see "clients," then you probably added `clients` to the `admins` group. This can increase the attack surface of your organization, and the time to lock it back down is now.

In order to clean up the permissions and keep your nodes running, you need to remove `clients` from the `admins` group, and give each client `read` and `update` permissions to their corresponding node, explicitly. We've created a recipe to help you do this, so you can take these steps:

1. Go to the directory for your organization where you run `knife`.

2. Grab the [clean_acls.rb](https://github.com/chef/chef-provisioning/blob/master/docs/examples/clean_acls.rb) recipe, for example by running:

   ```
   curl https://raw.githubusercontent.com/chef/chef-provisioning/master/docs/examples/clean_acls.rb
   ```

3. Run this command to see what it would change (without actually making the changes):

   ```
   chef-client clean_acls.rb --why-run -l error -o "" -c .chef/knife.rb
   ```

   You should see something like:

   ```
   * chef_group[admins] action create
     - Would update group admins at https://api.opscode.com/organizations/chef-provisioning
     -   update groups from ["clients"] to []
   * chef_acl[nodes/mario] action create
     - Would update acl nodes/mario at nodes/mario
     -   read:  update actors from ["pivotal", "jkeiser"] to ["mario", "pivotal", "jkeiser"] 
   ```

4. If you're satisfied with the changes that would be made, run it for real by removing `--why-run`:

   ```
   chef-client clean_acls.rb -l error -o "" -c .chef/knife.rb
   ```
   
   You should see something like:

  ```
  * chef_group[admins] action create
    - update group admins at https://api.opscode.com/organizations/chef-provisioning
    -   update groups from ["clients"] to []
  * chef_acl[nodes/mario] action create
    - update acl nodes/mario at nodes/mario
    -   read:  update actors from ["pivotal", "jkeiser"] to ["mario", "pivotal", "jkeiser"] 
  ```

5. Make sure your nodes are still able to run chef-client by going to the node and seeing if the run completes or not.
