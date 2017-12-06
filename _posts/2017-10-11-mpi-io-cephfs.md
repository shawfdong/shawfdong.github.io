---
layout: post
title: MPI-IO with CephFS?
tags: [HPC, MPI, Ceph]
---

Does MPI-IO works with CephFS?<!-- more --> Even omniscient Google is befuddled about this question. A quick Google search doesn't turn up any positive results as of this writing.

## NFS
Strictly speaking, NFS is a not a parallel file system, but rather a distributed file system. It is designed to let processes on multiple computers access a common set of files; it is *not* designed to give multiple processes efficient, concurrent access to the same file. Successfully performing MPI-IO on NFS-mounted file systems requires:
1. NFS is at version3;
2. Each NFS shared directory is mounted with the "no attribute caching" (NOACC) option enabled.

But still the performance will be poor!


## CephFS
CephFS is a traditional file system interface with POSIX semantics. And CephFS provides stronger consistency semantics than NFS. Is CephFS a true parallel file system?

### Sample MPI-IO codes
1) Sample MPI-IO C code for writing to a common file using explicit offset (from my course [AMS 250: Introduction to High Performance Computing](https://ams250-spring17-01.courses.soe.ucsc.edu/)):
{% highlight c %}
/* write to a common file using explicit offsets */
#include <stdlib.h>
#include "mpi.h"
#define FILESIZE (1024 * 1024)

int main(int argc, char **argv) {
  int *buf, i, rank, size, bufsize, nints, offset;
  MPI_File fh;
  MPI_Status status;
  MPI_Init(&argc, &argv);
  MPI_Comm_rank(MPI_COMM_WORLD, &rank); MPI_Comm_size(MPI_COMM_WORLD, &size);
  bufsize = FILESIZE/size;
  buf = (int *) malloc(bufsize);
  nints = bufsize/sizeof(int);
  for (i=0; i<nints; i++) buf[i] = rank*nints + i;
  offset = rank*bufsize;
  MPI_File_open(MPI_COMM_WORLD, "datafile", MPI_MODE_CREATE|MPI_MODE_WRONLY,
  MPI_INFO_NULL, &fh);
  MPI_File_write_at(fh, offset, buf, nints, MPI_INT, &status);
  MPI_File_close(&fh); free(buf);
  MPI_Finalize();
  return 0;
}
{% endhighlight %}

2) Sample MPI-IO C code for reading from a common file using individual file pointers (from my course [AMS 250: Introduction to High Performance Computing](https://ams250-spring17-01.courses.soe.ucsc.edu/)):
{% highlight c %}
/* read from a common file using individual file pointers */
#include <stdlib.h>
#include "mpi.h"
#define FILESIZE (1024 * 1024)

int main(int argc, char **argv){
  int *buf, rank, size, bufsize, nints;
  MPI_File fh;
  MPI_Status status;
  MPI_Init(&argc, &argv);
  MPI_Comm_rank(MPI_COMM_WORLD, &rank);
  MPI_Comm_size(MPI_COMM_WORLD, &size);
  bufsize = FILESIZE/size;
  buf = (int *) malloc(bufsize);
  nints = bufsize/sizeof(int);
  MPI_File_open(MPI_COMM_WORLD, "datafile", MPI_MODE_RDONLY, MPI_INFO_NULL, &fh)
;
  MPI_File_seek(fh, rank * bufsize, MPI_SEEK_SET);
  MPI_File_read(fh, buf, nints, MPI_INT, &status);
  MPI_File_close(&fh);
  free(buf);
  MPI_Finalize();
  return 0;
}
{% endhighlight %}

### Testing MPI-IO on a single node
Install Open-MPI on the 4-GPU workstation [Hydra]({{ site.baseurl }}{% post_url 2017-7-28-hydra %}):
{% highlight shell_session %}
[root@hydra ~]# yum install openmpi openmpi-devel
{% endhighlight %}

Go the Ceph file system:
{% highlight shell_session %}
$ cd /mnt/pulpos/dong
{% endhighlight %}

Compile and run the MPI-IO sample codes:
{% highlight shell_session %}
$ export PATH=/usr/lib64/openmpi/bin:$PATH
$ mpicc readFile1.c -o readFile1.x
$ mpicc writeFile1.c -o writeFile1.x
$ mpirun --mca btl self,sm,tcp -n 4 ./writeFile1.x
$ mpirun --mca btl self,sm,tcp -n 4 ./readFile1.x
{% endhighlight %}
On a single node, MPI-IO works flawlessly with CephFS! Would it still work on multiple nodes? To be continued.
