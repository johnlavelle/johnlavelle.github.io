---

theme: dark

---

<!-- Global style -->
<style>
img[alt~="center"] {
  display: block;
  margin: 0 auto;
}
</style>



# **The ECHOES Earth Observation Service**  <!-- fit -->
#
![center width:200px](images/echoes_icon.png)
![center width:550px](images/compass.png)

---

Polar Orbiting Satellite   |  Sentinel Satellites
:-------------------------:|:-------------------------:
![center width:600px](images/satellite_orbit.gif) | ![center width:600px](images/sentinel_satellites.gif)

---

## The challenge is to...

* Handle the large volumes of data for the satellites
* Apply algorithms to get meaningful information from the raw satellite data

---

# The ECHOES web application

![center](images/echoes_site.JPG)

---
# Sentinel Hub based processing chain

* Many processing scripts available in the [Custom Scripts repo](https://custom-scripts.sentinel-hub.com/#sentinel-2)
* Python code provides a CLI to the custom scripts
* The processing is done on Sentinel-Hub's servers'
* It does mosaicing for given interval (month, etc.)

---
# The system architecture

![bg w:480 right](images/img.png)

* A request is sent to process a given ROI with the given processing script
* The processing is done remotely, on a Sentinel-Hub server
* The results (GeoTIFF...) are stored an object store
* The GeoTIFF is transferred to the web-service VM (running GeoServer)    

---

<!-- backgroundColor: black -->

![center](images/eo_custom_scripts.gif)


---
<!-- backgroundColor: default -->

# Processing using the source satellite data

* The CREODIAS object store contains over 20 PB of satellite data (stored in e.g. the SAFE format)
* Code has been developed to process this data
* It has a similar interface to the Sentinel-Hub processing chain
* Works with SNAP, Satpy, ... 

[![bg right:50% 40%](https://mermaid.ink/img/pako:eNo1j7FuAjEQRH9ltRVIUNBeESkEahCkO1Ns7IEz3NlkvYeEEP-OpZBuRnozo3mwzwHc8Enl2tH3yqXPyQ4e8QZS_I4oNqX5_MOlZbuHqO_omJWKGPo-GiiIyeGP-Gq3mj1KefsYFu1k83OGNyqWFdMDEc94gA4SQ119uETk2DoMcNxUGUQvjl16Vm681nKsQ6xZbo7SF8xYRsv7e_LcmI74h1ZR6oPhTT1fagRI9w)](https://mermaid.live/edit#pako:eNo1j7FuAjEQRH9ltRVIUNBeESkEahCkO1Ns7IEz3NlkvYeEEP-OpZBuRnozo3mwzwHc8Enl2tH3yqXPyQ4e8QZS_I4oNqX5_MOlZbuHqO_omJWKGPo-GiiIyeGP-Gq3mj1KefsYFu1k83OGNyqWFdMDEc94gA4SQ119uETk2DoMcNxUGUQvjl16Vm681nKsQ6xZbo7SF8xYRsv7e_LcmI74h1ZR6oPhTT1fagRI9w)

---

<style scoped>
pre {
   font-size: 2rem;
}
</style>

# An example processor 

```cs
#!/usr/bin/env python3

from os.path import dirname
from satpy import Scene, find_files_and_readers
from shapely import wkt
from eoian import utils
from eoian import command_line_interface

def main(input_file: str, area_wkt: str) -> "Dataset":
    files = find_files_and_readers(base_dir=dirname(input_file), reader='msi_safe')
    scn = Scene(filenames=files)
    scn.load(['B04', 'B08'])
    area = wkt.loads(area_wkt)
    epsg = scn['B04'].area.crs.to_epsg()
    xy_bbox = utils.get_bounds(area, epsg)
    scn = scn.crop(xy_bbox=xy_bbox)
    extents = scn.finest_area().area_extent_ll
    ad = utils.area_def(extents, 0.0001)
    s = scn.resample(ad)
    ndvi = (s['B08'] - s['B04']) / (s['B08'] + s['B04'])
    s['ndvi'] = ndvi
    s['ndvi'].attrs['area'] = s['B08'].attrs['area']
    del s['B04']; del s['B08']
    return s

@command_line_interface.processing_chain_cli(to_zarr=False)
def cli(input_file: str, area_wkt: str):
    return main(input_file, area_wkt)

if __name__ == '__main__':
    cli()
```

More examples: https://github.com/ECHOESProj/eo-processors

---

Development and production servers

![w:600 center](images/eo_service.drawio.png
)

---

# The EO service is containerised


![bg w:400 right ](images/docker_kubernetes.png)


This makes it easier to deploy,

... and run on your local machine.

It can be run on a container service, such as K8,

... which improves the processing efficiency and scalability.

---

# Jupter Lab on the CREODIAS server

[![width:640 center](images/jupyter.JPG)](https://185.52.192.218:8888)

---

<!-- backgroundColor: default -->

# Automation of the servers

![bg right:30% 50%](images/Ansible_logo.svg.png)

Playbook tasks:
```cs
- name: Copy shh keys over to eo-processor
  copy:
    src: '{{ item.path }}'
    dest: '{{ ansible_env.HOME }}/echoes-deploy/eo-processors/credentials/'
    remote_src: yes
    owner: '{{ ansible_user_id }}'
    group: '{{ ansible_user_id }}'
    mode: '0700'
  with_items: "{{ ssh_files.files }}"

- name: Build Docker image
  community.docker.docker_image:
    name: '{{ item }}'
    build:
      path: '{{ ansible_env.HOME }}/echoes-deploy/{{ item }}'
      network : host
    source: build
  loop:
    - "eo-custom-scripts"
    - "eo-processors"
    - "websockets-server"

- name: Create a Docker volume
  docker_volume:
    name: el-vol

- name: Run `docker-compose up`
  community.docker.docker_compose:
    project_src: '{{ ansible_env.HOME }}/echoes-deploy/eo-stack/'
    env_file: '{{ ansible_env.HOME }}/env_file'
    files:
      - docker-compose.yml

```

---

<!-- backgroundColor: black -->

![center](images/playbook.gif)

---

<!-- backgroundColor: default -->

# The Repositories (https://github.com/ECHOESProj/)

**EO Service Code**
Processors: https://github.com/ECHOESProj/eo-processors
Sentinel Hub automation: https://github.com/ECHOESProj/eo-custom-scripts
The object store processing chain: https://github.com/ECHOESProj/eoian
Read/write to object store: https://github.com/ECHOESProj/eo-io
Websockets server: https://github.com/ECHOESProj/websockets-server
Docker Compose file: https://github.com/ECHOESProj/eo-stack 

**Prototyping**
Jupyter Notebooks: https://github.com/ECHOESProj/eo-notebooks

**Automation**
Server automation using Ansible: https://github.com/ECHOESProj/eo-playbooks

**Documentation**
Overview docs: https://github.com/ECHOESProj/eo-docs
