ovirt_renew_certs
=========

`oVirt` needs (at least) self generated certificates to make the engine and the hosts safely communicate. They are now valid 365 days by default. If a those certificates are not valid anymore, the hosts can't communicate anymore with the engine and the vms go into an unknown state. 
![image](https://user-images.githubusercontent.com/1138093/165585909-0a2ffa92-7e03-454b-8828-6ac96a7755e0.png)

The official way to renew certificates is to put the concerned host into maintenance and check `enroll certificates`, but this is not possible anymore when certificates have expirated, neither it is possible to interact with vms to migrate them or properly shutdown them. In this cas ,the only way is to use virsh shutdown and finally fence the host.
An intermediary solution is given by RedHat https://access.redhat.com/solutions/3532921, and this ansible role aims to automate it. 

The role can also be used when certificates are about to expire or if you want to chose a longer cert validity than 365 days.
![image](https://user-images.githubusercontent.com/1138093/165586219-a30d6415-c67d-4863-b527-6f813404e209.png)

As recommended in the RedHat solution, this way to do should be used as a workaround. When hosts are finally able to communicate with the engine, you should consider to put your hosts in a maintain state so as to enroll certificates with UI. Omitting this can lead to some issues like vms migrations failure or non functionnal graphical console.


Requirements
------------

* Ansible must be installed on the controller, and all targeted hosts (engine and hosts) must be reachable with a simple 

        ansible -m ping -i inventory ovirt_hosts
      
  If not, you should use `ssh-keygen` and `ssh-copy-id` on your controller
* By default, host checking is enabled in ansible.cfg. You can change this behaviour with `export ANSIBLE_HOST_KEY_CHECKING=False` or with `host_key_checking = False` in `ansible.cfg`.

* The role install python3 and pip dependencies to install the ovirt-engine-sdk-python, but it can be manually done with:

        yum install -y python3-pip gcc openssl-devel libcurl-devel libxml2-devel python36-devel
        or
        sudo apt install libxml2-dev lib32z1-dev libcurl4-openssl-dev build-dep python-lxml
        python3 -m pip install -U pip
        python3 -m pip install ovirt-engine-sdk-python
    
* RedHat or similar is the prefered OS for the controller, but it should run as well on Ubuntu/Debian.
* __Before using this role into a playbook, you should copy this role to your roles path into `ansible.cfg`__
* __The easiest way to use this role is to download it from `Ansible Galaxy` (https://galaxy.ansible.com/natman/ovirt_renew_certs) like this:__

         $ ansible-galaxy install natman.ovirt_renew_certs
         $ cd $HOME/.ansible/roles/natman.ovirt_renew_certs/tests/
         $ ansible-playbook -i inventory role_ovirt_renew_certs.yml (--limit host1,host2)

Role Variables
--------------

*  If using `test/role_ovirt_renew_certs.yml`, it prompts minimal questions to register custom variables, but prompt can be bypass adding `-e` args.
*  This role must be configured at least with `ovirt_password` and `engine` variable. 
* `server` is not mandatory because the value is extracted from a query to multiple engines, but it can be forced.
* `pem_validity` should be equal to `csr_validity` but strictly according to the Redhat solution, they must be distinct.

- ovirt_password: "admin@internal_password"
- engine: my_engine.domain.com
- csr_validity: 365
- pem_validity: 365
- vdsmkey: vdsmkey.pem
- vdsmcert: vdsmcert.pem
- vdsmkey_path: /tmp

Important Note
--------------

The hosts are not some variables but are some targeted hosts from the inventory. You should write a solid inventory file, you call them with the `--limit` flag on the CLI.

    [ovirt_hosts]
    host1 ansible_hostame=vm706-dev.my_domain.com
    host2 ansible_hostame=vm706-dev.my_domain.com

Example Playbook
----------------

        - hosts: all
          gather_facts: no

          vars_prompt:

            - name: engine
              prompt: Enter oVirt engine
              default: my_engine.domain.com
              private: no

            - name: ovirt_password
              prompt: Enter oVirt engine admin password
              unsafe: yes
              #encrypt: sha256_crypt

            - name: pem_validity
              prompt: Validity of the certificates in days
              default: "365"
              private: no

          tasks:
            - name: 
              include_role:
                name: ovirt_renew_certs
              vars: 
                vdsmkey: vdsmkey.pem
                vdsmkey_path: /tmp
                csr_validity: 365
                pem_validity: "{{survey_pem_validity|default('365')}}"
              tags: always

            
* Vars are default prompted but if you prefer, you can instead act as following.
            
    * You can change vars value on the CLI like this:

            ansible-playbook test/role_ovirt_renew_certs.yml --limit ovirt_hosts (-i inventory)
                                                          -e ovirt_password='my_password'
                                                          -e engine=my_engine.domain.com
                                                          -e csr_validity=365
                                                          -e vdsmkey=vdsmkey.pem
                                                          -e vdsmcert=vdsmcert.pem
                                                          -e vdsmkey_path=/tmp
                                                  
    * Or with a yaml file as input:

            ansible-playbook test/role_ovirt_renew_certs.yml --limit ovirt_hosts -e @vars.yml

        with `vars.yml` as following:

                ---
                ovirt_password: "admin@internal_password"
                engine: my_engine.domain.com
                csr_validity: 365
                pem_validity: 365
                vdsmkey: vdsmkey.pem
                vdsmcert: vdsmcert.pem
                vdsmkey_path: /tmp

                                                  
    * Or by modifying the `defaults/main.yml` file

Good Practice
-------------

You should always encrypt your password with `ansible-vault`. Then simply add `--ask-vault-pass` or `vault-id=password` to the CLI.

License
-------

BSD

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
