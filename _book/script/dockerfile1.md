```
# GCC support can be specified at major, minor, or micro version
# (e.g. 8, 8.2 or 8.2.0).
# See https://hub.docker.com/r/library/gcc/ for all supported GCC
# tags from Docker Hub.
# See https://docs.docker.com/samples/library/gcc/ for more on how to use this image
FROM gcc:latest

# These commands copy your files into the specified directory in the image
# and set that as the working location
RUN mkdir /usr/local/rush-rpc
# COPY command
# 1、如果源路径是个文件，且目标路径是以 / 结尾， 则docker会把目标路径当作一个目录，会把源文件拷贝到该目录下。
# 2、如果源路径是个文件，且目标路径是不是以 / 结尾，则docker会把目标路径当作一个文件。
# 如果目标文件实际是个存在的目录，则会源文件拷贝到该目录下。 注意，这种情况下，最好显示的以 / 结尾，以避免混淆。
# 3、如果源路径是个目录，且目标路径不存在，则docker会自动以目标路径创建一个目录，把源路径目录下的文件拷贝进来。
# 如果目标路径是个已经存在的目录，则docker会把源路径目录下的文件拷贝到该目录下。
COPY ./src /usr/local/rush-rpc
WORKDIR /usr/local/rush-rpc

RUN apt-get install -y make wget curl openssl \ 
    autoconf automake libtool unzip && \
    wget http://sourceforge.net/projects/boost/files/boost/1.60.0/boost_1_60_0.tar.gz && \
    tar -xzvf boost_1_60_0.tar.gz && \
    cd boost_1_60_0 && \
    ./bootstrap.sh --prefix=/usr/local && \
    ./b2 install --with=all && \
    cd .. && mv boost_1_60_0 /usr/local && \
    cd /usr/local/rush-rpc && git clone https://github.com/google/protobuf && \
    cd protobuf && \
    ./autogen.sh && \
    ./configure && \
    make && \
    make install && \
    ldconfig && \
    cd /usr/local/rush-rpc && git clone https://github.com/redis/hiredis.git && \
    cd hiredis && \
    make && \
    make install

ENV PATH $PATH:/usr/local/boost_1_60_0
ENV PATH $PATH:/usr/local/boost_1_60_0/stage/lib
# if ENV invalid, use export
#export   CPLUS_INCLUDE_PATH=$CPLUS_INCLUDE_PATH:/usr/local/boost_1_60_0
#export   LIBRARY_PATH=$LIBRARY_PATH:/usr/local/boost_1_60_0/stage/lib

# This command compiles your app using GCC, adjust for your source code
RUN cd /usr/local/rush-rpc && make

# This command runs your application, comment out this line to compile only
CMD ["./rpc_server_test"]

LABEL Name=RushRPC Version=0.0.1
```