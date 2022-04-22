ovirt_renew_certs
=========

Ovirt needs (at least) self generated certificates to make the engine and the hosts safely communicate. They are now valid 365 days by default. If a those certificates are not valid anymore, the hosts can't communicate anymore with the engine and the vms go into an unknown state. 
The official way to renew certificates is to put the concerned host into maintenance and check "enroll certificates", but this is not possible anymore when certificates have expirated. It is not possible to interact with vms to migrate them or properly shutdown them. The only way is to use virsh shutdown and finally fence the host.
An intermediary solution is given by RedHat https://access.redhat.com/solutions/3532921, and this ansible role aims to automate it. 

Requirements
------------

* Ansible must be installed on the controller, and all targeted hosts (engine and hosts) must be reachable with a simple "ansible -m ping".
* By default, host checking is disabled in ansible.cfg. You can change this behaviour with `export ANSIBLE_HOST_KEY_CHECKING=False` or with `host_key_checking = False` in `ansible.cfg`.
* Engine (server) must login into host without password.

        #engine ssh-copy-id root@host

* The role install python3 and pip dependencies to install the ovirt-engine-sdk-python, but it can be manually done with:

    yum install -y python3-pip gcc openssl-devel libcurl-devel libxml2-devel python36-devel
    or
    sudo apt install libxml2-dev lib32z1-dev libcurl4-openssl-dev build-dep python-lxml
    python3 -m pip install -U pip
    python3 -m pip install ovirt-engine-sdk-python
    
* RedHat or similar is the prefered OS for the controller, but it should run as well on Ubuntu/Debian.

Role Variables
--------------

* This role must be configured at least with `ovirt_password` and `domain` variable. 
* `server` is not mandatory because the value is extracted from a query to multiple engines, but it can be forced.
* `pem_validity` should be equal to `csr_validity` but strictly according to the Redhat solution, they must be distinct.

- ovirt_password: "admin@internal_password"
- server: arum
- csr_validity: 365
- pem_validity: 365
- vdsmkey: vdsmkey.pem
- vdsmcert: vdsmcert.pem
- vdsmkey_path: /tmp
- domain: v100.domain.com
- engines: [engine1, engine2, engine3]


Important Note
--------------

The hosts are not some variables but are some targeted hosts from the inventory. You should write a solid inventory file:

    [ovirt_hosts]
    host1 ansible_hostame=vm706-dev.my_domain.com
    host2 ansible_hostame=vm706-dev.my_domain.com

Example Playbook
----------------

    - hosts: all
      gather_facts: no

      tasks:
        - name: 
          include_role:
            name: natman.ovirt_renew_certs
          vars: 
            vdsmkey: vdsmkey.pem
            vdsmkey_path: /tmp
            csr_validity: 365
            pem_validity: "{{survey_pem_validity|default('365')}}"
            
You can change vars value on the CLI like this:

    ansible-playbook role_ovirt_renew_certs.yml --limit ovirt_hosts (-i inventory)
                                                  -e ovirt_password='my_password'
                                                  (-e server='my_engine')
                                                  -e engines="['host1', 'host2']"
                                                  -e csr_validity=365
                                                  -e vdsmkey=vdsmkey.pem
                                                  -e vdsmcert=vdsmcert.pem
                                                  -e vdsmkey_path=/tmp
                                                  -e domain=my_domain.com
                                                  
    ansible-playbook role_ovirt_renew_certs.yml --limit ovirt_hosts -e @vars.yml
                                                  
Or by modifying the `vars/main.yml` file

Good Practice
-------------

You should always encrypt your password with `ansible-vault`. Then simply add `--ask-vault-pass` or `vault-id=password` to the CLI.

License
-------

BSD

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
