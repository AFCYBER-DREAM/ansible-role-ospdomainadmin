osp-domain-admin
============================================

The purpose of this role is to define and manage any domain administrator-controlled 
OpenStack resources (i.e. projects, groups, project admin users/groups, quotas, secgroups, secgroup rules, et cetera)
within an OpenStack Platform installation utilizing Keystonev3 and Active Directory.

Requirements
------------

- Ansible 2.4+
- Python Shade library 1.9+
- python-openstackclient

Role Variables
--------------

/path/to/your/playbooks/host_vars/localhost:

    ---
    osp_domain_admin_domain_vars:
      domain_name: "BADGER"                                                  # Name of the domain in OpenStack
      domain_type: "active directory"                                        # Either "local" for a default keystone domain, or "active directory" for an AD-integrated one
      domain_admin_group: "OSP10_Admins_All"                                 # The group added to each of the domain's project assigned the role "admin"
      default_project_network_nameservers: "10.16.1.10,10.16.1.11"           # DNS servers for default private networks for projects with "use_default_networking" set to true
      default_project_network_cidr: "192.168.0.0/24"                         # CIDR for the default network created in projects with "use_default_networking" set to true
  
    osp_domain_admin_project_vars:
    - name: "cym-all-sbox"                                                   # Name of one of the projects list's projects
      description: "(Sandbox) Project for CYM all's sandbox infrastructure"  # Description of the project
      group: "CYM_All"                                                       # The actice directory group that should be given the "_member_" role for this project
      quota_flavor: "small"                                                  # Quota flavor to use from the osp_domain_admin_managed_quota_flavors dictionary
      use_default_networking: true                                           # Whether or not project gets domain-admin managed default networking settings
    ...

Role Dependencies
------------

None

Example Playbook
----------------

    ---
    - hosts: localhost
      connection: local
      become: false
      gather_facts: false
    
      roles:
        - osp-domain-admin
    ...


Notes
-----

You may have already noticed that there are no group_vars required for this role; however, there is a requirement for all of the authentication credentials to be sourced in the same shell that is used to execute the playbook. Prior to running the playbook, you should be able to run `env | grep "OS_"` and see all the environment variables you'll need defined. Then, from that same terminal prompt, you will execute the playbook. Trying to execute the `source openrc` (or similar) command inside of the playbook itself does not work, because each task inside of the playbook is a child process of the shell used to execute the playbook. So, once each task completes, any env vars that it set that were specific to that child process, are dumped. These do not flow back up to the parent process that launched it.

The whole reason these variables are even required, is because the `os_` series of Ansible modules requires authentication credentials in order to auth to the OpenStack installation it is managing. The modules are relatively uniform in their authentication options, and each of them offers three different ways to authenticate. The first is `cloud-client-config`. This python library reads some variables from a location on the filesystem (a cloud.yml) file. The file is setup to define multiple clouds in a single place and call them by a nickname. The modules use the `cloud` parameter to reference this nickname, and look for these files in specific directories (e.g. /etc/openstack/). This method appears to have some limitations related to how it defines and stores some of the Keystone/Identity v3 parameters though. 

A similar authentication method used by these modules is the `auth` series of parameters that can be defined. These, by and large, have a one-to-one equivalency to the bash env vars defined when you `source openrc` the OpenStack provide credentials that you can download from Horizon. There are; however, some exceptions to that, which make it impractical for use. 

Additionally, it is extremely advantageous to use the same exact authentication method that the `python-openstackclient` uses when it authenticates to Keystone to perform API calls. As a result, sourcing an OpenStack provided `openrc` file is the authentication method of choice for this role. You'll notice a test of the env vars is performed at the front end of the playbook, and you are afforded the opportunity to review the env vars before the playbook proceeds with its run. 

In order to check your env variables, they are displayed when it reaches the appropriate step in the playbook. You can skip the review process by hitting <CTRL>+<C> and then <C> (Continue) or you can abort by hitting <CTRL>+<C> and then <A> (Abort) when prompted. If not, the playbook will proceed after 10 seconds.


Usage
------- 
Executing the playbook is relatively simple. Since it is executed on the localhost, there is no need for an inventory file. Ansible will throw a little warning, but its default behavior is to execute on localhost when you don't provide a user-defined group for it to run the playbook against. You'll just need to ensure you've sourced your OpenStack credentials file prior to running the playbook. Also make sure you have the `shade` library installed and accessible in your path, or else the Openstack `os_` series Ansible modules cannot execute. For secgroup-related tasks, you'll also need the `python-openstackclient` installed via package manager or pip, since the os_security_group and os_security_group_rule ansible modules lack the appropriate parameters to deal with security groups that share a name with other security groups in other projects (i.e. "default"). Here is an example of the commands:

You also have the ability to "scope" your actions to specific task groups or projects by using extra_vars when you initialize the playbook.

The `project_scope` variable accepts any project name from your `host_vars/localhost` file in the `osp_domain_admin_project_vars` dictionary.

The `task_scope` accepts the name of any of the top-level ansible tasks listed within `roles/osp-domain-admin/tasks/main.yml`. These currently include:

   - "general"     # Creates the project and updates it's description
   - "users"       # Adds the domain-admin and project member groups and assigns the appropriate roles
   - "networking"  # If toggled on with "use_default_networking" in `host_vars/localhost` this creates a network, subnet, and router
   - "secgroups"   # Creates default secgroups and rules for project
   - "quotas"      # Sets a "quota_flavor" from `host_vars/localhost` to give the project sensible resource limitations based on need


If left undefined, both the `task_scope` and `project_scope` default to "all". After setting up your variables, and deciding your scope, you can run the playbook like so: 

```
$ pip freeze 2>&1 | egrep -i "shade"  # Ensure the "shade" module is listed in the output
$ source ./openrc   # Or whatever the name of your credentials file is...
$ ansible-playbook osp-admin.yml -e "project_scope=all" "task_scope=general"  # This runs the "general" tasks against all projects
```

License
-------

MIT

Author Information
------------------

The Development Range Engineering, Architecture, and Modernization (DREAM) Team.
