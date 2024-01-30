---
layout: post
title: Installing Frigate using Docker and Ansible
date: 2024-01-30 22:40 +0000
tags: [homelab, frigate, nvr]     # TAG names should always be lowercase
---

My partner and I are soon to get a dog. So I've been tasked to "figure out" a
way so we can remotely monitor a camera, whilst our pup is sleeping. Awesome, I thought! An excuse to play with Frigate.  
  
In case you aren't aware, [Frigate](https://frigate.video/) is an open source
NVR built around real-time AI object detection. All processing is performed
locally on your own hardware, and your camera feeds never leave your home.  
  
At my core, I'm a cheapskate. The less money I need to spend on cloud
subscriptions, the better. So yes, a simple solution to my task is to buy a
camera off of Amazon and use the provided app etc etc. But where's the fun in
that?  
  

## Set up
I had a spare mini-pc lying around so I thought I'd install Ubuntu 22.04 LTS on
to it to give Frigate a whirl. Thankfully, I have spent some time on automating
my Ubuntu server configurations and initial Docker setup using Ansible, of which
you can have a look at the source code
[here](https://github.com/jalexmatey/ubuntu-home-server-infra).  
  
I'm a big fan of automating what I can using Ansible when it comes to my local
servers. Mostly because I forget about how I initially set something up, and I 
really don't like spending time on the same thing twice. So, Ansible is my
saviour again!  
  
Creating the `.yml` file for frigate was surprisingly simple, and this is what
I ended up with:  

```yaml
---
- name: Create the frigate config directory.
  ansible.builtin.file:
    path: "/home/{{ username }}/frigate/config"
    state: directory
    mode: '0755'

- name: Copy config.yml file
  ansible.builtin.template:
    src: templates/config.yml.j2
    dest: "/home/{{ username }}/frigate/config/config.yml"
    owner: "{{ username }}"
    group: "{{ username }}"
    mode: u=rw,g=rw,o=r

- name: Deploy Frigate container.
  community.docker.docker_container:
    name: frigate
    image: ghcr.io/blakeblackshear/frigate:stable
    ports:
      - "5000:5000"
      - "8554:8554" # RTSP feeds
      - "8555:8555/tcp" # WebRTC over tcp
      - "8555:8555/udp" # WebRTC over udp
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /home/{{ username }}/frigate/config:/config
      - /mnt/HDD:/media/frigate
      - type: tmpfs # Optional: 1GB of memory, reduces SSD/SD Card wear
        target: /tmp/cache
        tmpfs:
          size: 1000000000
    shm_size: "64MB"
    devices:
      - /dev/dri/renderD128 # for intel hwaccel, needs to be updated for your hardware
    network_mode: "{{ docker_network }}"
    env:
      FRIGATE_RTSP_PASSWORD: "password"
    restart_policy: unless-stopped
```

I templated the recommended `config.yml` file as described by the Frigate
documentation:  
```yaml
mqtt:
  enabled: False

cameras:
  dummy_camera: # <--- this will be changed to your actual camera later
    enabled: False
    ffmpeg:
      inputs:
        - path: rtsp://127.0.0.1:554/rtsp
          roles:
            - detect
```

Once I fully get Frigate up and running, this can be changed. All I'd have to do
then is run the playbook again and it should copy over the config.  

Speaking of the playbook:  
```yaml
---
- name: Configure NVR systems
  hosts: docker_hosts

  vars_files:
    - vars/system_vars.yml
    - vars/docker_container_vars.yml
    - vars/secret_vars.yml

  tasks:
    - name: Deploy Frigate container
      ansible.builtin.import_tasks: tasks/docker-containers/frigate/main.yml
      tags: ['frigate']
```

I played around with a spare webcam I had lying around, using OBS studio and a
RTSP server plugin so Frigate could access it. Changing the `config.yml` to
use the webcam, I can confirm that it was working successfully.  
  
## Conclusion  
Honestly, I'm really surprised how quickly it can be to set up a self-hosted
NVR system. So props to Frigate for that!  

This also lays a lot of groundwork for adding more cameras to my home. Now I 
just need to buy more cameras...

## Acknowledgments
* Thanks to the creators of [Frigate](https://frigate.video/). It's an awesome
piece of OSS. Please remember to give Frigate a ⭐ on [GitHub](https://github.com/blakeblackshear/frigate)

