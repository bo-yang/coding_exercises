v1_need_build/malloc.cc                                                                             000755  000765  000024  00000012306 13334721342 015760  0                                                                                                    ustar 00bobyan                          staff                           000000  000000                                                                                                                                                                         #include <cstdio>
#include <cstring>
#include <string>
#include <vector>
#include <signal.h>
#include "malloc.h"
#include "my_malloc.h"
#include "readwritelock.hh"

Allocation allocs;  // current allocations
OverallStat ovstat; // record overall stats
ReadWriteLock rwlck;

static inline std::string alloc_src(const char *file, int line)
{
    return std::string(file) + ":" + std::to_string(line);
}

void * my_malloc(const char *file, int line, size_t size, func_type_t func_id)
{
    rwlck.write_lock();

    void * ptr = (void*)new char[size];
    if (!ptr) {
        rwlck.write_unlock();
        return NULL;
    }

    AllocAttrib attr(size, alloc_src(file,line), func_id);
    allocs[ptr] = attr;

    ovstat.alloc_cnt++;
    ovstat.alloc_size += size;

    rwlck.write_unlock();
    return ptr;
}

void * my_calloc(const char *file, int line, size_t num, size_t size, func_type_t func_id)
{
    size_t sz = size * num;
    void * ptr = my_malloc(file, line, sz, func_id);
    if (!ptr)
        return NULL;

    memset(ptr, 0, sz); // initialize all bits to zero
    return ptr;
}

void * my_realloc(const char *file, int line, void *p, size_t size, func_type_t func_id)
{
    rwlck.write_lock();

    AllocAttrib attr;
    if (allocs.count(p) > 0)
        attr = allocs[p];

    delete [] (char*)p; // always free p first
    void * ptr = (void*)new char[size];
    if (!ptr) {
        rwlck.write_unlock();
        return NULL;
    }

    // update overall stats
    ovstat.alloc_cnt++; // TODO: differentiate malloc and realloc
    ovstat.alloc_size += (int)((int)size - (int)attr.size);

    // update the stat of this allocation - keep timestamp unchanged
    attr.size = size;
    attr.src = alloc_src(file,line);
    attr.func = func_id;
    allocs[ptr] = attr;

    rwlck.write_unlock();
    return ptr;
}

void my_free(const char *file, int line, void *p, func_type_t func_id)
{
    rwlck.write_lock();

    if (allocs.count(p) > 0) {
        ovstat.free_cnt++;
        ovstat.free_size += allocs[p].size;
        allocs.erase(p);
    }
    delete [] (char*)p;

    rwlck.write_unlock();
}

static std::vector<size_t> curr_alloc_by_size()
{
    // Count allocations by memory size
    //0 - 4 bytes: ##########
    //4 - 8 bytes:
    //8 - 16 bytes: ####
    //16 - 32 bytes:
    //32 - 64 bytes:
    //64 - 128 bytes:
    //128 - 256 bytes:
    //256 - 512 bytes: ##
    //512 - 1024 bytes: #
    //1024 - 2048 bytes: #
    //2048 - 4096 bytes: #
    //4096 + bytes: #######
    std::vector<size_t> stat(12, 0);
    auto idx = [&](size_t x) -> size_t {
        size_t i = 0, prod = 4;
        if (x <= 4)
            return i;
        if (x >= 4096)
            return stat.size()-1;
        while(prod < x) {
            prod = prod << 1;
            i++;
        }
        return i;
    };

    rwlck.read_lock();
    for (const auto& a : allocs)
        stat[idx(a.second.size)]++;
    rwlck.read_unlock();

    return stat;
}

static std::vector<size_t> curr_alloc_by_age()
{
    //< 1 sec: ###
    //< 10 sec: ##
    //< 100 sec: ##
    //< 1000 sec: #########################
    //> 1000 sec:
    std::vector<size_t> stat(5,0);
    auto idx = [&](time_t end, time_t begin) -> size_t {
        int secs = (int)difftime(end, begin);
        if (secs < 1)
            return 0;
        if (secs >= 1000)
            return stat.size()-1;
        size_t i = 0;
        while (secs > 0) {
            secs /= 10;
            i++;
        }
        return i;
    };

    rwlck.read_lock();
    for (const auto& a : allocs)
        stat[idx(a.second.ts, ovstat.epoch)]++;
    rwlck.read_unlock();

    return stat;
}

void dump_stats()
{
    size_t curr_alloc_size = 0;
    OverallStat tmpstat; // local copy of the overallstats

    rwlck.read_lock();
    tmpstat = ovstat;
    for (const auto& a : allocs)
        curr_alloc_size += a.second.size;
    rwlck.read_unlock();

    time_t t = time(NULL);
    char* ptm = asctime(localtime(&t));
    std::string timestr = std::string(ptm);
    if (!timestr.empty() && timestr[timestr.length()-1] == '\n')
        timestr.erase(timestr.length()-1); // remove new line
    fprintf(stderr, ">>>>>>>>>>>>> %s <<<<<<<<<<<\n", timestr.c_str());
    fprintf(stderr, "Overall stats:\n");
    fprintf(stderr, "%u overall allocations(%u MB) since start\n", tmpstat.alloc_cnt, tmpstat.alloc_size/(1024*1024));
    fprintf(stderr, "%u MB current total allocated size\n", curr_alloc_size/(1024*1024));

    fprintf(stderr, "\nCurrent allocations by size:\n");
    auto alloc_stats = curr_alloc_by_size();
    int prod = 4;
    fprintf(stderr, "%d - %d bytes: %u\n", 0, prod, alloc_stats[0]);
    size_t idx;
    for (idx = 1; idx < alloc_stats.size()-1; ++idx) {
        fprintf(stderr, "%d - %d bytes: %u\n", prod << idx-1, prod << idx, alloc_stats[idx]);
    }
    fprintf(stderr, "%d + bytes: %u\n", prod << (idx-1), alloc_stats[idx]);

    fprintf(stderr,"\nCurrent allocations by age:\n");
    alloc_stats = curr_alloc_by_age();
    prod = 1;
    for (idx = 0; idx < alloc_stats.size()-1; ++idx) {
        fprintf(stderr, "< %d sec: %u\n", prod, alloc_stats[idx]);
        prod *= 10;
    }
    fprintf(stderr, "> %d sec: %u\n", prod, alloc_stats[idx]);
}

void sig_quit_handler(int sig)
{
    fprintf(stderr, "Received signal %d\n", sig);
    dump_stats();
}
                                                                                                                                                                                                                                                                                                                          v1_need_build/malloc.h                                                                              000755  000765  000024  00000002244 13334664462 015633  0                                                                                                    ustar 00bobyan                          staff                           000000  000000                                                                                                                                                                         #ifndef __MALLOC_H__
#define __MALLOC_H__

#include <ctime>
#include <cstdlib>
#include <string>
#include <unordered_map>
#include "my_malloc.h"

// Required Stats:
//  - overall allocations since start
//  - current total allocated memory(in MB)
//  - current allocation by size
//  - current allocatioj by age

// Memory allocation attribution
struct AllocAttrib {
    size_t size;        // in bytes
    time_t ts;     // timestamp
    std::string src;    // source file:line
    func_type_t func;   // function type

    AllocAttrib(size_t sz=0, std::string source="", func_type_t fid=MALLOC_FUNC_UNKNOWN):
        size(sz),src(source), func(fid) {
        ts = std::time(nullptr);
    }
};
typedef std::unordered_map<void*,AllocAttrib> Allocation;

struct OverallStat {
    size_t alloc_cnt;   // number of allocations
    size_t alloc_size;  // allocated memory size
    size_t free_cnt;    // number of frees
    size_t free_size;   // freed memory size
    std::time_t epoch;  // start time stamp

    OverallStat() {
        alloc_cnt = 0;
        alloc_size = 0;
        free_cnt = 0;
        free_size = 0;
        epoch = std::time(nullptr);
    }
};

#endif /*__MALLOC_H__*/
                                                                                                                                                                                                                                                                                                                                                            v1_need_build/my_malloc.h                                                                           000755  000765  000024  00000002263 13334664462 016341  0                                                                                                    ustar 00bobyan                          staff                           000000  000000                                                                                                                                                                         #ifndef __MY_MALLOC_H__
#define __MY_MALLOC_H__

#ifndef DLL_PUBLIC
#define DLL_PUBLIC __attribute__ ((visibility("default")))
#endif

#ifdef __cplusplus
extern "C" {
#endif

typedef enum {
    MALLOC_FUNC_MALLOC = 0,
    MALLOC_FUNC_CALLOC,
    MALLOC_FUNC_REALLOC,
    MALLOC_FUNC_FREE,

    MALLOC_FUNC_UNKNOWN
} func_type_t;

void * my_malloc(const char *file, int line, size_t size, func_type_t func_id);
void * my_calloc(const char *file, int line, size_t num, size_t size, func_type_t func_id);
void * my_realloc(const char *file, int line, void *p, size_t size, func_type_t func_id);
void my_free(const char *file, int line, void *p, func_type_t func_id);
void dump_stats();

#undef malloc
#define malloc(size) \
    my_malloc(__FILE__, __LINE__, (size), MALLOC_FUNC_MALLOC)
#undef calloc
#define calloc(count, size) \
    my_malloc(__FILE__, __LINE__, (count)*(size), MALLOC_FUNC_CALLOC)
#undef realloc
#define realloc(ptr, size) \
    my_realloc(__FILE__, __LINE__, (ptr), (size), MALLOC_FUNC_REALLOC)

#undef free
#define free(ptr) \
    my_free(__FILE__, __LINE__, (ptr), MALLOC_FUNC_FREE)

DLL_PUBLIC void sig_quit_handler(int sig);

#ifdef  __cplusplus
}
#endif
#endif /*__MY_MALLOC_H__*/
                                                                                                                                                                                                                                                                                                                                             v1_need_build/readme.txt                                                                            000644  000765  000024  00000002033 13334723417 016176  0                                                                                                    ustar 00bobyan                          staff                           000000  000000                                                                                                                                                                         1. The shared library is implemented in malloc.h and malloc.cc. my_malloc.h
defines the public interfaces of the shared library libmy_malloc.so.

2. A read-write lock(write-preference) is also implemented in my shared
library to support multi-threading.

3. The test program is implemented in test.cc, with 3 threads for memory
allocation and free, respectively.

4. To try out my code, please
    make
    ./my_malloc_test

Then in a new terminal run command
    kill -SIGQUIT `pidof my_malloc_test`
.

The sample output is like:

>>>>>>>>>>>>> Tue Aug 14 21:17:04 2018 <<<<<<<<<<<
Overall stats:
103831 overall allocations(246 MB) since start
93 MB current total allocated size

Current allocations by size:
0 - 4 bytes: 34
4 - 8 bytes: 23
8 - 16 bytes: 53
16 - 32 bytes: 118
32 - 64 bytes: 267
64 - 128 bytes: 499
128 - 256 bytes: 1035
256 - 512 bytes: 1999
512 - 1024 bytes: 4015
1024 - 2048 bytes: 7963
2048 - 4096 bytes: 16134
4096 + bytes: 6992

Current allocations by age:
< 1 sec: 0
< 10 sec: 0
< 100 sec: 0
< 1000 sec: 39132
> 10000 sec: 0

                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     v1_need_build/._readwritelock.hh                                                                    000755  000765  000024  00000000324 13334664505 017603  0                                                                                                    ustar 00bobyan                          staff                           000000  000000                                                                                                                                                                             Mac OS X            	   2   �      �                                      ATTR       �   �   <                  �   <  com.apple.quarantine q/0081;5b730333;Chrome;89CC3260-3CA2-4964-A6F2-3A4024E3B87D                                                                                                                                                                                                                                                                                                             v1_need_build/readwritelock.hh                                                                      000755  000765  000024  00000003271 13334664505 017372  0                                                                                                    ustar 00bobyan                          staff                           000000  000000                                                                                                                                                                         #ifndef __READ_WRITE_LOCK_HH__
#define __READ_WRITE_LOCK_HH__

#include <mutex>
#include <condition_variable>

// A write-preference Read-Write lock
class ReadWriteLock {
    public:
        ReadWriteLock() {
            _nread = _nread_waiters = 0;
            _nwrite = _nwrite_waiters = 0;
        }

        void read_lock() {
            std::unique_lock<std::mutex> lck(_mtx);
            if (_nwrite || _nwrite_waiters) {
                _nread_waiters++;
                while (_nwrite || _nwrite_waiters)
                    _rcond.wait(lck); // calls lck.unlock() inherently, lck.lock() is called after notified.
                _nread_waiters--;
            }
            _nread++;
        }

        void read_unlock() {
            std::unique_lock<std::mutex> lck(_mtx);
            _nread--;
            if (_nwrite_waiters)
                _wcond.notify_all();
        }

        void write_lock() {
            std::unique_lock<std::mutex> lck(_mtx);
            if (_nread || _nwrite) {
                _nwrite_waiters++;
                while (_nread || _nwrite)
                    _wcond.wait(lck);
                _nwrite_waiters--;
            }
            _nwrite++;
        }

        void write_unlock() {
            std::unique_lock<std::mutex> lck(_mtx);
            _nwrite--;
            if (_nwrite_waiters) // write-preference
                _wcond.notify_all();
            else if (_nread_waiters)
                _rcond.notify_all();
        }

    private:
        std::mutex _mtx;
        std::condition_variable _rcond;
        std::condition_variable _wcond;
        uint32_t _nread, _nread_waiters;
        uint32_t _nwrite, _nwrite_waiters;
};

#endif //__READ_WRITE_LOCK_HH__
                                                                                                                                                                                                                                                                                                                                       v1_need_build/._readwritelock_test.cc                                                               000755  000765  000024  00000000324 13334664467 020637  0                                                                                                    ustar 00bobyan                          staff                           000000  000000                                                                                                                                                                             Mac OS X            	   2   �      �                                      ATTR       �   �   <                  �   <  com.apple.quarantine q/0081;5b730333;Chrome;89CC3260-3CA2-4964-A6F2-3A4024E3B87D                                                                                                                                                                                                                                                                                                             v1_need_build/readwritelock_test.cc                                                                 000755  000765  000024  00000004674 13334664467 020436  0                                                                                                    ustar 00bobyan                          staff                           000000  000000                                                                                                                                                                         #include <iostream>
#include <thread>
#include <list>
#include <vector>
#include <chrono>
#include <cstdlib>
#include <ctime>
#include "readwritelock.hh"

ReadWriteLock rwlck;
const size_t buffer_size = 20;
std::list<int> cargo;
const int max_cargo = 10000;
int total_cargo = 0;

void consumer() {
    bool iamdone = false;
    while (true) {
        rwlck.write_lock();
        if (cargo.size()) {
            std::cout << cargo.front() << " consumed by " << std::this_thread::get_id() << std::endl;
            cargo.pop_front();
        }
        iamdone = cargo.empty() && (total_cargo >= max_cargo);
        rwlck.write_unlock();

        if (iamdone)
            break;

        std::this_thread::sleep_for (std::chrono::milliseconds(rand()%15+1));
    }
}

void producer(int start, int cnt) {
    int i = start;
    while (i < start+cnt) {
        rwlck.write_lock();
        if (cargo.size() < buffer_size) {
            std::cout << i << " produced by " << std::this_thread::get_id() << std::endl;
            cargo.push_back(i++);
            total_cargo++;
        }
        rwlck.write_unlock();
        std::this_thread::sleep_for (std::chrono::milliseconds(rand()%15+1));
    }
}

void watcher() {
    bool iamdone = false;
    while (true) {
        rwlck.read_lock();
        if (total_cargo % 10 == 0) {
            std::cout << std::this_thread::get_id() << ": total_cargo = " << total_cargo << std::endl;
            for (auto c : cargo)
                std::cout << c << " ";
            std::cout << std::endl;
        }
        iamdone = (total_cargo >= max_cargo);
        rwlck.read_unlock();

        if (iamdone)
            break;

        std::this_thread::sleep_for (std::chrono::milliseconds(rand()%8+1));
    }
}

int main()
{
    srand(time(NULL));
    const int prod_threads = 5;
    const int cons_threads = 3;
    const int watch_threads = 2;
    std::vector<std::thread> consumers, producers, watchers;
    for (int i = 0; i < prod_threads; ++i)
        producers.push_back(std::thread(producer, i*max_cargo/prod_threads, max_cargo/prod_threads));
    for (int i = 0; i < cons_threads; ++i)
        consumers.push_back(std::thread(consumer));
    for (int i = 0; i < watch_threads; ++i)
        watchers.push_back(std::thread(watcher));

    for (int i = 0; i < prod_threads; ++i)
        producers[i].join();
    for (int i = 0; i < cons_threads; ++i)
        consumers[i].join();
    for (int i = 0; i < watch_threads; ++i)
        watchers[i].join();
}
                                                                    v1_need_build/test.cc                                                                               000755  000765  000024  00000003561 13334664467 015511  0                                                                                                    ustar 00bobyan                          staff                           000000  000000                                                                                                                                                                         #include <iostream>
#include <cstdlib>
#include <ctime>
#include <cmath>
#include <list>
#include <thread>
#include <mutex>
#include <chrono>
#include <signal.h>
#include "my_malloc.h"

std::mutex mtx;
std::list<void*> pointer_buffer;
size_t alloc_cnt = 0;
size_t free_cnt = 0;

void memory_allocater()
{
    while(true) {
        int sz = rand()%5000+1;
        void * ptr = malloc(sz);
        if (ptr != NULL) {
            mtx.lock();
            pointer_buffer.push_back(ptr);
            alloc_cnt++;
            mtx.unlock();
        }
        if (alloc_cnt % 1000 == 0)
            std::cout << std::this_thread::get_id() << ": " << alloc_cnt << " allocated." << std::endl;
        std::this_thread::sleep_for (std::chrono::milliseconds(rand()%50+1));
    }
}

void memory_freer()
{
    while(true) {
        mtx.lock();
        if (pointer_buffer.size()) {
            auto ptr = pointer_buffer.front();
            free(ptr);
            pointer_buffer.pop_front();
            free_cnt++;
        }
        mtx.unlock();
        if (free_cnt % 1000 == 0)
            std::cout << std::this_thread::get_id() << ": " << free_cnt << " freed." << std::endl;
        std::this_thread::sleep_for (std::chrono::milliseconds(rand()%80+1));
    }
}

int main()
{
    srand(time(NULL));

    struct sigaction sa;
    sa.sa_flags = 0;
    sigemptyset(&sa.sa_mask);
    sa.sa_handler = sig_quit_handler;
    if (sigaction(SIGQUIT, &sa, NULL) == -1) {
        perror("Failed to set SIGQUIT handler.");
        exit(1);
    }

    const int nthreads = 3;
    std::list<std::thread> thread_alloc, thread_free;
    for (int i = 0; i < nthreads; ++i) {
        thread_alloc.push_back(std::thread(memory_allocater));
        thread_free.push_back(std::thread(memory_freer));
    }

    for (auto& thrd : thread_alloc)
        thrd.join();
    for (auto& thrd : thread_free)
        thrd.join();

    return 0;
}
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               