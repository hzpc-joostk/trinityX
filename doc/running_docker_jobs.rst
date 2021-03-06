
Running docker apps on a cluster
================================

TrinityX comes with the ability to run arbitrary dockerized MPI jobs on a cluster.

.. note:: Please consult the `official docker documentation <https://docs.docker.com/>`_ for more information on Docker and how it works.

The only requirement to be able to run such jobs is to have openssh-server running in the container and listening on the non-standard port 2222.

Keep in mind that the application to be run as an MPI job must be installed or pre-compiled in the container image.
All of its dependencies need to be installed/pre-compiled as well.

Building a Docker image
-----------------------

To be able to run the dockerized MPI job you need first to provide it to the cluster as a Docker image. To do so, two options are available:

Building on the controller
``````````````````````````

A TrinityX controller comes pre-installed with Docker and Docker-registry (assuming that the docker option was selected at install time).
This makes it possible for an administrator to create a Docker image that can subsequently be run on the cluster.

It is worth mentioning here that regular users cannot issue Docker commands unless permitted by the admins.

With that cleared up, let's build a Docker image that we can then use to run an OSU MPI benchmark:

1. First, create an empty directory to serve as the workdir.
2. Create a special file called `Dockerfile` that will be used to build the Docker image; an example is provided below::

    FROM centos:latest

    RUN yum -y install epel-release && \
        yum -y install openssh-server openssh-clients\
                       environment-modules \
                       openmpi openmpi-devel \
                       libibverbs librdmacm \
                       make gcc-c++ && \
        yum clean all

    RUN echo "module load mpi" > /etc/profile.d/openmpi.sh && \
        chmod +x /etc/profile.d/openmpi.sh

    RUN curl -O http://mvapich.cse.ohio-state.edu/download/mvapich/osu-micro-benchmarks-5.3.2.tar.gz && \
        tar xf osu-micro-benchmarks-5.3.2.tar.gz -C /root/

    RUN bash -lc "cd /root/osu-micro-benchmarks-5.3.2; ./configure CC=mpicc; make; make install"

    ENTRYPOINT ["/usr/sbin/sshd", "-p", "2222", "-D"]


As you can see, this Dockerfile satisfies the TrinityX requirements as it sets up openssh-server correctly.
It also installs all of the dependencies required to run the OSU benchmarks.

3. We then can build our image using `docker build`::

    # docker build -t <controller-hostname>:5000/osu .

Make sure to replace <controller-hostname> with the correct hostname.

4. Lastly, we need to publish our image so that the compute nodes can fetch it when required::

    # docker push <controller-hostname>:5000/osu


Using a remote Docker registry
``````````````````````````````

If you prefer to build your images elsewhere and store them on a Docker registry other than the one provided by TrinityX, then it will be necessary to update the compute images.
 
Since compute nodes will need to query a remote Docker registry for Docker images, this one needs to be declared in `/etc/sysconfig/docker` in your compute images.

It can be updated using `lchroot`.


Job scripts
-----------

Now that the image is ready, we need to create a job script to run our dockerized OSU benchmarks.

A job script would need to include the following commands (besides the usual directives):

- A Docker image name::

    export DOCKER_IMAGE="<docker_image_to_run>"

- An executable application::

    export APPLICATION="<mpi_app>"

And optionally:

- A list of shared folders and mount points::

    export DOCKER_SHARES="/host/share-1:/docker/mnt-1;/host/share-2:/docker/mnt-2;"

- A working directory to which mpi-drun will switch before launching the MPI application::

    export DOCKER_WORKDIR="</path/to/work/dir>"

- A set of options to pass on to the underlying MPI implementation::

    export MPI_OPTS="<options>"

Then finally:

- The ``mpi-drun`` command


Following is an example that can be used to run our previous OSU image::

    #!/bin/bash

    #SBATCH --partition=defq
    #SBATCH --nodes=2
    #SBATCH --ntasks-per-node=1
    #SBATCH --job-name="osu-docker"

    export DOCKER_IMAGE="<controller-hostname>:5000/osu"
    export DOCKER_WORKDIR="/usr/local/libexec/osu-micro-benchmarks/mpi/pt2pt"
    export APPLICATION="osu_latency"
    export MPI_OPTS="-np 2 -mca orte_base_help_aggregate 0"
    mpi-drun

Then, as a user, you can submit the job using sbatch::

    sbatch job.sh

