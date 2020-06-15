# Meraki Ansible Deployment Guide

When it comes to the enterprise more and more organisations are looking to standardise their tooling for the configuration and management of these networks. An increasingly popular tool is the use of Ansible as part of a wider service chain to standardise what deployments look like across multiple different vendors and platforms.

In this short guide I will be showing how we can use Ansible and Cisco Meraki to automate the deployment on branches the Meraki dashboards all the way from creation of networks, claiming of devices, binding of network templates and updating network specific details. This integration could also include a ITSM system such as ServiceNow or Jira which would have a handoff to Ansible to generate a data structure such as a CSV or YAML file which Ansible will use to create the network configuration in our examples

This kind of usecase is suitable for any kind of environment where branch networks are likely to be simple but the numbers of actual physical locations could go into the thousands and the challenge sits in the deployment. It’s typically not feasible to have a dedicated network professional visit each location to carry out the install. With this kind of solution the deployment can be a mostly physical job where all thats required it to plug in the cables and test connectivity as much of the deployment will have been done before the infrastructure even arrives at its destination. With this workflow we’re able to automate the whole logical deployment with a single playbook that can be triggered through a CI pipeline or ITSM ticket/request.

```Please note: This repo is intended for a demonstration and is not a 'production ready' CICD pipeline tha can be deployed, they're arre parts of this project that are missing such as network testing and robustness improvements that should be made before this could go into production.```

## The workflow

How this would actually be implemented in an organisation will differ the handover to the networking configuration here could happen and different points of the service chain. Some possible triggers for this playbook could include:

* CSV/YAML files generated by an ITSM system as the result of a service request, eg ServiceNow, Jira
* Configuration files per network built and commited to a source control system, kicking off a CICD process
* Part of an Ansible Tower workflow

In this example we'll be showing how we use a YAML file with the required details for our networks and devices this will populate the playbook we've build and allow it to run carrying out all the tasks required to set up the networks. For a full breakdown of the tasks in the playbook skip ahead to the 'playbook walkthrough' section.

![](images/workflow.png)

One of the things that makes this such a simple workflow is the Meraki platform, which provides us two main benefits here that are quite unique. Firstly Meraki supports ZTP natively therefore will allow the devices to call home as soon as they receive an internet connection aslong as their serial number has been registered to an organisation, which we do in our playbook. This will then allow the devices to pull down a config thats already been set well in advance and we don’t have to rely on our playbook getting individual device connectivity, the dashboard has the device configs already applied waiting for the device to announce itself.

Secondly the way Meraki supports templates for network configuration allows us to abstract away much of the manual configuration in our playbook and automate much of the deployment simply by attaching a template. All we have to do is build a few custom tasks for firewall rules and IP Addressing which is specific to our branch.

## CICD 

As we've demonstrated in our network above this could form part of a CICD pipeline to automate the deployment of branches. The devices and network elements can be defined through a YAML file like the example below. As new files are added to the source control for each branch we can automate CICD functionality (e.g. Gitlab CICD or Github actions) to automatically run the the playbook and deploy our branches.

In this scenario we can define the devices to be added to our network and the template to be bound from our YAML file definition. Like the below example where we define 4 devices to be added to the network, the Ansible playbook will read these files (included in the repo) and use the variables to carry out the required tasks in the playbook. Should you wish to add more or less devices to your network you just need to edit the YAML file with the appropriate number of devices.

```
---
 device-1:
    network_name: Branch-1321
    template_name: Branch-Template-Small
    device_name: br_1321_mx1
    device_type: MX67C-WW
    serial_no: XXXX-XXXX-XXXX
    vlan_id: 1

 device-2:
    network_name: Branch-1321
    template_name: Branch-Template-Small
    device_name: br_1321_sw1
    device_type: MS120-8LP
    serial_no: XXXX-XXXX-XXXX
    vlan_id: 1

 device-3:
    network_name: Branch-1321
    template_name: Branch-Template-Small
    device_name: br_1321_ap1
    device_type: MR70
    serial_no: XXXX-XXXX-XXXX
    vlan_id: 1

 device-4:
    network_name: Branch-1321
    template_name: Branch-Template-Small
    device_name: br_1321_ap2
    device_type: MR70
    serial_no: XXXX-XXXX-XXXX
    vlan_id: 1

```

In this scenario we're also outlining the IP addressing and subnets through an accompanying YAML file. As can be seen below, these override the VLANs for the network which the original template defines. Please note, if you do not have the correct number of subjects and the names aren't correct for your corresponding template this is likely to fail
```
---
 subnet-1:
    network_name: Branch-1321
    template_name: Branch-Template-Small
    name: VLAN_management
    vlan_id: 1
    subnet: 10.0.0.0/24
    default_gw: 10.0.00.1
    
 subnet-2:
    network_name: Branch-1321
    template_name: Branch-Template-Small
    name: VLAN_Payments
    vlan_id: 100
    subnet: 192.168.10.0/29
    default_gw: 192.168.10.1

 subnet-3:
    network_name: Branch-1321
    template_name: Branch-Template-Small
    name: VLAN_Corp
    vlan_id: 101
    subnet: 10.20.0.0/24
    default_gw: 10.20.0.1
```

## Prerequsites

To fully replicate this guide you will need a Meraki network which isn't something we can quickly spin up through virtual means such as DevNet sandboxes. However if you have a Meraki device, it's possible to replicate quite easily by building up your own YAML datastructures to meet the devices you have, the template you wish to deploy and he IP addressing for your environment.

You will need a machine with Ansible installed, a basic working knowledge of Ansible is preferred for this guide. Please see the following repositories which also cover Ansible:

* [Getting started with Ansible]()
* [Deploying ASAv in AWS with Ansible]()


## Ansible CLI

### Breakdown of playbook


```
---
- name: meraki deployment
  hosts: localhost
  vars:
    org_name: Meraki-Demo

  tasks:

    - name: include variables for devices
      include_vars:
        file: devices.yaml
        name: devices

    - name: include variables for addresses
      include_vars:
        file: addresses.yaml
        name: addresses

    - name: Create site network
      meraki_network:
        auth_key: "{{ auth_key }}"
        state: present
        org_name: "{{ org_name }}"
        name: "{{ item.value.network_name }}"
        type:
          - switch
          - appliance
          - wireless
      register: off_network
      loop: "{{ lookup('dict', devices) }}"
      when: "'device-1' in item.key"

    - name: Add devices to Network
      meraki_device:
        auth_key: "{{ auth_key }}"
        org_name: "{{ org_name }}"
        net_id: "{{ off_network.results.0.data.id }}"
        state: present
        serial: "{{ item.value.serial_no }}"
      register: off_add_dev1
      loop: "{{ lookup('dict', devices) }}"

    - name: Update device Information
      meraki_device:
        auth_key: "{{ auth_key }}"
        org_name: "{{ org_name }}"
        net_id: "{{ off_network.results.0.data.id }}"
        state: present
        serial: "{{ item.value.serial_no }}"
        name: " {{ item.value.device_name }}"
        move_map_marker: no
      register: off_update_dev1
      loop: "{{ lookup('dict', devices) }}"


    - name: Bind a template from a network
      meraki_config_template:
        auth_key: "{{ auth_key }}"
        state: present
        org_name: "{{ org_name }}"
        net_name: "{{ item.value.network_name }}"
        config_template: "{{ item.value.template_name }}"
      delegate_to: localhost
      loop: "{{ lookup('dict', devices) }}"
      when: "'device-1' in item.key"

    - name: Add subnets
      meraki_vlan:
        auth_key: "{{ auth_key }}"
        org_name: "{{ org_name }}"
        net_id: "{{ off_network.results.0.data.id}}"
        state: present
        name: "{{ item.value.name }}"
        vlan_id: "{{ item.value.vlan_id }}"
        subnet: "{{ item.value.subnet }}"
        appliance_ip: "{{ item.value.default_gw }}"
      loop: "{{ lookup('dict', addresses) }}"

```

Walking through this playbook you can see we have

### Running playbook

Now lets run the playbook, to do this manually this can be done simply with the command executed on your local workstation.

```
ansible-playbook deploy-branch-readyaml.yaml
```

The playbook will execute as per the animation below and create the required resources in the Meraki dashboard which you can now verify.

## Automate deployment with Github Actions

A more preferred option may be to automate this deployment process with a CICD pipeline.


### Github Actions

```
name: CICD  # feel free to pick your own name

on:
  push:
    branches: 
      - master
    paths:
      - 'playbooks/vars/**'

jobs:
  lint:

    runs-on: ubuntu-latest

    steps:
    # Important: This sets up your GITHUB_WORKSPACE environment variable
    - uses: actions/checkout@v2
     
    - name: Lint Ansible Playbook
      # replace "master" with any valid ref
      uses: ansible/ansible-lint-action@master
      with:
        targets: |
           playbooks/deploy-branch-readyaml.yaml
        override-deps: |
          ansible==2.8
          ansible-lint==4.2.0
        
    - shell: bash
      env:
        auth: ${{ secrets.MERAKIAPI }}
      run: sh ./scripts/entrypoint.sh
```


### Secrets

One of the biggest challenges

Luckily, Github actions solves this by allowing you to create secret variables for a repo which can be loaded in at runtime. Go to your repo settings and create a variable called "MERAKIAPI". As you can see below in the last action of our playbook we end up loading this as an environment variable called 'auth' which your shell script that calls the Ansible playbook runs.

```
    - shell: bash
      env:
        auth: ${{ secrets.MERAKIAPI }}
      run: sh ./scripts/entrypoint.sh
```

### Running

Now all that's left to do is get things running

```
name: CICD-Meraki


on:
  push:
    branches: 
      - master
    paths:
      - 'playbooks/vars/**'
```

