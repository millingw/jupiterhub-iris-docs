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


