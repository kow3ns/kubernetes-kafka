# Docker Image
The docker image contained in this repository is comprised of a base Ubuntu 16.04 image using the latest release of the 
OpenJDK JRE based on the 1.8 JVM (JDK 8u111), the latest stable release of Kafka (10.2.1) using Scala 2.11. Ubuntu is a 
much larger image than BusyBox or Alpine, but these images contain mucl or ulibc. This requires a custom version of 
OpenJDK to be built against a libc runtime other than glibc. While there are smaller Kafka images based on Alpine and 
BusyBox, the interactions between Kafka, the JVM, and glibc are better understood and easier to debug.

The image is built such that the Kafka JVM process is designated to run as a non-root user. By default, this user is 
kafka and has UID 1000 and GID 1000. The Kafka package is installed into the /opt/kafka directory, all configuration is 
installed into /opt/kafka/config and all executables are in /opt/kafka/bin. Due to the implementation of the scripts in 
/opt/kafka/bin, it is not feasible to symbolically link them into the /user/bin directory. As such, the /opt/kafka/bin 
directory is added to the PATH environment variable.

## Makefile 
The [makefile](Makefile) contained in the docker directory has three commands.
- The `build` command will build the Docker image locally.
- The `push` command will push the image, provided you have correct permissions, 
to grc.io/containers repository.
- The `all` command will perform the `build` command.