# \
cc = gcc \
program = rpc \
depends = $(shell find ./ -name *.h) \
source = $(shell find ./ -name *.cpp) \
obj = $(patsubst %.cpp,%.o,%(source)) \
\
$(program): $(obj) \
	$(cc) -o $(program) $(obj) \
\
%.o: $c $(depends) \
	$(cc) -c $< -o $@ \
\
clean: \
	rm -rf $(obj) $(program) \

# abspath 求绝对路径
# word(nums,text) 获取文本的第nums个单词
# words(text) 求text的单词数
# MAKEFILE_LIST 指向当前makefile的宏 多makefile编译时由incude控制指向
# word(words(MAKEFILE_LIST),$(MAKEFILE_LIST)) 指获取当前makefile的文件名
where-am-i = $(abspath $(word $(words $(MAKEFILE_LIST))),$(MAKEFILE_LIST))
# call(func,$1,$2)传参
ROOT = $(dir $(call where-am-i))
CC = g++
# -I 指定 除默认头文件路径/usr/include和/usr/local/include外的用户自定义头文件夹
CXXFLAGS = -std=c++11 -g -I $(ROOT) -pthread
LDFLAGS = -lpthread -lprotobuf -lz -lhiredis -lssl -lcrypto
# wildcard 使能通配符*
SOURCE = $(wildcard base/*.cpp net/*cpp rpc/*cpp)
MAINSOURCE = $(wildcard base/*cpp net/*cpp)
BINARIES = client_test server_test
TARGET = rpc_client_test rpc_server_test
# patsubst(a,b,file)将file中的所有符合条件a的单词换成b
OBJS = $(patsubst %.cpp,%.o,$(SOURCE))

# target: 依赖项 \
[tab]commond
.PHONY: clean all

all: $(BINARIES) $(TARGET)
# filter(pattern,text) 删除text中不符合pattern的单词
# $@ 代指target
# $< 代指当前语句依赖第一项
# $^  代指当前语句所有依赖
$(BINARIES):
	$(CC) $(CXXFLAGS) -o $@ $(MAINSOURCE) $(filter %.cpp,$^) $(LDFLAGS)

$(TARGET):
	$(CC) $(CXXFLAGS) -o $@ $(SOURCE) $(filter %.cpp,$^) $(LDFLAGS)

debug:
	@echo $(SOURCE)

clean:
	rm -f $(BINARIES) $(TARGET) core

client_test: test/client_test.cpp
server_test: test/server_test.cpp
rpc_client_test: test/rpc_client_test.cpp
rpc_server_test: test/rpc_server_test.cpp