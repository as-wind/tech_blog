## gperf
http://www.wyqbupt.com/2017/06/09/%E5%86%85%E5%AD%98%E6%8E%92%E6%9F%A5%E6%80%BB%E7%BB%93/
https://linux.die.net/man/1/pprof 
https://gperftools.github.io/gperftools/heap_checker.html

env HEAPPROFILE="./memcheck/dump" ./bin/retrieval_server --flagfile=/export/servers/retrieval_server/conf/ad_retrieval.gflags &> memcheck/dump_log

env HEAPCHECK=normal  ./bin/retrieval_server --flagfile=/export/servers/retrieval_server/conf/ad_retrieval.gflags

524288000    500M
1048576000  1G
env HEAP_PROFILE_ALLOCATION_INTERVAL=524288000 HEAPPROFILE="./memcheck/dump" ./bin/retrieval_server --flagfile=/export/servers/retrieval_server/conf/ad_retrieval.gflags &> memcheck/dump_log &

对比：
pprof --text --base=dump/dump.1223.heap  ./retrieval_server  dump/dump.1240.heap  > 1223_1240
pprof --text --inuse_space --base=dump/dump.1223.heap  ./retrieval_server  dump/dump.1240.heap  > 1223_1240

pprof --text ./server_debug test.log.0257.heap > text257
pprof --alloc_space -cum -svg ./retrieval_server dump/dump.1233.heap > dump.1233.svg


env HEAPPROFILE="./memcheck/dump" ./bin/retrieval_server --flagfile=/export/servers/retrieval_server/conf/ad_retrieval.gflags &> memcheck/dump_log

env HEAPCHECK=normal  ./bin/retrieval_server --flagfile=/export/servers/retrieval_server/conf/ad_retrieval.gflags &> mem_check

