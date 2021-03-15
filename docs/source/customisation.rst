Customisation
=============
Once we have a working JupyterHub service we will likely want to customise it in various ways.

Custom Notebook Images
----------------------
Creating a custom notebook image is typically used to preinstall software. The most straightforward method here is to start from an existing base image and customise it as necessary. For example, this Dockerfile takes the fairly minimal base-notebook image and adds the astropy library (using Conda) and git (using apt)::

    FROM jupyter/base-notebook

    RUN conda update -n base conda \
      && conda install --quiet --yes \
      'astropy'

    USER root
    RUN apt-get update && apt-get -yq dist-upgrade \
      && apt-get install -yq --no-install-recommends \
      git \
      && apt-get clean


We could then build this and push it to a suitable docker repository using something like the following::

    $ docker build .
    $ docker tag 0a9231ff8196 my-repo/my-notebook:0.1.2
    $ docker push my-repo/my-notebook:0.1.2

If we are using helm chart to deploy JupyterHub we might specify our new image in the values.yml file like this::

    ...
    singleuser:
      image:
        name: my-repo/my-notebook
        tag: 0.1.2
    ...

Custom Hub Image
----------------
Customising the hub image is not always required, but may be useful if we want to modify the templates to change the look and feel of JupyterHub, or include additional content. For example we might want to customise the login page template to add terms and conditions for accessing the service.

This example Dockerfile replaces the login page template with a custom version::

    FROM jupyterhub/k8s-hub:0.8.2

    USER root
    RUN mkdir /usr/local/share/jupyterhub/custom_templates 
    COPY login.html /usr/local/share/jupyterhub/custom_templates
    USER $NB_UID

We could then build this and push it to a suitable docker repository using something like the following::

    $ docker build .
    $ docker tag 019a31f72196 my-repo/my-hub:0.8.2
    $ docker push my-repo/my-hub:0.8.2

If we are using helm chart to deploy JupyterHub we might specify our new image in the values.yml file like this::

    ...
    hub:
      image:
        name: my-repo/my-hub
        tag: 0.1.2
    ...


User Image Selection
--------------------
We might want to allow a user to select from a range of notebook images. This is possible by overriding the KubeSpawner class.

If we are using helm chart to deploy JupyterHub we can do this in the values.yml file::

    ...
    hub:
      extraconfig:
        custom_spawner: |
          from kubespawner.spawner import KubeSpawner
          from traitlets import observe
          class MySpawner(KubeSpawner):
            def _update_options(self, change):
              options = change.new
              if 'image' in options:
                self.singleuser_image_spec = options['image']
            def options_from_form(self, formdata):
              images = {
                0: "192.168.1.2:5000/my-notebook:0.1",
                1: "jupyter/base-notebook",
                2: "jupyter/datascience-notebook",
              }
              img = int(formdata.get('image', [''])[0])
              options['image'] = images.get(img, images[0])
              return options
            @gen.coroutine
            def options_form(self, spawner):
              form = """
              <label for='image'>Image</label>&nbsp;
              <select name='image'>
                <option value='0' selected='selected'>My Notebook</option>
                <option value='1'>Standard Notebook</option>
                <option value='2'>Data Science Notebook</option>
              </select>"""
          c.JupyterHub.spawner_class = MySpawner
    ...

Here we use the hub.extraconfig parameter which simply inserts additional python code at the end of jupyterhub_config.py. We could also do this in other ways, by creating a custom hub image for example.

User Selection of Resources
---------------------------
This allows a user to select diffrent resource limits depending on their requirements. The principle is very similar to image selection above.

If we are using helm chart to deploy JupyterHub we can do this in the values.yml file::

    ...
    hub:
      extraconfig:
        custom_spawner: |
          from kubespawner.spawner import KubeSpawner
          from traitlets import observe
          class MySpawner(KubeSpawner):
            def _update_options(self, change):
              options = change.new
                if 'mem' in options:
                  self.mem_guarantee = options['mem']
                  self.mem_limit = options['mem']
                if 'cpu' in options:
                  self.cpu_guarantee = options['cpu']
                  self.cpu_limit = options['cpu']
            def options_from_form(self, formdata):
              flavours = {
                "small": ("512M", 0.25),
                "medium": ("2G", 1.0),
                "large": ("8G", 4.0),
              }
              flavour = str(formdata.get('flavour', [''])[0])
              options['mem'],options['cpu'] = flavours.get(flavour, flavours[1])
              return options
            @gen.coroutine
            def options_form(self, spawner):
              form = """
              <label for='flavour'>Flavour</label>&nbsp;
              <select name='flavour'>
                <option value='small'>Small (0.5G / 0.25 CPU)</option>
                <option value='medium' selected='selected'>Medium (2G / 1 CPU)</option>
                <option value='large'>Large (8G / 4 CPU)</option>
              </select>"""
          c.JupyterHub.spawner_class = MySpawner
    ...


