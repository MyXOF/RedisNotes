## 测试框架

// 参考：http://bbs.redis.cn/forum.php?mod=viewthread&tid=563

// http://bbs.redis.cn/forum.php?mod=viewthread&tid=564

// http://bbs.redis.cn/forum.php?mod=viewthread&tid=565

// http://bbs.redis.cn/forum.php?mod=viewthread&tid=571

用于测试的文件一共有这些

1. memtest.c 内存检测
2. redis_benchmark.c 用于redis性能测试的实现。
3. redis_check_aof.c 用于更新日志检查的实现。
4. redis_check_rdb.c 用于本地数据库检查的实现。
5. testhelp.c 一个C风格的小型测试框架。






## memtest.c

```C
#include <stdint.h>
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <assert.h>
#include <limits.h>
#include <errno.h>
#include <termios.h>
#include <sys/ioctl.h>
#if defined(__sun)
#include <stropts.h>
#endif
#include "config.h"

#if (ULONG_MAX == 4294967295UL)
#define MEMTEST_32BIT
#elif (ULONG_MAX == 18446744073709551615ULL)
#define MEMTEST_64BIT
#else
#error "ULONG_MAX value not supported."
#endif

#ifdef MEMTEST_32BIT
#define ULONG_ONEZERO 0xaaaaaaaaUL
#define ULONG_ZEROONE 0x55555555UL
#else
#define ULONG_ONEZERO 0xaaaaaaaaaaaaaaaaUL
#define ULONG_ZEROONE 0x5555555555555555UL
#endif

static struct winsize ws;
size_t progress_printed; /* Printed chars in screen-wide progress bar. */
size_t progress_full; /* How many chars to write to fill the progress bar. */

void memtest_progress_start(char *title, int pass) {
    int j;

    printf("\x1b[H\x1b[2J");    /* Cursor home, clear screen. */
    /* Fill with dots. */
    for (j = 0; j < ws.ws_col*(ws.ws_row-2); j++) printf(".");
    printf("Please keep the test running several minutes per GB of memory.\n");
    printf("Also check http://www.memtest86.com/ and http://pyropus.ca/software/memtester/");
    printf("\x1b[H\x1b[2K");          /* Cursor home, clear current line.  */
    printf("%s [%d]\n", title, pass); /* Print title. */
    progress_printed = 0;
    progress_full = ws.ws_col*(ws.ws_row-3);
    fflush(stdout);
}

void memtest_progress_end(void) {
    printf("\x1b[H\x1b[2J");    /* Cursor home, clear screen. */
}

void memtest_progress_step(size_t curr, size_t size, char c) {
    size_t chars = ((unsigned long long)curr*progress_full)/size, j;

    for (j = 0; j < chars-progress_printed; j++) printf("%c",c);
    progress_printed = chars;
    fflush(stdout);
}

/* Test that addressing is fine. Every location is populated with its own
 * address, and finally verified. This test is very fast but may detect
 * ASAP big issues with the memory subsystem. */
int memtest_addressing(unsigned long *l, size_t bytes, int interactive) {
    unsigned long words = bytes/sizeof(unsigned long);
    unsigned long j, *p;

    /* Fill */
    p = l;
    for (j = 0; j < words; j++) {
        *p = (unsigned long)p;
        p++;
        if ((j & 0xffff) == 0 && interactive)
            memtest_progress_step(j,words*2,'A');
    }
    /* Test */
    p = l;
    for (j = 0; j < words; j++) {
        if (*p != (unsigned long)p) {
            if (interactive) {
                printf("\n*** MEMORY ADDRESSING ERROR: %p contains %lu\n",
                    (void*) p, *p);
                exit(1);
            }
            return 1;
        }
        p++;
        if ((j & 0xffff) == 0 && interactive)
            memtest_progress_step(j+words,words*2,'A');
    }
    return 0;
}

/* Fill words stepping a single page at every write, so we continue to
 * touch all the pages in the smallest amount of time reducing the
 * effectiveness of caches, and making it hard for the OS to transfer
 * pages on the swap.
 *
 * In this test we can't call rand() since the system may be completely
 * unable to handle library calls, so we have to resort to our own
 * PRNG that only uses local state. We use an xorshift* PRNG. */
#define xorshift64star_next() do { \
        rseed ^= rseed >> 12; \
        rseed ^= rseed << 25; \
        rseed ^= rseed >> 27; \
        rout = rseed * UINT64_C(2685821657736338717); \
} while(0)

void memtest_fill_random(unsigned long *l, size_t bytes, int interactive) {
    unsigned long step = 4096/sizeof(unsigned long);
    unsigned long words = bytes/sizeof(unsigned long)/2;
    unsigned long iwords = words/step;  /* words per iteration */
    unsigned long off, w, *l1, *l2;
    uint64_t rseed = UINT64_C(0xd13133de9afdb566); /* Just a random seed. */
    uint64_t rout = 0;

    assert((bytes & 4095) == 0);
    for (off = 0; off < step; off++) {
        l1 = l+off;
        l2 = l1+words;
        for (w = 0; w < iwords; w++) {
            xorshift64star_next();
            *l1 = *l2 = (unsigned long) rout;
            l1 += step;
            l2 += step;
            if ((w & 0xffff) == 0 && interactive)
                memtest_progress_step(w+iwords*off,words,'R');
        }
    }
}

/* Like memtest_fill_random() but uses the two specified values to fill
 * memory, in an alternated way (v1|v2|v1|v2|...) */
void memtest_fill_value(unsigned long *l, size_t bytes, unsigned long v1,
                        unsigned long v2, char sym, int interactive)
{
    unsigned long step = 4096/sizeof(unsigned long);
    unsigned long words = bytes/sizeof(unsigned long)/2;
    unsigned long iwords = words/step;  /* words per iteration */
    unsigned long off, w, *l1, *l2, v;

    assert((bytes & 4095) == 0);
    for (off = 0; off < step; off++) {
        l1 = l+off;
        l2 = l1+words;
        v = (off & 1) ? v2 : v1;
        for (w = 0; w < iwords; w++) {
#ifdef MEMTEST_32BIT
            *l1 = *l2 = ((unsigned long)     v) |
                        (((unsigned long)    v) << 16);
#else
            *l1 = *l2 = ((unsigned long)     v) |
                        (((unsigned long)    v) << 16) |
                        (((unsigned long)    v) << 32) |
                        (((unsigned long)    v) << 48);
#endif
            l1 += step;
            l2 += step;
            if ((w & 0xffff) == 0 && interactive)
                memtest_progress_step(w+iwords*off,words,sym);
        }
    }
}

int memtest_compare(unsigned long *l, size_t bytes, int interactive) {
    unsigned long words = bytes/sizeof(unsigned long)/2;
    unsigned long w, *l1, *l2;

    assert((bytes & 4095) == 0);
    l1 = l;
    l2 = l1+words;
    for (w = 0; w < words; w++) {
        if (*l1 != *l2) {
            if (interactive) {
                printf("\n*** MEMORY ERROR DETECTED: %p != %p (%lu vs %lu)\n",
                    (void*)l1, (void*)l2, *l1, *l2);
                exit(1);
            }
            return 1;
        }
        l1 ++;
        l2 ++;
        if ((w & 0xffff) == 0 && interactive)
            memtest_progress_step(w,words,'=');
    }
    return 0;
}

int memtest_compare_times(unsigned long *m, size_t bytes, int pass, int times,
                          int interactive)
{
    int j;
    int errors = 0;

    for (j = 0; j < times; j++) {
        if (interactive) memtest_progress_start("Compare",pass);
        errors += memtest_compare(m,bytes,interactive);
        if (interactive) memtest_progress_end();
    }
    return errors;
}

/* Test the specified memory. The number of bytes must be multiple of 4096.
 * If interactive is true the program exists with an error and prints
 * ASCII arts to show progresses. Instead when interactive is 0, it can
 * be used as an API call, and returns 1 if memory errors were found or
 * 0 if there were no errors detected. */
int memtest_test(unsigned long *m, size_t bytes, int passes, int interactive) {
    int pass = 0;
    int errors = 0;

    while (pass != passes) {
        pass++;

        if (interactive) memtest_progress_start("Addressing test",pass);
        errors += memtest_addressing(m,bytes,interactive);
        if (interactive) memtest_progress_end();

        if (interactive) memtest_progress_start("Random fill",pass);
        memtest_fill_random(m,bytes,interactive);
        if (interactive) memtest_progress_end();
        errors += memtest_compare_times(m,bytes,pass,4,interactive);

        if (interactive) memtest_progress_start("Solid fill",pass);
        memtest_fill_value(m,bytes,0,(unsigned long)-1,'S',interactive);
        if (interactive) memtest_progress_end();
        errors += memtest_compare_times(m,bytes,pass,4,interactive);

        if (interactive) memtest_progress_start("Checkerboard fill",pass);
        memtest_fill_value(m,bytes,ULONG_ONEZERO,ULONG_ZEROONE,'C',interactive);
        if (interactive) memtest_progress_end();
        errors += memtest_compare_times(m,bytes,pass,4,interactive);
    }
    return errors;
}

/* A version of memtest_test() that tests memory in small pieces
 * in order to restore the memory content at exit.
 *
 * One problem we have with this approach, is that the cache can avoid
 * real memory accesses, and we can't test big chunks of memory at the
 * same time, because we need to backup them on the stack (the allocator
 * may not be usable or we may be already in an out of memory condition).
 * So what we do is to try to trash the cache with useless memory accesses
 * between the fill and compare cycles. */
#define MEMTEST_BACKUP_WORDS (1024*(1024/sizeof(long)))
/* Random accesses of MEMTEST_DECACHE_SIZE are performed at the start and
 * end of the region between fill and compare cycles in order to trash
 * the cache. */
#define MEMTEST_DECACHE_SIZE (1024*8)
int memtest_preserving_test(unsigned long *m, size_t bytes, int passes) {
    unsigned long backup[MEMTEST_BACKUP_WORDS];
    unsigned long *p = m;
    unsigned long *end = (unsigned long*) (((unsigned char*)m)+(bytes-MEMTEST_DECACHE_SIZE));
    size_t left = bytes;
    int errors = 0;

    if (bytes & 4095) return 0; /* Can't test across 4k page boundaries. */
    if (bytes < 4096*2) return 0; /* Can't test a single page. */

    while(left) {
        /* If we have to test a single final page, go back a single page
         * so that we can test two pages, since the code can't test a single
         * page but at least two. */
        if (left == 4096) {
            left += 4096;
            p -= 4096/sizeof(unsigned long);
        }

        int pass = 0;
        size_t len = (left > sizeof(backup)) ? sizeof(backup) : left;

        /* Always test an even number of pages. */
        if (len/4096 % 2) len -= 4096;

        memcpy(backup,p,len); /* Backup. */
        while(pass != passes) {
            pass++;
            errors += memtest_addressing(p,len,0);
            memtest_fill_random(p,len,0);
            if (bytes >= MEMTEST_DECACHE_SIZE) {
                memtest_compare_times(m,MEMTEST_DECACHE_SIZE,pass,1,0);
                memtest_compare_times(end,MEMTEST_DECACHE_SIZE,pass,1,0);
            }
            errors += memtest_compare_times(p,len,pass,4,0);
            memtest_fill_value(p,len,0,(unsigned long)-1,'S',0);
            if (bytes >= MEMTEST_DECACHE_SIZE) {
                memtest_compare_times(m,MEMTEST_DECACHE_SIZE,pass,1,0);
                memtest_compare_times(end,MEMTEST_DECACHE_SIZE,pass,1,0);
            }
            errors += memtest_compare_times(p,len,pass,4,0);
            memtest_fill_value(p,len,ULONG_ONEZERO,ULONG_ZEROONE,'C',0);
            if (bytes >= MEMTEST_DECACHE_SIZE) {
                memtest_compare_times(m,MEMTEST_DECACHE_SIZE,pass,1,0);
                memtest_compare_times(end,MEMTEST_DECACHE_SIZE,pass,1,0);
            }
            errors += memtest_compare_times(p,len,pass,4,0);
        }
        memcpy(p,backup,len); /* Restore. */
        left -= len;
        p += len/sizeof(unsigned long);
    }
    return errors;
}

/* Perform an interactive test allocating the specified number of megabytes. */
void memtest_alloc_and_test(size_t megabytes, int passes) {
    size_t bytes = megabytes*1024*1024;
    unsigned long *m = malloc(bytes);

    if (m == NULL) {
        fprintf(stderr,"Unable to allocate %zu megabytes: %s",
            megabytes, strerror(errno));
        exit(1);
    }
    memtest_test(m,bytes,passes,1);
    free(m);
}

void memtest(size_t megabytes, int passes) {
    if (ioctl(1, TIOCGWINSZ, &ws) == -1) {
        ws.ws_col = 80;
        ws.ws_row = 20;
    }
    memtest_alloc_and_test(megabytes,passes);
    printf("\nYour memory passed this test.\n");
    printf("Please if you are still in doubt use the following two tools:\n");
    printf("1) memtest86: http://www.memtest86.com/\n");
    printf("2) memtester: http://pyropus.ca/software/memtester/\n");
    exit(0);
}

```


## redis_benchmark.c

```C
#include "fmacros.h"

#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <time.h>
#include <sys/time.h>
#include <signal.h>
#include <assert.h>

#include <sds.h> /* Use hiredis sds. */
#include "ae.h"
#include "hiredis.h"
#include "adlist.h"
#include "zmalloc.h"

#define UNUSED(V) ((void) V)
#define RANDPTR_INITIAL_SIZE 8

static struct config {
    aeEventLoop *el;
    const char *hostip;
    int hostport;
    const char *hostsocket;
    int numclients;
    int liveclients;
    int requests;
    int requests_issued;
    int requests_finished;
    int keysize;
    int datasize;
    int randomkeys;
    int randomkeys_keyspacelen;
    int keepalive;
    int pipeline;
    int showerrors;
    long long start;
    long long totlatency;
    long long *latency;
    const char *title;
    list *clients;
    int quiet;
    int csv;
    int loop;
    int idlemode;
    int dbnum;
    sds dbnumstr;
    char *tests;
    char *auth;
} config;

typedef struct _client {
    redisContext *context;
    sds obuf;
    char **randptr;         /* Pointers to :rand: strings inside the command buf */
    size_t randlen;         /* Number of pointers in client->randptr */
    size_t randfree;        /* Number of unused pointers in client->randptr */
    size_t written;         /* Bytes of 'obuf' already written */
    long long start;        /* Start time of a request */
    long long latency;      /* Request latency */
    int pending;            /* Number of pending requests (replies to consume) */
    int prefix_pending;     /* If non-zero, number of pending prefix commands. Commands
                               such as auth and select are prefixed to the pipeline of
                               benchmark commands and discarded after the first send. */
    int prefixlen;          /* Size in bytes of the pending prefix commands */
} *client;

/* Prototypes */
static void writeHandler(aeEventLoop *el, int fd, void *privdata, int mask);
static void createMissingClients(client c);

/* Implementation */
static long long ustime(void) {
    struct timeval tv;
    long long ust;

    gettimeofday(&tv, NULL);
    ust = ((long)tv.tv_sec)*1000000;
    ust += tv.tv_usec;
    return ust;
}

static long long mstime(void) {
    struct timeval tv;
    long long mst;

    gettimeofday(&tv, NULL);
    mst = ((long long)tv.tv_sec)*1000;
    mst += tv.tv_usec/1000;
    return mst;
}

static void freeClient(client c) {
    listNode *ln;
    aeDeleteFileEvent(config.el,c->context->fd,AE_WRITABLE);
    aeDeleteFileEvent(config.el,c->context->fd,AE_READABLE);
    redisFree(c->context);
    sdsfree(c->obuf);
    zfree(c->randptr);
    zfree(c);
    config.liveclients--;
    ln = listSearchKey(config.clients,c);
    assert(ln != NULL);
    listDelNode(config.clients,ln);
}

static void freeAllClients(void) {
    listNode *ln = config.clients->head, *next;

    while(ln) {
        next = ln->next;
        freeClient(ln->value);
        ln = next;
    }
}

static void resetClient(client c) {
    aeDeleteFileEvent(config.el,c->context->fd,AE_WRITABLE);
    aeDeleteFileEvent(config.el,c->context->fd,AE_READABLE);
    aeCreateFileEvent(config.el,c->context->fd,AE_WRITABLE,writeHandler,c);
    c->written = 0;
    c->pending = config.pipeline;
}

static void randomizeClientKey(client c) {
    size_t i;

    for (i = 0; i < c->randlen; i++) {
        char *p = c->randptr[i]+11;
        size_t r = random() % config.randomkeys_keyspacelen;
        size_t j;

        for (j = 0; j < 12; j++) {
            *p = '0'+r%10;
            r/=10;
            p--;
        }
    }
}

static void clientDone(client c) {
    if (config.requests_finished == config.requests) {
        freeClient(c);
        aeStop(config.el);
        return;
    }
    if (config.keepalive) {
        resetClient(c);
    } else {
        config.liveclients--;
        createMissingClients(c);
        config.liveclients++;
        freeClient(c);
    }
}

static void readHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
    client c = privdata;
    void *reply = NULL;
    UNUSED(el);
    UNUSED(fd);
    UNUSED(mask);

    /* Calculate latency only for the first read event. This means that the
     * server already sent the reply and we need to parse it. Parsing overhead
     * is not part of the latency, so calculate it only once, here. */
    if (c->latency < 0) c->latency = ustime()-(c->start);

    if (redisBufferRead(c->context) != REDIS_OK) {
        fprintf(stderr,"Error: %s\n",c->context->errstr);
        exit(1);
    } else {
        while(c->pending) {
            if (redisGetReply(c->context,&reply) != REDIS_OK) {
                fprintf(stderr,"Error: %s\n",c->context->errstr);
                exit(1);
            }
            if (reply != NULL) {
                if (reply == (void*)REDIS_REPLY_ERROR) {
                    fprintf(stderr,"Unexpected error reply, exiting...\n");
                    exit(1);
                }

                if (config.showerrors) {
                    static time_t lasterr_time = 0;
                    time_t now = time(NULL);
                    redisReply *r = reply;
                    if (r->type == REDIS_REPLY_ERROR && lasterr_time != now) {
                        lasterr_time = now;
                        printf("Error from server: %s\n", r->str);
                    }
                }

                freeReplyObject(reply);
                /* This is an OK for prefix commands such as auth and select.*/
                if (c->prefix_pending > 0) {
                    c->prefix_pending--;
                    c->pending--;
                    /* Discard prefix commands on first response.*/
                    if (c->prefixlen > 0) {
                        size_t j;
                        sdsrange(c->obuf, c->prefixlen, -1);
                        /* We also need to fix the pointers to the strings
                        * we need to randomize. */
                        for (j = 0; j < c->randlen; j++)
                            c->randptr[j] -= c->prefixlen;
                        c->prefixlen = 0;
                    }
                    continue;
                }

                if (config.requests_finished < config.requests)
                    config.latency[config.requests_finished++] = c->latency;
                c->pending--;
                if (c->pending == 0) {
                    clientDone(c);
                    break;
                }
            } else {
                break;
            }
        }
    }
}

static void writeHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
    client c = privdata;
    UNUSED(el);
    UNUSED(fd);
    UNUSED(mask);

    /* Initialize request when nothing was written. */
    if (c->written == 0) {
        /* Enforce upper bound to number of requests. */
        if (config.requests_issued++ >= config.requests) {
            freeClient(c);
            return;
        }

        /* Really initialize: randomize keys and set start time. */
        if (config.randomkeys) randomizeClientKey(c);
        c->start = ustime();
        c->latency = -1;
    }

    if (sdslen(c->obuf) > c->written) {
        void *ptr = c->obuf+c->written;
        ssize_t nwritten = write(c->context->fd,ptr,sdslen(c->obuf)-c->written);
        if (nwritten == -1) {
            if (errno != EPIPE)
                fprintf(stderr, "Writing to socket: %s\n", strerror(errno));
            freeClient(c);
            return;
        }
        c->written += nwritten;
        if (sdslen(c->obuf) == c->written) {
            aeDeleteFileEvent(config.el,c->context->fd,AE_WRITABLE);
            aeCreateFileEvent(config.el,c->context->fd,AE_READABLE,readHandler,c);
        }
    }
}

/* Create a benchmark client, configured to send the command passed as 'cmd' of
 * 'len' bytes.
 *
 * The command is copied N times in the client output buffer (that is reused
 * again and again to send the request to the server) accordingly to the configured
 * pipeline size.
 *
 * Also an initial SELECT command is prepended in order to make sure the right
 * database is selected, if needed. The initial SELECT will be discarded as soon
 * as the first reply is received.
 *
 * To create a client from scratch, the 'from' pointer is set to NULL. If instead
 * we want to create a client using another client as reference, the 'from' pointer
 * points to the client to use as reference. In such a case the following
 * information is take from the 'from' client:
 *
 * 1) The command line to use.
 * 2) The offsets of the __rand_int__ elements inside the command line, used
 *    for arguments randomization.
 *
 * Even when cloning another client, prefix commands are applied if needed.*/
static client createClient(char *cmd, size_t len, client from) {
    int j;
    client c = zmalloc(sizeof(struct _client));

    if (config.hostsocket == NULL) {
        c->context = redisConnectNonBlock(config.hostip,config.hostport);
    } else {
        c->context = redisConnectUnixNonBlock(config.hostsocket);
    }
    if (c->context->err) {
        fprintf(stderr,"Could not connect to Redis at ");
        if (config.hostsocket == NULL)
            fprintf(stderr,"%s:%d: %s\n",config.hostip,config.hostport,c->context->errstr);
        else
            fprintf(stderr,"%s: %s\n",config.hostsocket,c->context->errstr);
        exit(1);
    }
    /* Suppress hiredis cleanup of unused buffers for max speed. */
    c->context->reader->maxbuf = 0;

    /* Build the request buffer:
     * Queue N requests accordingly to the pipeline size, or simply clone
     * the example client buffer. */
    c->obuf = sdsempty();
    /* Prefix the request buffer with AUTH and/or SELECT commands, if applicable.
     * These commands are discarded after the first response, so if the client is
     * reused the commands will not be used again. */
    c->prefix_pending = 0;
    if (config.auth) {
        char *buf = NULL;
        int len = redisFormatCommand(&buf, "AUTH %s", config.auth);
        c->obuf = sdscatlen(c->obuf, buf, len);
        free(buf);
        c->prefix_pending++;
    }

    /* If a DB number different than zero is selected, prefix our request
     * buffer with the SELECT command, that will be discarded the first
     * time the replies are received, so if the client is reused the
     * SELECT command will not be used again. */
    if (config.dbnum != 0) {
        c->obuf = sdscatprintf(c->obuf,"*2\r\n$6\r\nSELECT\r\n$%d\r\n%s\r\n",
            (int)sdslen(config.dbnumstr),config.dbnumstr);
        c->prefix_pending++;
    }
    c->prefixlen = sdslen(c->obuf);
    /* Append the request itself. */
    if (from) {
        c->obuf = sdscatlen(c->obuf,
            from->obuf+from->prefixlen,
            sdslen(from->obuf)-from->prefixlen);
    } else {
        for (j = 0; j < config.pipeline; j++)
            c->obuf = sdscatlen(c->obuf,cmd,len);
    }

    c->written = 0;
    c->pending = config.pipeline+c->prefix_pending;
    c->randptr = NULL;
    c->randlen = 0;

    /* Find substrings in the output buffer that need to be randomized. */
    if (config.randomkeys) {
        if (from) {
            c->randlen = from->randlen;
            c->randfree = 0;
            c->randptr = zmalloc(sizeof(char*)*c->randlen);
            /* copy the offsets. */
            for (j = 0; j < (int)c->randlen; j++) {
                c->randptr[j] = c->obuf + (from->randptr[j]-from->obuf);
                /* Adjust for the different select prefix length. */
                c->randptr[j] += c->prefixlen - from->prefixlen;
            }
        } else {
            char *p = c->obuf;

            c->randlen = 0;
            c->randfree = RANDPTR_INITIAL_SIZE;
            c->randptr = zmalloc(sizeof(char*)*c->randfree);
            while ((p = strstr(p,"__rand_int__")) != NULL) {
                if (c->randfree == 0) {
                    c->randptr = zrealloc(c->randptr,sizeof(char*)*c->randlen*2);
                    c->randfree += c->randlen;
                }
                c->randptr[c->randlen++] = p;
                c->randfree--;
                p += 12; /* 12 is strlen("__rand_int__). */
            }
        }
    }
    if (config.idlemode == 0)
        aeCreateFileEvent(config.el,c->context->fd,AE_WRITABLE,writeHandler,c);
    listAddNodeTail(config.clients,c);
    config.liveclients++;
    return c;
}

static void createMissingClients(client c) {
    int n = 0;

    while(config.liveclients < config.numclients) {
        createClient(NULL,0,c);

        /* Listen backlog is quite limited on most systems */
        if (++n > 64) {
            usleep(50000);
            n = 0;
        }
    }
}

static int compareLatency(const void *a, const void *b) {
    return (*(long long*)a)-(*(long long*)b);
}

static void showLatencyReport(void) {
    int i, curlat = 0;
    float perc, reqpersec;

    reqpersec = (float)config.requests_finished/((float)config.totlatency/1000);
    if (!config.quiet && !config.csv) {
        printf("====== %s ======\n", config.title);
        printf("  %d requests completed in %.2f seconds\n", config.requests_finished,
            (float)config.totlatency/1000);
        printf("  %d parallel clients\n", config.numclients);
        printf("  %d bytes payload\n", config.datasize);
        printf("  keep alive: %d\n", config.keepalive);
        printf("\n");

        qsort(config.latency,config.requests,sizeof(long long),compareLatency);
        for (i = 0; i < config.requests; i++) {
            if (config.latency[i]/1000 != curlat || i == (config.requests-1)) {
                curlat = config.latency[i]/1000;
                perc = ((float)(i+1)*100)/config.requests;
                printf("%.2f%% <= %d milliseconds\n", perc, curlat);
            }
        }
        printf("%.2f requests per second\n\n", reqpersec);
    } else if (config.csv) {
        printf("\"%s\",\"%.2f\"\n", config.title, reqpersec);
    } else {
        printf("%s: %.2f requests per second\n", config.title, reqpersec);
    }
}

static void benchmark(char *title, char *cmd, int len) {
    client c;

    config.title = title;
    config.requests_issued = 0;
    config.requests_finished = 0;

    c = createClient(cmd,len,NULL);
    createMissingClients(c);

    config.start = mstime();
    aeMain(config.el);
    config.totlatency = mstime()-config.start;

    showLatencyReport();
    freeAllClients();
}

/* Returns number of consumed options. */
int parseOptions(int argc, const char **argv) {
    int i;
    int lastarg;
    int exit_status = 1;

    for (i = 1; i < argc; i++) {
        lastarg = (i == (argc-1));

        if (!strcmp(argv[i],"-c")) {
            if (lastarg) goto invalid;
            config.numclients = atoi(argv[++i]);
        } else if (!strcmp(argv[i],"-n")) {
            if (lastarg) goto invalid;
            config.requests = atoi(argv[++i]);
        } else if (!strcmp(argv[i],"-k")) {
            if (lastarg) goto invalid;
            config.keepalive = atoi(argv[++i]);
        } else if (!strcmp(argv[i],"-h")) {
            if (lastarg) goto invalid;
            config.hostip = strdup(argv[++i]);
        } else if (!strcmp(argv[i],"-p")) {
            if (lastarg) goto invalid;
            config.hostport = atoi(argv[++i]);
        } else if (!strcmp(argv[i],"-s")) {
            if (lastarg) goto invalid;
            config.hostsocket = strdup(argv[++i]);
        } else if (!strcmp(argv[i],"-a") ) {
            if (lastarg) goto invalid;
            config.auth = strdup(argv[++i]);
        } else if (!strcmp(argv[i],"-d")) {
            if (lastarg) goto invalid;
            config.datasize = atoi(argv[++i]);
            if (config.datasize < 1) config.datasize=1;
            if (config.datasize > 1024*1024*1024) config.datasize = 1024*1024*1024;
        } else if (!strcmp(argv[i],"-P")) {
            if (lastarg) goto invalid;
            config.pipeline = atoi(argv[++i]);
            if (config.pipeline <= 0) config.pipeline=1;
        } else if (!strcmp(argv[i],"-r")) {
            if (lastarg) goto invalid;
            config.randomkeys = 1;
            config.randomkeys_keyspacelen = atoi(argv[++i]);
            if (config.randomkeys_keyspacelen < 0)
                config.randomkeys_keyspacelen = 0;
        } else if (!strcmp(argv[i],"-q")) {
            config.quiet = 1;
        } else if (!strcmp(argv[i],"--csv")) {
            config.csv = 1;
        } else if (!strcmp(argv[i],"-l")) {
            config.loop = 1;
        } else if (!strcmp(argv[i],"-I")) {
            config.idlemode = 1;
        } else if (!strcmp(argv[i],"-e")) {
            config.showerrors = 1;
        } else if (!strcmp(argv[i],"-t")) {
            if (lastarg) goto invalid;
            /* We get the list of tests to run as a string in the form
             * get,set,lrange,...,test_N. Then we add a comma before and
             * after the string in order to make sure that searching
             * for ",testname," will always get a match if the test is
             * enabled. */
            config.tests = sdsnew(",");
            config.tests = sdscat(config.tests,(char*)argv[++i]);
            config.tests = sdscat(config.tests,",");
            sdstolower(config.tests);
        } else if (!strcmp(argv[i],"--dbnum")) {
            if (lastarg) goto invalid;
            config.dbnum = atoi(argv[++i]);
            config.dbnumstr = sdsfromlonglong(config.dbnum);
        } else if (!strcmp(argv[i],"--help")) {
            exit_status = 0;
            goto usage;
        } else {
            /* Assume the user meant to provide an option when the arg starts
             * with a dash. We're done otherwise and should use the remainder
             * as the command and arguments for running the benchmark. */
            if (argv[i][0] == '-') goto invalid;
            return i;
        }
    }

    return i;

invalid:
    printf("Invalid option \"%s\" or option argument missing\n\n",argv[i]);

usage:
    printf(
"Usage: redis-benchmark [-h <host>] [-p <port>] [-c <clients>] [-n <requests>] [-k <boolean>]\n\n"
" -h <hostname>      Server hostname (default 127.0.0.1)\n"
" -p <port>          Server port (default 6379)\n"
" -s <socket>        Server socket (overrides host and port)\n"
" -a <password>      Password for Redis Auth\n"
" -c <clients>       Number of parallel connections (default 50)\n"
" -n <requests>      Total number of requests (default 100000)\n"
" -d <size>          Data size of SET/GET value in bytes (default 2)\n"
" -dbnum <db>        SELECT the specified db number (default 0)\n"
" -k <boolean>       1=keep alive 0=reconnect (default 1)\n"
" -r <keyspacelen>   Use random keys for SET/GET/INCR, random values for SADD\n"
"  Using this option the benchmark will expand the string __rand_int__\n"
"  inside an argument with a 12 digits number in the specified range\n"
"  from 0 to keyspacelen-1. The substitution changes every time a command\n"
"  is executed. Default tests use this to hit random keys in the\n"
"  specified range.\n"
" -P <numreq>        Pipeline <numreq> requests. Default 1 (no pipeline).\n"
" -e                 If server replies with errors, show them on stdout.\n"
"                    (no more than 1 error per second is displayed)\n"
" -q                 Quiet. Just show query/sec values\n"
" --csv              Output in CSV format\n"
" -l                 Loop. Run the tests forever\n"
" -t <tests>         Only run the comma separated list of tests. The test\n"
"                    names are the same as the ones produced as output.\n"
" -I                 Idle mode. Just open N idle connections and wait.\n\n"
"Examples:\n\n"
" Run the benchmark with the default configuration against 127.0.0.1:6379:\n"
"   $ redis-benchmark\n\n"
" Use 20 parallel clients, for a total of 100k requests, against 192.168.1.1:\n"
"   $ redis-benchmark -h 192.168.1.1 -p 6379 -n 100000 -c 20\n\n"
" Fill 127.0.0.1:6379 with about 1 million keys only using the SET test:\n"
"   $ redis-benchmark -t set -n 1000000 -r 100000000\n\n"
" Benchmark 127.0.0.1:6379 for a few commands producing CSV output:\n"
"   $ redis-benchmark -t ping,set,get -n 100000 --csv\n\n"
" Benchmark a specific command line:\n"
"   $ redis-benchmark -r 10000 -n 10000 eval 'return redis.call(\"ping\")' 0\n\n"
" Fill a list with 10000 random elements:\n"
"   $ redis-benchmark -r 10000 -n 10000 lpush mylist __rand_int__\n\n"
" On user specified command lines __rand_int__ is replaced with a random integer\n"
" with a range of values selected by the -r option.\n"
    );
    exit(exit_status);
}

int showThroughput(struct aeEventLoop *eventLoop, long long id, void *clientData) {
    UNUSED(eventLoop);
    UNUSED(id);
    UNUSED(clientData);

    if (config.liveclients == 0) {
        fprintf(stderr,"All clients disconnected... aborting.\n");
        exit(1);
    }
    if (config.csv) return 250;
    if (config.idlemode == 1) {
        printf("clients: %d\r", config.liveclients);
        fflush(stdout);
	return 250;
    }
    float dt = (float)(mstime()-config.start)/1000.0;
    float rps = (float)config.requests_finished/dt;
    printf("%s: %.2f\r", config.title, rps);
    fflush(stdout);
    return 250; /* every 250ms */
}

/* Return true if the named test was selected using the -t command line
 * switch, or if all the tests are selected (no -t passed by user). */
int test_is_selected(char *name) {
    char buf[256];
    int l = strlen(name);

    if (config.tests == NULL) return 1;
    buf[0] = ',';
    memcpy(buf+1,name,l);
    buf[l+1] = ',';
    buf[l+2] = '\0';
    return strstr(config.tests,buf) != NULL;
}

int main(int argc, const char **argv) {
    int i;
    char *data, *cmd;
    int len;

    client c;

    srandom(time(NULL));
    signal(SIGHUP, SIG_IGN);
    signal(SIGPIPE, SIG_IGN);

    config.numclients = 50;
    config.requests = 100000;
    config.liveclients = 0;
    config.el = aeCreateEventLoop(1024*10);
    aeCreateTimeEvent(config.el,1,showThroughput,NULL,NULL);
    config.keepalive = 1;
    config.datasize = 3;
    config.pipeline = 1;
    config.showerrors = 0;
    config.randomkeys = 0;
    config.randomkeys_keyspacelen = 0;
    config.quiet = 0;
    config.csv = 0;
    config.loop = 0;
    config.idlemode = 0;
    config.latency = NULL;
    config.clients = listCreate();
    config.hostip = "127.0.0.1";
    config.hostport = 6379;
    config.hostsocket = NULL;
    config.tests = NULL;
    config.dbnum = 0;
    config.auth = NULL;

    i = parseOptions(argc,argv);
    argc -= i;
    argv += i;

    config.latency = zmalloc(sizeof(long long)*config.requests);

    if (config.keepalive == 0) {
        printf("WARNING: keepalive disabled, you probably need 'echo 1 > /proc/sys/net/ipv4/tcp_tw_reuse' for Linux and 'sudo sysctl -w net.inet.tcp.msl=1000' for Mac OS X in order to use a lot of clients/requests\n");
    }

    if (config.idlemode) {
        printf("Creating %d idle connections and waiting forever (Ctrl+C when done)\n", config.numclients);
        c = createClient("",0,NULL); /* will never receive a reply */
        createMissingClients(c);
        aeMain(config.el);
        /* and will wait for every */
    }

    /* Run benchmark with command in the remainder of the arguments. */
    if (argc) {
        sds title = sdsnew(argv[0]);
        for (i = 1; i < argc; i++) {
            title = sdscatlen(title, " ", 1);
            title = sdscatlen(title, (char*)argv[i], strlen(argv[i]));
        }

        do {
            len = redisFormatCommandArgv(&cmd,argc,argv,NULL);
            benchmark(title,cmd,len);
            free(cmd);
        } while(config.loop);

        return 0;
    }

    /* Run default benchmark suite. */
    data = zmalloc(config.datasize+1);
    do {
        memset(data,'x',config.datasize);
        data[config.datasize] = '\0';

        if (test_is_selected("ping_inline") || test_is_selected("ping"))
            benchmark("PING_INLINE","PING\r\n",6);

        if (test_is_selected("ping_mbulk") || test_is_selected("ping")) {
            len = redisFormatCommand(&cmd,"PING");
            benchmark("PING_BULK",cmd,len);
            free(cmd);
        }

        if (test_is_selected("set")) {
            len = redisFormatCommand(&cmd,"SET key:__rand_int__ %s",data);
            benchmark("SET",cmd,len);
            free(cmd);
        }

        if (test_is_selected("get")) {
            len = redisFormatCommand(&cmd,"GET key:__rand_int__");
            benchmark("GET",cmd,len);
            free(cmd);
        }

        if (test_is_selected("incr")) {
            len = redisFormatCommand(&cmd,"INCR counter:__rand_int__");
            benchmark("INCR",cmd,len);
            free(cmd);
        }

        if (test_is_selected("lpush")) {
            len = redisFormatCommand(&cmd,"LPUSH mylist %s",data);
            benchmark("LPUSH",cmd,len);
            free(cmd);
        }

        if (test_is_selected("rpush")) {
            len = redisFormatCommand(&cmd,"RPUSH mylist %s",data);
            benchmark("RPUSH",cmd,len);
            free(cmd);
        }

        if (test_is_selected("lpop")) {
            len = redisFormatCommand(&cmd,"LPOP mylist");
            benchmark("LPOP",cmd,len);
            free(cmd);
        }

        if (test_is_selected("rpop")) {
            len = redisFormatCommand(&cmd,"RPOP mylist");
            benchmark("RPOP",cmd,len);
            free(cmd);
        }

        if (test_is_selected("sadd")) {
            len = redisFormatCommand(&cmd,
                "SADD myset element:__rand_int__");
            benchmark("SADD",cmd,len);
            free(cmd);
        }

        if (test_is_selected("spop")) {
            len = redisFormatCommand(&cmd,"SPOP myset");
            benchmark("SPOP",cmd,len);
            free(cmd);
        }

        if (test_is_selected("lrange") ||
            test_is_selected("lrange_100") ||
            test_is_selected("lrange_300") ||
            test_is_selected("lrange_500") ||
            test_is_selected("lrange_600"))
        {
            len = redisFormatCommand(&cmd,"LPUSH mylist %s",data);
            benchmark("LPUSH (needed to benchmark LRANGE)",cmd,len);
            free(cmd);
        }

        if (test_is_selected("lrange") || test_is_selected("lrange_100")) {
            len = redisFormatCommand(&cmd,"LRANGE mylist 0 99");
            benchmark("LRANGE_100 (first 100 elements)",cmd,len);
            free(cmd);
        }

        if (test_is_selected("lrange") || test_is_selected("lrange_300")) {
            len = redisFormatCommand(&cmd,"LRANGE mylist 0 299");
            benchmark("LRANGE_300 (first 300 elements)",cmd,len);
            free(cmd);
        }

        if (test_is_selected("lrange") || test_is_selected("lrange_500")) {
            len = redisFormatCommand(&cmd,"LRANGE mylist 0 449");
            benchmark("LRANGE_500 (first 450 elements)",cmd,len);
            free(cmd);
        }

        if (test_is_selected("lrange") || test_is_selected("lrange_600")) {
            len = redisFormatCommand(&cmd,"LRANGE mylist 0 599");
            benchmark("LRANGE_600 (first 600 elements)",cmd,len);
            free(cmd);
        }

        if (test_is_selected("mset")) {
            const char *argv[21];
            argv[0] = "MSET";
            for (i = 1; i < 21; i += 2) {
                argv[i] = "key:__rand_int__";
                argv[i+1] = data;
            }
            len = redisFormatCommandArgv(&cmd,21,argv,NULL);
            benchmark("MSET (10 keys)",cmd,len);
            free(cmd);
        }

        if (!config.csv) printf("\n");
    } while(config.loop);

    return 0;
}

```

## redis_check_aof.c

```C
#include "fmacros.h"
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <sys/stat.h>
#include "config.h"

#define ERROR(...) { \
    char __buf[1024]; \
    sprintf(__buf, __VA_ARGS__); \
    sprintf(error, "0x%16llx: %s", (long long)epos, __buf); \
}

static char error[1024];
static off_t epos;

int consumeNewline(char *buf) {
    if (strncmp(buf,"\r\n",2) != 0) {
        ERROR("Expected \\r\\n, got: %02x%02x",buf[0],buf[1]);
        return 0;
    }
    return 1;
}

int readLong(FILE *fp, char prefix, long *target) {
    char buf[128], *eptr;
    epos = ftello(fp);
    if (fgets(buf,sizeof(buf),fp) == NULL) {
        return 0;
    }
    if (buf[0] != prefix) {
        ERROR("Expected prefix '%c', got: '%c'",buf[0],prefix);
        return 0;
    }
    *target = strtol(buf+1,&eptr,10);
    return consumeNewline(eptr);
}

int readBytes(FILE *fp, char *target, long length) {
    long real;
    epos = ftello(fp);
    real = fread(target,1,length,fp);
    if (real != length) {
        ERROR("Expected to read %ld bytes, got %ld bytes",length,real);
        return 0;
    }
    return 1;
}

int readString(FILE *fp, char** target) {
    long len;
    *target = NULL;
    if (!readLong(fp,'$',&len)) {
        return 0;
    }

    /* Increase length to also consume \r\n */
    len += 2;
    *target = (char*)malloc(len);
    if (!readBytes(fp,*target,len)) {
        return 0;
    }
    if (!consumeNewline(*target+len-2)) {
        return 0;
    }
    (*target)[len-2] = '\0';
    return 1;
}

int readArgc(FILE *fp, long *target) {
    return readLong(fp,'*',target);
}

off_t process(FILE *fp) {
    long argc;
    off_t pos = 0;
    int i, multi = 0;
    char *str;

    while(1) {
        if (!multi) pos = ftello(fp);
        if (!readArgc(fp, &argc)) break;

        for (i = 0; i < argc; i++) {
            if (!readString(fp,&str)) break;
            if (i == 0) {
                if (strcasecmp(str, "multi") == 0) {
                    if (multi++) {
                        ERROR("Unexpected MULTI");
                        break;
                    }
                } else if (strcasecmp(str, "exec") == 0) {
                    if (--multi) {
                        ERROR("Unexpected EXEC");
                        break;
                    }
                }
            }
            free(str);
        }

        /* Stop if the loop did not finish */
        if (i < argc) {
            if (str) free(str);
            break;
        }
    }

    if (feof(fp) && multi && strlen(error) == 0) {
        ERROR("Reached EOF before reading EXEC for MULTI");
    }
    if (strlen(error) > 0) {
        printf("%s\n", error);
    }
    return pos;
}

int main(int argc, char **argv) {
    char *filename;
    int fix = 0;

    if (argc < 2) {
        printf("Usage: %s [--fix] <file.aof>\n", argv[0]);
        exit(1);
    } else if (argc == 2) {
        filename = argv[1];
    } else if (argc == 3) {
        if (strcmp(argv[1],"--fix") != 0) {
            printf("Invalid argument: %s\n", argv[1]);
            exit(1);
        }
        filename = argv[2];
        fix = 1;
    } else {
        printf("Invalid arguments\n");
        exit(1);
    }

    FILE *fp = fopen(filename,"r+");
    if (fp == NULL) {
        printf("Cannot open file: %s\n", filename);
        exit(1);
    }

    struct redis_stat sb;
    if (redis_fstat(fileno(fp),&sb) == -1) {
        printf("Cannot stat file: %s\n", filename);
        exit(1);
    }

    off_t size = sb.st_size;
    if (size == 0) {
        printf("Empty file: %s\n", filename);
        exit(1);
    }

    off_t pos = process(fp);
    off_t diff = size-pos;
    printf("AOF analyzed: size=%lld, ok_up_to=%lld, diff=%lld\n",
        (long long) size, (long long) pos, (long long) diff);
    if (diff > 0) {
        if (fix) {
            char buf[2];
            printf("This will shrink the AOF from %lld bytes, with %lld bytes, to %lld bytes\n",(long long)size,(long long)diff,(long long)pos);
            printf("Continue? [y/N]: ");
            if (fgets(buf,sizeof(buf),stdin) == NULL ||
                strncasecmp(buf,"y",1) != 0) {
                    printf("Aborting...\n");
                    exit(1);
            }
            if (ftruncate(fileno(fp), pos) == -1) {
                printf("Failed to truncate AOF\n");
                exit(1);
            } else {
                printf("Successfully truncated AOF\n");
            }
        } else {
            printf("AOF is not valid\n");
            exit(1);
        }
    } else {
        printf("AOF is valid\n");
    }

    fclose(fp);
    return 0;
}

```

## redis_check_rdb.c

```C

#include "server.h"
#include "rdb.h"

#include <stdarg.h>

void createSharedObjects(void);
void rdbLoadProgressCallback(rio *r, const void *buf, size_t len);
long long rdbLoadMillisecondTime(rio *rdb);
int rdbCheckMode = 0;

struct {
    rio *rio;
    robj *key;                      /* Current key we are reading. */
    int key_type;                   /* Current key type if != -1. */
    unsigned long keys;             /* Number of keys processed. */
    unsigned long expires;          /* Number of keys with an expire. */
    unsigned long already_expired;  /* Number of keys already expired. */
    int doing;                      /* The state while reading the RDB. */
    int error_set;                  /* True if error is populated. */
    char error[1024];
} rdbstate;

/* At every loading step try to remember what we were about to do, so that
 * we can log this information when an error is encountered. */
#define RDB_CHECK_DOING_START 0
#define RDB_CHECK_DOING_READ_TYPE 1
#define RDB_CHECK_DOING_READ_EXPIRE 2
#define RDB_CHECK_DOING_READ_KEY 3
#define RDB_CHECK_DOING_READ_OBJECT_VALUE 4
#define RDB_CHECK_DOING_CHECK_SUM 5
#define RDB_CHECK_DOING_READ_LEN 6
#define RDB_CHECK_DOING_READ_AUX 7

char *rdb_check_doing_string[] = {
    "start",
    "read-type",
    "read-expire",
    "read-key",
    "read-object-value",
    "check-sum",
    "read-len",
    "read-aux"
};

char *rdb_type_string[] = {
    "string",
    "list-linked",
    "set-hashtable",
    "zset-v1",
    "hash-hashtable",
    "zset-v2",
    "module-value",
    "","",
    "hash-zipmap",
    "list-ziplist",
    "set-intset",
    "zset-ziplist",
    "hash-ziplist",
    "quicklist"
};

/* Show a few stats collected into 'rdbstate' */
void rdbShowGenericInfo(void) {
    printf("[info] %lu keys read\n", rdbstate.keys);
    printf("[info] %lu expires\n", rdbstate.expires);
    printf("[info] %lu already expired\n", rdbstate.already_expired);
}

/* Called on RDB errors. Provides details about the RDB and the offset
 * we were when the error was detected. */
void rdbCheckError(const char *fmt, ...) {
    char msg[1024];
    va_list ap;

    va_start(ap, fmt);
    vsnprintf(msg, sizeof(msg), fmt, ap);
    va_end(ap);

    printf("--- RDB ERROR DETECTED ---\n");
    printf("[offset %llu] %s\n",
        (unsigned long long) (rdbstate.rio ?
            rdbstate.rio->processed_bytes : 0), msg);
    printf("[additional info] While doing: %s\n",
        rdb_check_doing_string[rdbstate.doing]);
    if (rdbstate.key)
        printf("[additional info] Reading key '%s'\n",
            (char*)rdbstate.key->ptr);
    if (rdbstate.key_type != -1)
        printf("[additional info] Reading type %d (%s)\n",
            rdbstate.key_type,
            ((unsigned)rdbstate.key_type <
             sizeof(rdb_type_string)/sizeof(char*)) ?
                rdb_type_string[rdbstate.key_type] : "unknown");
    rdbShowGenericInfo();
}

/* Print informations during RDB checking. */
void rdbCheckInfo(const char *fmt, ...) {
    char msg[1024];
    va_list ap;

    va_start(ap, fmt);
    vsnprintf(msg, sizeof(msg), fmt, ap);
    va_end(ap);

    printf("[offset %llu] %s\n",
        (unsigned long long) (rdbstate.rio ?
            rdbstate.rio->processed_bytes : 0), msg);
}

/* Used inside rdb.c in order to log specific errors happening inside
 * the RDB loading internals. */
void rdbCheckSetError(const char *fmt, ...) {
    va_list ap;

    va_start(ap, fmt);
    vsnprintf(rdbstate.error, sizeof(rdbstate.error), fmt, ap);
    va_end(ap);
    rdbstate.error_set = 1;
}

/* During RDB check we setup a special signal handler for memory violations
 * and similar conditions, so that we can log the offending part of the RDB
 * if the crash is due to broken content. */
void rdbCheckHandleCrash(int sig, siginfo_t *info, void *secret) {
    UNUSED(sig);
    UNUSED(info);
    UNUSED(secret);

    rdbCheckError("Server crash checking the specified RDB file!");
    exit(1);
}

void rdbCheckSetupSignals(void) {
    struct sigaction act;

    sigemptyset(&act.sa_mask);
    act.sa_flags = SA_NODEFER | SA_RESETHAND | SA_SIGINFO;
    act.sa_sigaction = rdbCheckHandleCrash;
    sigaction(SIGSEGV, &act, NULL);
    sigaction(SIGBUS, &act, NULL);
    sigaction(SIGFPE, &act, NULL);
    sigaction(SIGILL, &act, NULL);
}

/* Check the specified RDB file. Return 0 if the RDB looks sane, otherwise
 * 1 is returned. */
int redis_check_rdb(char *rdbfilename) {
    uint64_t dbid;
    int type, rdbver;
    char buf[1024];
    long long expiretime, now = mstime();
    FILE *fp;
    static rio rdb; /* Pointed by global struct riostate. */

    if ((fp = fopen(rdbfilename,"r")) == NULL) return 1;

    rioInitWithFile(&rdb,fp);
    rdbstate.rio = &rdb;
    rdb.update_cksum = rdbLoadProgressCallback;
    if (rioRead(&rdb,buf,9) == 0) goto eoferr;
    buf[9] = '\0';
    if (memcmp(buf,"REDIS",5) != 0) {
        rdbCheckError("Wrong signature trying to load DB from file");
        return 1;
    }
    rdbver = atoi(buf+5);
    if (rdbver < 1 || rdbver > RDB_VERSION) {
        rdbCheckError("Can't handle RDB format version %d",rdbver);
        return 1;
    }

    startLoading(fp);
    while(1) {
        robj *key, *val;
        expiretime = -1;

        /* Read type. */
        rdbstate.doing = RDB_CHECK_DOING_READ_TYPE;
        if ((type = rdbLoadType(&rdb)) == -1) goto eoferr;

        /* Handle special types. */
        if (type == RDB_OPCODE_EXPIRETIME) {
            rdbstate.doing = RDB_CHECK_DOING_READ_EXPIRE;
            /* EXPIRETIME: load an expire associated with the next key
             * to load. Note that after loading an expire we need to
             * load the actual type, and continue. */
            if ((expiretime = rdbLoadTime(&rdb)) == -1) goto eoferr;
            /* We read the time so we need to read the object type again. */
            rdbstate.doing = RDB_CHECK_DOING_READ_TYPE;
            if ((type = rdbLoadType(&rdb)) == -1) goto eoferr;
            /* the EXPIRETIME opcode specifies time in seconds, so convert
             * into milliseconds. */
            expiretime *= 1000;
        } else if (type == RDB_OPCODE_EXPIRETIME_MS) {
            /* EXPIRETIME_MS: milliseconds precision expire times introduced
             * with RDB v3. Like EXPIRETIME but no with more precision. */
            rdbstate.doing = RDB_CHECK_DOING_READ_EXPIRE;
            if ((expiretime = rdbLoadMillisecondTime(&rdb)) == -1) goto eoferr;
            /* We read the time so we need to read the object type again. */
            rdbstate.doing = RDB_CHECK_DOING_READ_TYPE;
            if ((type = rdbLoadType(&rdb)) == -1) goto eoferr;
        } else if (type == RDB_OPCODE_EOF) {
            /* EOF: End of file, exit the main loop. */
            break;
        } else if (type == RDB_OPCODE_SELECTDB) {
            /* SELECTDB: Select the specified database. */
            rdbstate.doing = RDB_CHECK_DOING_READ_LEN;
            if ((dbid = rdbLoadLen(&rdb,NULL)) == RDB_LENERR)
                goto eoferr;
            rdbCheckInfo("Selecting DB ID %d", dbid);
            continue; /* Read type again. */
        } else if (type == RDB_OPCODE_RESIZEDB) {
            /* RESIZEDB: Hint about the size of the keys in the currently
             * selected data base, in order to avoid useless rehashing. */
            uint64_t db_size, expires_size;
            rdbstate.doing = RDB_CHECK_DOING_READ_LEN;
            if ((db_size = rdbLoadLen(&rdb,NULL)) == RDB_LENERR)
                goto eoferr;
            if ((expires_size = rdbLoadLen(&rdb,NULL)) == RDB_LENERR)
                goto eoferr;
            continue; /* Read type again. */
        } else if (type == RDB_OPCODE_AUX) {
            /* AUX: generic string-string fields. Use to add state to RDB
             * which is backward compatible. Implementations of RDB loading
             * are requierd to skip AUX fields they don't understand.
             *
             * An AUX field is composed of two strings: key and value. */
            robj *auxkey, *auxval;
            rdbstate.doing = RDB_CHECK_DOING_READ_AUX;
            if ((auxkey = rdbLoadStringObject(&rdb)) == NULL) goto eoferr;
            if ((auxval = rdbLoadStringObject(&rdb)) == NULL) goto eoferr;

            rdbCheckInfo("AUX FIELD %s = '%s'",
                (char*)auxkey->ptr, (char*)auxval->ptr);
            decrRefCount(auxkey);
            decrRefCount(auxval);
            continue; /* Read type again. */
        } else {
            if (!rdbIsObjectType(type)) {
                rdbCheckError("Invalid object type: %d", type);
                return 1;
            }
            rdbstate.key_type = type;
        }

        /* Read key */
        rdbstate.doing = RDB_CHECK_DOING_READ_KEY;
        if ((key = rdbLoadStringObject(&rdb)) == NULL) goto eoferr;
        rdbstate.key = key;
        rdbstate.keys++;
        /* Read value */
        rdbstate.doing = RDB_CHECK_DOING_READ_OBJECT_VALUE;
        if ((val = rdbLoadObject(type,&rdb)) == NULL) goto eoferr;
        /* Check if the key already expired. This function is used when loading
         * an RDB file from disk, either at startup, or when an RDB was
         * received from the master. In the latter case, the master is
         * responsible for key expiry. If we would expire keys here, the
         * snapshot taken by the master may not be reflected on the slave. */
        if (server.masterhost == NULL && expiretime != -1 && expiretime < now)
            rdbstate.already_expired++;
        if (expiretime != -1) rdbstate.expires++;
        rdbstate.key = NULL;
        decrRefCount(key);
        decrRefCount(val);
        rdbstate.key_type = -1;
    }
    /* Verify the checksum if RDB version is >= 5 */
    if (rdbver >= 5 && server.rdb_checksum) {
        uint64_t cksum, expected = rdb.cksum;

        rdbstate.doing = RDB_CHECK_DOING_CHECK_SUM;
        if (rioRead(&rdb,&cksum,8) == 0) goto eoferr;
        memrev64ifbe(&cksum);
        if (cksum == 0) {
            rdbCheckInfo("RDB file was saved with checksum disabled: no check performed.");
        } else if (cksum != expected) {
            rdbCheckError("RDB CRC error");
        } else {
            rdbCheckInfo("Checksum OK");
        }
    }

    fclose(fp);
    return 0;

eoferr: /* unexpected end of file is handled here with a fatal exit */
    if (rdbstate.error_set) {
        rdbCheckError(rdbstate.error);
    } else {
        rdbCheckError("Unexpected EOF reading RDB file");
    }
    return 1;
}

/* RDB check main: called form redis.c when Redis is executed with the
 * redis-check-rdb alias.
 *
 * The function never returns, but exits with the status code according
 * to success (RDB is sane) or error (RDB is corrupted). */
int redis_check_rdb_main(int argc, char **argv) {
    if (argc != 2) {
        fprintf(stderr, "Usage: %s <rdb-file-name>\n", argv[0]);
        exit(1);
    }
    /* In order to call the loading functions we need to create the shared
     * integer objects, however since this function may be called from
     * an already initialized Redis instance, check if we really need to. */
    if (shared.integers[0] == NULL)
        createSharedObjects();
    server.loading_process_events_interval_bytes = 0;
    rdbCheckMode = 1;
    rdbCheckInfo("Checking RDB file %s", argv[1]);
    rdbCheckSetupSignals();
    int retval = redis_check_rdb(argv[1]);
    if (retval == 0) {
        rdbCheckInfo("\\o/ RDB looks OK! \\o/");
        rdbShowGenericInfo();
    }
    exit(retval);
}

```


## testhelp.c

```C
/* This is a really minimal testing framework for C.
 *
 * Example:
 *
 * test_cond("Check if 1 == 1", 1==1)
 * test_cond("Check if 5 > 10", 5 > 10)
 * test_report()
 */

#ifndef __TESTHELP_H
#define __TESTHELP_H

int __failed_tests = 0;
int __test_num = 0;
#define test_cond(descr,_c) do { \
    __test_num++; printf("%d - %s: ", __test_num, descr); \
    if(_c) printf("PASSED\n"); else {printf("FAILED\n"); __failed_tests++;} \
} while(0);
#define test_report() do { \
    printf("%d tests, %d passed, %d failed\n", __test_num, \
                    __test_num-__failed_tests, __failed_tests); \
    if (__failed_tests) { \
        printf("=== WARNING === We have failed tests here...\n"); \
        exit(1); \
    } \
} while(0);

#endif

```
