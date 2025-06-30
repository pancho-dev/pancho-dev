---
title: "Lightning Post: Don't have a Docker registry? save and load to the rescue"
date: 2020-08-06T20:30:48-03:00
draft: false
tags: ["docker","containers"]
---

Whenever someone is working with docker and need to move images between hosts without the need for rebuilding them or hosting a private registry. This [lightning post]({{< relref "lightning-post.md" >}}) will show us 2 commands that will help us to easy move container images between hosts.

# How to save a docker image for reuse in other host?

There is a handy command that helps save a docker image that is already in the system or already built, there are situations where builds take a long time and building the images in multiple hosts is time consuming, so we can save the image and copy it to other host and import it for reuse.  
To save a docker image:
```bash
# Pull an ubuntu image, just for demo purpose but it can be any image
$ sudo docker pull ubuntu

# Then save the image to a tar.gz file.
# once finished this file can be moved to other
# host running docker or backed up if we 
# don't have a registry
$ sudo docker save ubuntu | gzip > test.tar.gz
```

# How to restore an image?

Once we saved the image as tar.gz then it's easy to copy by scp or other means, and restore the same image there for future use.

```bash
# Load the file that was created with save
$ sudo gzcat test.tar.gz | sudo docker load
```
This will restore the image in a new host or from a backup.


# Bonus Track

There are other commands that can be useful to export and import containers instead of images. While `docker save` and `docker load` are used to handle and copy container images. There might be times where we need to copy a container, it can be a container instance that has some data or modifications in it and we want to move it somewhere else. IMPORTANT: always differentiate both use cases as one is for images and other is for container.

```bash

# Create a copy of a container in tar-gz file
$ sudo docker export testcontainer | gzip > testcontainer.tar.gz

# Restore a copy of a container as an image
$ sudo gzcat testcontainer.tar.gz | sudo docker import - testimage:new
```
This last example restores the backup of a container as an image not the container itself. So in this case the import will create a new image named `testimage:new` that will have all changes that testcontainer had, however the new container needs to be started with the new image in order to run it.

# Conclusion

I found this docker commands very handy when developing container and copying images from server to server without the need of a registry. It is something I use a lot at home for images I build and keep them locally, but I recommend using a private or public registry to keep your images, it simplifies the managing of container images, but we can't always have the infrastructure we need so this commands come very handy.  
I really care about being able to save and copy images as they are software artifacts and they need to be properly managed to be reused, I added a bonus track to show how to backup or copy and move a container just to differentiate the 2 operations, however if there is a need to use export it could be an indication that we are not following some recommended practices. This is not a hard "no" to use export, but in my case I always like to keep images and containers as immutable as possible and leave configurations outside of the container, this helps with reusability and reproducibility of a container, and if there is a need to modify some config inside the container either find a way to configure it with ENV vars or volume mount the config files so the container is easy to reproduce even if we destroy the container completely, this applies to stateful things like databases, always volume mount the data and take backups of that data. Reproducibility and reusability is something I take very serious as the same setup can be reproduced much faster and restore times get lowered, or it can be reused in multiple projects.

