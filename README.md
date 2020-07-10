# flatcar-python-bootstrap

In order to effectively run ansible, the target machine needs to have a python interpreter. [Flatcar Container Linux](https://www.flatcar-linux.org/) machines are minimal and do not ship with any version of python. To get around this limitation we can install [pypy](http://pypy.org/), a lightweight python interpreter. The coreos-bootstrap role will install pypy for us and we will update our inventory file to use the installed python interpreter.

## install

```bash
ansible-galaxy install git+https://github.com/swoehrl-mw/ansible-flatcar-python-bootstrap
```

## Configure your project

Unlike a typical role, you need to configure Ansible to use an alternative python interpreter for coreos hosts. This can be done by adding a `coreos` group to your inventory file and setting the group's vars to use the new python interpreter. This way, you can use ansible to manage CoreOS and non-CoreOS hosts. Simply put every host that has CoreOS into the `coreos` inventory group and it will automatically use the specified python interpreter.

```ini
[coreos]
host-01
host-02

[coreos:vars]
ansible_user=core
ansible_python_interpreter=/home/core/bin/python
```

This will configure ansible to use the python interpreter at `/home/core/bin/python` which will be created by the flatcar-python-bootstrap role.

This role has two modes. By default it will pull pypy from [github](https://github.com/squeaky-pl/portable-pypy) get-pip.py from [https://bootstrap.pypa.io/get-pip.py](https://bootstrap.pypa.io/get-pip.py). For air-gapped scenarios where you do not have internet access or do not want ansible to pull files from the internet you can also provide a custom URL for pypy and provide pip in the form of a bundle.

To create a bundle run the following commands:

```bash
mkdir -p python_bundle
pip3 download pip==20.0.2 -d python_bundle
tar -zcf python_bundle.tar.gz python_bundle
```

Then upload this archive and the [pypy-portable archive](https://github.com/squeaky-pl/portable-pypy/releases/download/pypy3.6-7.2.0/pypy3.6-7.2.0-linux_x86_64-portable.tar.bz2) to a location your hosts can reach. To use them set the following group vars:

```yaml
python_bootstrap:
  pypy_url: <location-of-pypy-archive>
  bundle_url: <location-of-bundle>
```

You may need to set `hash_behaviour = merge` in your `ansible.cfg`.

### Bootstrap Playbook

Now you can simply add the following to your playbook file and include it in your `site.yml` so that it runs on all hosts in the coreos group.

```yaml
- hosts: coreos
  gather_facts: False
  roles:
    - ansible-flatcar-python-bootstrap
```

Make sure that `gather_facts` is set to false, otherwise ansible will try to first gather system facts using python which is not yet installed!

### Example Playbook

After bootstrap, you can use ansible as usual to manage system services, install python modules (via pip), and run containers. Below is a basic example that starts the `etcd` service, installs the `docker-py` module and then uses the ansible `docker` module to pull and start a basic nginx container.

```yaml
- name: Nginx Example
  hosts: web
  sudo: true
  tasks:
    - name: Start etcd
      service: name=etcd.service state=started

    - name: Install docker-py
      pip: name=docker-py

    - name: pull container
      raw: docker pull nginx:1.7.1

    - name: launch nginx container
      docker:
        image="nginx:1.7.1"
        name="example-nginx"
        ports="8080:80"
        state=running
```
