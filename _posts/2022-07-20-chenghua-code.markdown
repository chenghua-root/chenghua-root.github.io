---  
layout: post  
title:  "个人工作代码"  
date:   2022-07-20 00:00:00 +0530   
---  
  
<style>  
.tablelines table, .tablelines td, .tablelines th {  
  border: 1px solid black;  
  }  
</style>  
  
## 介绍  
本文列出了以往工作项目的部分代码截图。分别包含:  
- PolarStore压缩存储相关代码  
- 美团对象存储LRC测试代码  
- 美团图片服务多进程改造和水印相关代码  
- 美团对象存储离线任务buffer管理相关代码  
- **美团对象存储离线任务buffer管理单测**  
  
## PolarStore压缩存储  
PolarStore压缩存储元数据管理代码截图  
![压缩存储元数据01](https://chenghua-root.github.io/images/code_compress_meta_07.png)  
![压缩存储元数据02](https://chenghua-root.github.io/images/code_compress_meta_08.png)  

## 跨机房纠删码LRC测试  
通过递归的方式遍历LRC各种损坏数据的情况  
![LRC递归](https://chenghua-root.github.io/images/code_lrc_test_01.png)  
![LRC3+3+6](https://chenghua-root.github.io/images/code_lrc_test_02.png)  

## 图片服务  
图片服务多进程改造  
![图片服务多进程改造](https://chenghua-root.github.io/images/code_image_multi_process_01.png)  
图片服务水印功能代码截图  
![图片服务水印](https://chenghua-root.github.io/images/code_image_watermark_01.png)  

## 离线任务buffer  
离线任务流水线处理buffer管理代码截图  
![store buffer header](https://chenghua-root.github.io/images/code_store_buffer_header_01.png)  
![store buffer 01](https://chenghua-root.github.io/images/code_store_buffer_01.png)  
![store buffer 02](https://chenghua-root.github.io/images/code_store_buffer_02.png)  

## 离线任务buffer单测  
![store buffer unittest](https://chenghua-root.github.io/images/code_store_buffer_unittest_01.png)  

```  
#include "ctest.h"

#include "storeserver/s3_storeserver.h"
#include "storeserver/storebuffer/s3_store_buffer_mgr.h"
#include "storeserver/storebuffer/s3_store_buffer_callback.h"

#include "unittest/storeserver/global.h"

int test_rpc_ctx_queue_init_v2();


void test_rpc_ctx_queue_destroy();
extern void s3_store_buffer_cb_arg_destruct(S3StoreBufferCBArg *arg);

TEST(test_store_buffer, store_buffer_mgr_init) {
  S3StoreBufferMgr *mgr = s3_store_buffer_mgr_construct();
  ASSERT_EQ(S3_OK, s3_store_buffer_mgr_init(mgr));

  ASSERT_EQ(1, mgr->inited_);
  ASSERT_NE(NULL, mgr->rb_tree_.root);

  s3_store_buffer_mgr_destroy(mgr);
  ASSERT_EQ(0, memcmp(mgr, &(S3StoreBufferMgr){{0}}, sizeof(S3StoreBufferMgr)));

  s3_store_buffer_mgr_destruct(mgr);
}

TEST(test_store_buffer, store_buffer_unit_init) {
  int64_t size = 1024;

  S3StoreBufferUnit *unit = s3_store_buffer_unit_construct();
  ASSERT_EQ(S3_OK, s3_store_buffer_unit_init(unit, size));

  ASSERT_NE(NULL, unit->data);
  ASSERT_EQ(size, unit->size);
  ASSERT_EQ(0, unit->used_size);
  ASSERT_EQ(0, unit->offset);
  ASSERT_EQ(1, unit->inited_);

  s3_store_buffer_unit_destroy(unit);
  ASSERT_EQ(0, memcmp(unit, &(S3StoreBufferUnit){{0}}, sizeof(S3StoreBufferUnit)));

  s3_store_buffer_unit_destruct(unit);
}

TEST(test_store_buffer, store_buffer_chunk_init) {
  S3ReplicaID chunk_id = {.ip = 1234, .type = S3_PARTITION_TYPE_CP3P, .ts = 4567, .inited_ = 1};
  int64_t try_limit = 10;

  S3StoreBufferChunk *chunk = s3_store_buffer_chunk_construct();
  ASSERT_EQ(S3_OK, s3_store_buffer_chunk_init(chunk, &chunk_id, try_limit));

  ASSERT_EQ(S3_STORE_BUFFER_CHUNK_STAGE_INIT, chunk->stage);
  ASSERT_EQ(0, s3_replica_id_cmp(&chunk_id, &chunk->chunk_id));
  ASSERT_EQ(try_limit, chunk->try_limit);
  ASSERT_EQ(S3_OK, chunk->rpc_ret);
  ASSERT_EQ(0, chunk->try_cnt);
  ASSERT_EQ(1, chunk->inited_);

  s3_store_buffer_chunk_destroy(chunk);
  ASSERT_EQ(0, chunk->inited_);

  s3_store_buffer_chunk_destruct(chunk);
}

#define TEST_STORE_BUFFER_CNT_COMMON 5
#define TEST_STORE_BUFFER_SIZE_COMMON 1024
#define TEST_STORE_BUFFER_CHUNK_LEN_COMMON (2 * TEST_STORE_BUFFER_CNT_COMMON * TEST_STORE_BUFFER_SIZE_COMMON)
#define TEST_STORE_BUFFER_TRY_LIMIT 2

void test_store_buffer_cb_func(void *arg, int rst) {
  if (S3_OK == rst) {
    if ((*(int *)arg) < 0 && S3_ERR_AGAIN != (*(int *)arg)) {
      s3_bug("unit repeat callback, CB_ARG=%d", (*(int*)arg));
    }
    s3_atomic_inc((int *)arg);
  } else {
    s3_atomic_store((int *)arg, rst);
  }
}

int CB_ARG = 0;
static int CHUNKS_CNT = 2;
static S3ReplicaID CHUNK_ID = {.ip = 1234, .type = S3_PARTITION_TYPE_CP3P, .ts = 4567, .inited_ = 1};
static S3ReplicaID CHUNK_ID02 = {.ip = 2232, .type = S3_PARTITION_TYPE_CP3P, .ts = 4567, .inited_ = 1};
static S3ReplicaID CHUNK_ID03 = {.ip = 2233, .type = S3_PARTITION_TYPE_CP3P, .ts = 4567, .inited_ = 1};
static S3ReplicaID CHUNK_ID04 = {.ip = 2234, .type = S3_PARTITION_TYPE_CP3P, .ts = 4567, .inited_ = 1};
static S3ReplicaID CHUNK_ID05 = {.ip = 2235, .type = S3_PARTITION_TYPE_CP3P, .ts = 4567, .inited_ = 1};

void test_store_buffer_init(S3StoreBuffer *b, int64_t buffer_cnt, int64_t chunk_len) {
  int64_t size = TEST_STORE_BUFFER_SIZE_COMMON;
  uint64_t trace_id = s3_get_inc_timestamp();
  S3Array chunk_ids = s3_array_null;
  S3PartitionID gid = s3_partition_id_null;

  ASSERT_EQ(S3_OK, s3_replica_init_partition_id(&gid, S3_PARTITION_TYPE_ECGP));

  ASSERT_EQ(S3_OK, s3_array_init(&chunk_ids, sizeof(S3ReplicaID), (S3FuncArrayEltDestructor)s3_replica_id_destroy));
  ASSERT_EQ(S3_OK, s3_array_append_val(&chunk_ids, &CHUNK_ID));
  if (CHUNKS_CNT >= 2) {
    ASSERT_EQ(S3_OK, s3_array_append_val(&chunk_ids, &CHUNK_ID02));
  }
  if (CHUNKS_CNT >= 3) {
    ASSERT_EQ(S3_OK, s3_array_append_val(&chunk_ids, &CHUNK_ID03));
  }
  if (CHUNKS_CNT >= 4) {
    ASSERT_EQ(S3_OK, s3_array_append_val(&chunk_ids, &CHUNK_ID04));
  }
  if (CHUNKS_CNT >= 5) {
    ASSERT_EQ(S3_OK, s3_array_append_val(&chunk_ids, &CHUNK_ID05));
  }

  ASSERT_EQ(S3_OK, s3_store_buffer_init(b, size, buffer_cnt, &chunk_ids, TEST_STORE_BUFFER_TRY_LIMIT, trace_id, &gid));

  s3_partition_id_destroy(&gid);
  s3_array_destroy(&chunk_ids);
}

int test_store_buffer_check_status(S3StoreBuffer *b) {
  s3_must_inited(b, S3_ERR_UNINITED);

  int ret = S3_OK;
  S3StoreBufferChunk *chunk = NULL;

  s3_spin_lock_safe(&b->lock_);

  s3_array_for_each_element(&b->chunks, chunk) {
    ret = chunk->rpc_ret;

    if (S3_ERR_TIMEOUT == ret) {
      if (S3_STORE_BUFFER_CHUNK_STAGE_INIT == chunk->stage && chunk->try_cnt <= chunk->try_limit) {
        ret = S3_OK;
      } else if (S3_STORE_BUFFER_CHUNK_STAGE_FINISH == chunk->stage && chunk->try_cnt < chunk->try_limit) {
        ret = S3_OK;
      }
    }
    if s3_unlikely(S3_OK != ret) {
      break;
    }
  }

  s3_spin_unlock_safe(&b->lock_);

  return ret;
}

TEST(test_store_buffer, store_buffer_init) {
  int64_t try_limit = 10;
  int64_t size = 1024;
  int64_t buffer_cnt = 5;
  uint64_t trace_id = s3_get_inc_timestamp();
  S3Array chunk_ids = s3_array_null;
  S3PartitionID gid = s3_partition_id_null;

  ASSERT_EQ(S3_OK, s3_replica_init_partition_id(&gid, S3_PARTITION_TYPE_ECGP));

  ASSERT_EQ(S3_OK, s3_array_init(&chunk_ids, sizeof(S3ReplicaID), (S3FuncArrayEltDestructor)s3_replica_id_destroy));
  S3ReplicaID chunk_id = {.ip = 1234, .type = S3_PARTITION_TYPE_CP3P, .ts = 4567, .inited_ = 1};
  S3ReplicaID chunk_id02 = {.ip = 2234, .type = S3_PARTITION_TYPE_CP3P, .ts = 4567, .inited_ = 1};
  ASSERT_EQ(S3_OK, s3_array_append_val(&chunk_ids, &chunk_id));
  ASSERT_EQ(S3_OK, s3_array_append_val(&chunk_ids, &chunk_id02));

  S3StoreBuffer *b = s3_store_buffer_construct();
  ASSERT_EQ(S3_OK, s3_store_buffer_init(b, size, buffer_cnt, &chunk_ids, try_limit, trace_id, &gid));

  ASSERT_NE(0, b->id);
  ASSERT_EQ(trace_id, b->trace_id);
  ASSERT_EQ(0, b->ref_cnt_);
  ASSERT_EQ(1, b->inited_);

  int unit_cnt = 0;
  S3StoreBufferUnit *unit = NULL, *unitn = NULL;
  s3_list_for_each_entry_safe(unit, unitn, &b->free_list, node_) {
    ASSERT_NE(NULL, unit->data);
    ASSERT_EQ(size, unit->size);
    ASSERT_EQ(0, unit->used_size);
    ASSERT_EQ(0, unit->offset);
    ASSERT_EQ(1, unit->inited_);
    unit_cnt++;
  }
  ASSERT_EQ(unit_cnt, buffer_cnt);

  int chunk_idx = 0;
  S3StoreBufferChunk *chunk = NULL;
  s3_array_for_each_element(&b->chunks, chunk) {
    ASSERT_EQ(S3_STORE_BUFFER_CHUNK_STAGE_INIT, chunk->stage);
    ASSERT_EQ(try_limit, chunk->try_limit);
    ASSERT_EQ(S3_OK, chunk->rpc_ret);
    ASSERT_EQ(0, chunk->try_cnt);
    ASSERT_EQ(1, chunk->inited_);

    S3ReplicaID *chunk_id = s3_array_get_element(&chunk_ids, chunk_idx);
    ASSERT_EQ(0, s3_replica_id_cmp(chunk_id, &chunk->chunk_id));
    chunk_idx++;
  }

  s3_store_buffer_destroy(b);
  ASSERT_EQ(0, b->inited_);

  s3_partition_id_destroy(&gid);
  s3_store_buffer_destruct(b);
  s3_array_destroy(&chunk_ids);
}

/*
 * 对store buffer的添加、删除、获取进行测试
 */
TEST(test_store_buffer, store_buffer_mgr_add_del_get_buffer) {
  S3StoreBufferMgr *mgr = s3_store_buffer_mgr_construct();
  ASSERT_EQ(S3_OK, s3_store_buffer_mgr_init(mgr));

  int64_t id = 0;
  S3StoreBuffer *ret_buffer = NULL;
  S3StoreBuffer *b = s3_store_buffer_construct();

  test_store_buffer_init(b, TEST_STORE_BUFFER_CNT_COMMON, TEST_STORE_BUFFER_CHUNK_LEN_COMMON);
  id = b->id;

  // add
  ASSERT_EQ(S3_OK, s3_store_buffer_mgr_add(mgr, b));
  ASSERT_EQ(S3_ERR_EXIST, s3_store_buffer_mgr_add(mgr, b));

  // get
  ASSERT_EQ(S3_OK, s3_store_buffer_mgr_get(mgr, id, &ret_buffer));
  ASSERT_EQ(b, ret_buffer);
  s3_store_buffer_revert(ret_buffer);

  // get a buffer with not exist id
  ASSERT_EQ(S3_ERR_NOT_FOUND, s3_store_buffer_mgr_get(mgr, id+1, &ret_buffer));

  // del
  ASSERT_EQ(S3_OK, s3_store_buffer_mgr_del(mgr, b->id));
  ASSERT_EQ(S3_ERR_NOT_FOUND, s3_store_buffer_mgr_get(mgr, id, &ret_buffer));

  s3_store_buffer_mgr_destruct(mgr);
}

TEST(test_store_buffer, store_buffer_revert) {
  S3StoreBuffer *b = s3_store_buffer_construct();
  test_store_buffer_init(b, TEST_STORE_BUFFER_CNT_COMMON, TEST_STORE_BUFFER_CHUNK_LEN_COMMON);

  ASSERT_EQ(0, b->ref_cnt_);

  s3_atomic_inc(&b->ref_cnt_);
  s3_atomic_inc(&b->ref_cnt_);

  s3_store_buffer_revert(b);
  ASSERT_EQ(1, b->ref_cnt_);
  s3_store_buffer_revert(b);
}

/***************************************************************************/

TEST(test_store_buffer, store_buffer_create_close) {
  S3StoreBufferMgr *mgr = s3_store_buffer_mgr_construct();
  ASSERT_EQ(S3_OK, s3_store_buffer_mgr_init(mgr));

  int64_t id = 0;
  int64_t try_limit = 10;
  int64_t size = 1024;
  int64_t buffer_cnt = 5;
  uint64_t trace_id = s3_get_inc_timestamp();
  S3Array chunk_ids = s3_array_null;
  S3StoreBuffer *b = NULL;
  S3StoreBuffer *b02 = NULL;
  S3PartitionID gid = s3_partition_id_null;

  ASSERT_EQ(S3_OK, s3_replica_init_partition_id(&gid, S3_PARTITION_TYPE_ECGP));

  ASSERT_EQ(S3_OK, s3_array_init(&chunk_ids, sizeof(S3ReplicaID), (S3FuncArrayEltDestructor)s3_replica_id_destroy));
  S3ReplicaID chunk_id = {.ip = 1234, .type = S3_PARTITION_TYPE_CP3P, .ts = 4567, .inited_ = 1};
  S3ReplicaID chunk_id02 = {.ip = 2234, .type = S3_PARTITION_TYPE_CP3P, .ts = 4567, .inited_ = 1};
  ASSERT_EQ(S3_OK, s3_array_append_val(&chunk_ids, &chunk_id));
  ASSERT_EQ(S3_OK, s3_array_append_val(&chunk_ids, &chunk_id02));

  ASSERT_EQ(S3_OK, s3_store_buffer_create(mgr, size, buffer_cnt, &chunk_ids, try_limit, trace_id, &gid, &b));
  id = b->id;

  ASSERT_EQ(S3_OK, s3_store_buffer_mgr_get(mgr, id, &b02));
  ASSERT_EQ(b, b02);
  s3_store_buffer_revert(b02);

  ASSERT_EQ(S3_OK, s3_store_buffer_close(mgr, b));

  ASSERT_EQ(S3_ERR_NOT_FOUND, s3_store_buffer_mgr_get(mgr, id, &b02));

  s3_partition_id_destroy(&gid);
  s3_array_destroy(&chunk_ids);
  s3_store_buffer_mgr_destruct(mgr);
}

#define TEST_SWAP_ASSERT_AND_RESET \
  ASSERT_NE(NULL, ret_buf); \
  data = ret_buf; \
  ret_buf = NULL; \
  offset += used_size; \

TEST(test_store_buffer, store_buffer_swap) {
  S3StoreBuffer *b = s3_store_buffer_construct();
  test_store_buffer_init(b, TEST_STORE_BUFFER_CNT_COMMON, TEST_STORE_BUFFER_CHUNK_LEN_COMMON);
  ASSERT_EQ(S3_OK, test_rpc_ctx_queue_init_v2());

  int size = TEST_STORE_BUFFER_SIZE_COMMON;
  int64_t used_size = size;
  S3ReplicaHeader rch = s3_replica_header_null;
  int offset = s3_replica_header_size(&rch);
  char *data = s3_malloc(size);
  char *ret_buf = NULL;

  {
    S3StoreBufferSwapReq req = {
      .data = data,
      .size = size,
      .offset = offset,
      .used_size = used_size ,
      .cb = test_store_buffer_cb_func,
      .cb_arg = &CB_ARG,
    };
    ASSERT_EQ(S3_OK, s3_store_buffer_swap(b, &req, &ret_buf));
    TEST_SWAP_ASSERT_AND_RESET; // will increment offset
  }

  {
    S3StoreBufferSwapReq req = {
      .data = data,
      .size = size - 1,
      .offset = offset,
      .used_size = size - 1,
      .cb = test_store_buffer_cb_func,
      .cb_arg = &CB_ARG,
    };
    ASSERT_EQ(S3_ERR_INVALID_ARG, s3_store_buffer_swap(b, &req, &ret_buf));
  }

  {
    S3StoreBufferSwapReq req = {
      .data = data,
      .size = size,
      .offset = offset,
      .used_size = size + 1,
      .cb = test_store_buffer_cb_func,
      .cb_arg = &CB_ARG,
    };
    ASSERT_EQ(S3_ERR_INVALID_ARG, s3_store_buffer_swap(b, &req, &ret_buf));
    CB_ARG = S3_OK;
  }

  for (int i = 1; i < TEST_STORE_BUFFER_CNT_COMMON; i++) {
    S3StoreBufferSwapReq req = {
      .data = data,
      .size = size,
      .offset = offset,
      .used_size = used_size ,
      .cb = test_store_buffer_cb_func,
      .cb_arg = &CB_ARG,
    };
    ASSERT_EQ(S3_OK, s3_store_buffer_swap(b, &req, &ret_buf));
    TEST_SWAP_ASSERT_AND_RESET; // will increment offset
  }

  // free list is empty, would return S3_ERR_AGAIN
  {
    S3StoreBufferSwapReq req = {
      .data = data,
      .size = size,
      .offset = offset,
      .used_size = used_size ,
      .cb = test_store_buffer_cb_func,
      .cb_arg = &CB_ARG,
    };
    ASSERT_EQ(S3_ERR_AGAIN, s3_store_buffer_swap(b, &req, &ret_buf));
  }

  s3_store_buffer_destruct(b);
  s3_free(data);

  test_rpc_ctx_queue_destroy();
}

TEST(test_store_buffer, store_buffer_flushed_all) {
  S3StoreBuffer *b = s3_store_buffer_construct();
  test_store_buffer_init(b, TEST_STORE_BUFFER_CNT_COMMON, TEST_STORE_BUFFER_CHUNK_LEN_COMMON);
  ASSERT_EQ(S3_OK, test_rpc_ctx_queue_init_v2());

  int64_t size = TEST_STORE_BUFFER_SIZE_COMMON;
  int64_t used_size = 0;
  int64_t offset = 128;
  char *data = s3_malloc(size);
  char *ret_buf = NULL;

  used_size = size;

  S3StoreBufferSwapReq req = {
    .data = data,
    .size = size,
    .offset = offset,
    .used_size = used_size,
    .cb = test_store_buffer_cb_func,
    .cb_arg = &CB_ARG,
  };
  ASSERT_EQ(S3_OK, s3_store_buffer_swap(b, &req, &ret_buf));

  ASSERT_EQ(S3_ERR_AGAIN, s3_store_buffer_flushed(b));

  s3_store_buffer_destruct(b);
  s3_free(ret_buf);

  test_rpc_ctx_queue_destroy();
}

/********************************rpc test*******************************************/

extern int s3_store_buffer_try_write_chunks(S3StoreBuffer *b);
extern void s3_store_buffer_rpc_do_callback(void *arg);

S3Queue test_rpc_ctx_queue = s3_queue_null;

char *src_chunk = NULL;
char *dst_chunk = NULL;
static void test_prepare_chunk_data(char** chunk, int chunk_len) {
  char *chunk_ = s3_malloc(chunk_len);
  ASSERT_NE(NULL, chunk_);
  *chunk = chunk_;
  for (int i = 0; i < chunk_len; i++) {
    *(chunk_ + i) = test_random_char();
  }
}

static void test_set_chunk_data_zero(char* chunk, int chunk_len) {
  memset(chunk, 0, chunk_len);
}

void test_destruct_chunk_data(char* chunk) {
  s3_free(chunk);
}

static void test_create_chunk(char** chunk, int chunk_len) {
  char * chunk_ = s3_malloc(chunk_len);
  ASSERT_NE(NULL, chunk_);
  *chunk = chunk_;
}

void test_destruct_chunk(char* chunk) {
  s3_free(chunk);
}


typedef struct TestRpcCtx TestRpcCtx;
struct TestRpcCtx {
  S3ListHead node_;
  S3Server server;
  S3ReplicaID chunk_id;
  S3String data;
  int64_t offset;
  S3StoreBufferCBArg *arg;
};

#define test_rpc_ctx_null { \
}

TestRpcCtx * test_rpc_ctx_construct() {
  TestRpcCtx * ctx = s3_malloc(sizeof(TestRpcCtx));
  if s3_likely(NULL != ctx) {
    *ctx = (TestRpcCtx)test_rpc_ctx_null;
  }
  return ctx;
}

int test_rpc_ctx_init() {
  int ret = S3_OK;
  return ret;
}

void test_rpc_ctx_destroy(TestRpcCtx *ctx) {
  if (NULL != ctx) {
    s3_string_destroy(&ctx->data);
  }
}

void test_rpc_ctx_destroy_v2(TestRpcCtx *ctx) {
  if (NULL != ctx) {
    s3_string_destroy(&ctx->data);
    s3_store_buffer_cb_arg_destruct(ctx->arg);
  }
}

void test_rpc_ctx_destruct(TestRpcCtx *ctx) {
  if (NULL != ctx) {

  S3_DEBUG("TEST begin");
    test_rpc_ctx_destroy(ctx);
  S3_DEBUG("TEST end");
    s3_free(ctx);
  S3_DEBUG("TEST endend");
  }
}

void test_rpc_ctx_destruct_v2(TestRpcCtx *ctx) {
  if (NULL != ctx) {
    test_rpc_ctx_destroy_v2(ctx);
    s3_free(ctx);
  }
}

int test_rpc_ctx_queue_init () {
    return s3_queue_init(&test_rpc_ctx_queue, (S3QueueDestroyHandler)test_rpc_ctx_destruct);
}

int test_rpc_ctx_queue_init_v2 () {
    return s3_queue_init(&test_rpc_ctx_queue, (S3QueueDestroyHandler)test_rpc_ctx_destruct_v2);
}

void test_rpc_ctx_queue_destroy() {
    s3_queue_destroy(&test_rpc_ctx_queue);
}

int test_create_rpc_ctx(const S3Server *server, const S3ReplicaID *rid, int64_t offset, const S3String *data, void *args, TestRpcCtx **ret_ctx) {
  int ret = S3_OK;

  TestRpcCtx * ctx = test_rpc_ctx_construct();
  s3_check_ret(ret = (ctx == NULL ? S3_ERR_OUT_OF_MEM : S3_OK), "");

  ret = s3_server_dupfrom(&ctx->server, server);
  s3_check_ret(ret, "");

  ret = s3_replica_id_dupfrom(&ctx->chunk_id, rid);
  s3_check_ret(ret, "");

  ret = s3_string_dupfrom(&ctx->data, data);
  s3_check_ret(ret, "");

  ctx->offset = offset;
  ctx->arg = (S3StoreBufferCBArg*)args;

  *ret_ctx = ctx;

exit:
  return ret;
}

int test_rpc_ctx_append(TestRpcCtx *ctx) {
  S3_DEBUG("[TEST]append ctx, offset:%ld", ctx->offset);
  return s3_queue_append(&test_rpc_ctx_queue, &ctx->node_);
}

TestRpcCtx *test_rpc_ctx_get() {
  S3ListHead *node = s3_queue_remove_head(&test_rpc_ctx_queue);
  if (NULL != node) {
    TestRpcCtx *ctx = (TestRpcCtx*)s3_list_entry(node, TestRpcCtx, node_);
    S3_DEBUG("[TEST]get ctx, offset:%ld", ctx->offset);
    return ctx;
  } else {
    return NULL;
  }
}

int64_t test_rpc_ctx_length() {
  return s3_queue_get_length(&test_rpc_ctx_queue);
}

int test_store_buffer_rpc_request_write_chunk(const S3Server *server, const S3ReplicaID *rid,
  int64_t offset, const S3String *data, void *args) {
  int ret = S3_OK;
  TestRpcCtx *ctx;

  ret = test_create_rpc_ctx(server, rid, offset, data, args, &ctx);
  s3_check_ret(ret, "");

  ret = test_rpc_ctx_append(ctx);
  s3_check_ret(ret, "");

exit:
  return ret;
}

int test_store_buffer_rpc_check_and_do_callback(TestRpcCtx *ctx,
    const S3ReplicaID *rid, char *data, int64_t size, int64_t offset, int rst) {
  int ret = S3_OK;

  if (S3_OK == rst) {
    S3Server srv = s3_server_null;
    S3String data_ = s3_string_wrap(data, size);

    ASSERT_EQ(S3_OK, s3_server_init(&srv, s3_ip4_to_cstring(rid->ip), s3_g.conf.port, IPV4));
    ASSERT_EQ(0, s3_server_compare_by_ip(&srv, &ctx->server));

    ASSERT_EQ(0, s3_replica_id_cmp(rid, &ctx->chunk_id));

    ASSERT_EQ(0, s3_string_cmp(&ctx->data, &data_));

    ASSERT_EQ(offset, ctx->offset);

    memcpy(dst_chunk + ctx->offset, ctx->data.data, size);
  }

  ctx->arg->result = rst;
  s3_store_buffer_rpc_do_callback(ctx->arg);

  test_rpc_ctx_destruct(ctx);

  return ret;
}

S3StoreBuffer *test_store_buffer_write_chunks_init(int64_t buffer_cnt, int64_t chunk_len) {
  S3StoreBufferMgr *mgr = s3_store_buffer_mgr_construct();
  ASSERT_EQ(S3_OK, s3_store_buffer_mgr_init(mgr));
  s3_g.store_buffer_mgr = mgr;

  S3StoreBuffer *b = s3_store_buffer_construct();
  test_store_buffer_init(b, buffer_cnt, chunk_len);
  ASSERT_EQ(S3_OK, s3_store_buffer_mgr_add(mgr, b));

  test_prepare_chunk_data(&src_chunk, chunk_len);
  test_create_chunk(&dst_chunk, chunk_len);

  ASSERT_EQ(S3_OK, test_rpc_ctx_queue_init());

  return b;
}

void test_store_buffer_write_chunks_destroy(S3StoreBufferMgr *mgr, S3StoreBuffer *b) {
  test_rpc_ctx_queue_destroy();

  test_destruct_chunk_data(src_chunk);
  test_destruct_chunk(dst_chunk);

  ASSERT_EQ(S3_OK, s3_store_buffer_mgr_del(mgr, b->id));
  s3_store_buffer_mgr_destruct(mgr);
}

TEST(test_store_buffer, store_buffer_write_chunks) {
  ASSERT_EQ(2, CHUNKS_CNT);

  int size =  TEST_STORE_BUFFER_SIZE_COMMON;
  char *data = s3_malloc(size); char *data_cmp = s3_malloc(size);
  ASSERT_NE(NULL, data); ASSERT_NE(NULL, data_cmp);
  char *ret_buf = NULL;
  int start_offset = 0, offset = 0;
  CB_ARG = 0;
  int swap_cnt = 0;

  /*
   *
   * 每次swap后都执行回调（写unit完成）
   *
   */
  int chunk_len = 20 * TEST_STORE_BUFFER_SIZE_COMMON + TEST_STORE_BUFFER_SIZE_COMMON / 2;
  S3StoreBuffer *b = test_store_buffer_write_chunks_init(TEST_STORE_BUFFER_CNT_COMMON, chunk_len);
  int chunk_cnt = s3_array_get_element_cnt(&b->chunks);
  for (start_offset = 128, offset = start_offset ; offset < chunk_len; swap_cnt++) {
    int used_size = rand() % (TEST_STORE_BUFFER_SIZE_COMMON) + 1;
    memcpy(data, src_chunk+offset, used_size);
    memcpy(data_cmp, src_chunk+offset, used_size);

    used_size = chunk_len < offset + used_size ? (chunk_len - offset) : used_size;
    S3StoreBufferSwapReq req = {
      .data = data,
      .size = size,
      .offset = offset,
      .used_size = used_size,
      .cb = test_store_buffer_cb_func,
      .cb_arg = &CB_ARG,
    };
    ASSERT_EQ(S3_OK, s3_store_buffer_swap(b, &req, &ret_buf));
    TEST_SWAP_ASSERT_AND_RESET; // will increment offset
    for (int j = 0; j < chunk_cnt; ++j) {
      TestRpcCtx *ctx = test_rpc_ctx_get();
      ASSERT_NE(NULL, ctx);
      switch (j) {
        case 0:
          test_store_buffer_rpc_check_and_do_callback(ctx, &CHUNK_ID, data_cmp, used_size, offset-used_size, S3_OK);
          break;
        case 1:
          test_store_buffer_rpc_check_and_do_callback(ctx, &CHUNK_ID02, data_cmp, used_size, offset-used_size, S3_OK);
          break;
        default:
          ASSERT_NE(0, 0);
      }
    }
  }
  ASSERT_EQ(CB_ARG, swap_cnt);
  ASSERT_EQ(0, s3_memcmp(src_chunk + start_offset, dst_chunk + start_offset, chunk_len - start_offset));
  test_store_buffer_write_chunks_destroy(s3_g.store_buffer_mgr, b);

  /*
   *
   * 每次都swap直到没有了free node, 然后do callback直到所有的unit被rpc write完
   *
   */
  int ret = S3_OK;
  int64_t rpc_offset = 0;
  chunk_len = 20 * TEST_STORE_BUFFER_SIZE_COMMON + TEST_STORE_BUFFER_SIZE_COMMON / 2;
  b = test_store_buffer_write_chunks_init(TEST_STORE_BUFFER_CNT_COMMON, chunk_len);
  chunk_cnt = s3_array_get_element_cnt(&b->chunks);
  CB_ARG = 0;
  swap_cnt = 0;
  start_offset = TEST_STORE_BUFFER_SIZE_COMMON + TEST_STORE_BUFFER_SIZE_COMMON / 3; // 从offset非零开始
  for (offset = start_offset, rpc_offset = start_offset; offset < chunk_len; ) {
    int used_size = rand() % (TEST_STORE_BUFFER_SIZE_COMMON) + 1;
    memcpy(data, src_chunk+offset, used_size);

    S3StoreBufferSwapReq req = {
      .data = data,
      .size = size,
      .offset = offset,
      .used_size = used_size,
      .cb = test_store_buffer_cb_func,
      .cb_arg = &CB_ARG,
    };
    ret = s3_store_buffer_swap(b, &req, &ret_buf);
    if (S3_OK == ret) {
      if (offset < chunk_len) {
        swap_cnt++;
      }
      TEST_SWAP_ASSERT_AND_RESET; // will increment offset
      continue;
    } else {
      ASSERT_EQ(S3_ERR_AGAIN, ret);

      while(test_rpc_ctx_length()) {
        for (int j = 0; j < chunk_cnt; ++j) {
          TestRpcCtx *ctx = test_rpc_ctx_get();
          ASSERT_NE(NULL, ctx);
          int arg_size = ctx->arg->size;
          memcpy(data_cmp, src_chunk + rpc_offset, ctx->arg->size);
          switch (j) {
            case 0:
              test_store_buffer_rpc_check_and_do_callback(ctx, &CHUNK_ID, data_cmp, ctx->arg->size, rpc_offset, S3_OK);
              break;
            case 1:
              test_store_buffer_rpc_check_and_do_callback(ctx, &CHUNK_ID02, data_cmp, ctx->arg->size, rpc_offset, S3_OK);
              rpc_offset += arg_size;
              break;
            default:
              ASSERT_NE(0, 0);
          }
        }
      }
    }
  }

  // swap已经完成，处理还未完成rpc的unit
  while(test_rpc_ctx_length()) {
    for (int j = 0; j < chunk_cnt; ++j) {
      TestRpcCtx *ctx = test_rpc_ctx_get();
      ASSERT_NE(NULL, ctx);
        int arg_size = ctx->arg->size;
      memcpy(data_cmp, src_chunk + rpc_offset, ctx->arg->size);
      switch (j) {
        case 0:
          test_store_buffer_rpc_check_and_do_callback(ctx, &CHUNK_ID, data_cmp, ctx->arg->size, rpc_offset, S3_OK);
          break;
        case 1:
          test_store_buffer_rpc_check_and_do_callback(ctx, &CHUNK_ID02, data_cmp, ctx->arg->size, rpc_offset, S3_OK);
          rpc_offset += arg_size;
          break;
        default:
          ASSERT_NE(0, 0);
      }
    }
  }
  ASSERT_EQ(S3_OK, s3_store_buffer_flushed(b));
  ASSERT_EQ(CB_ARG, swap_cnt);
  ASSERT_EQ(0, s3_memcmp(src_chunk + start_offset, dst_chunk + start_offset, chunk_len - start_offset));
  test_store_buffer_write_chunks_destroy(s3_g.store_buffer_mgr, b);

  s3_free(data);
  s3_free(data_cmp);
}

TEST(test_store_buffer, store_buffer_write_chunks_with_error) {
  CHUNKS_CNT = 1;

  int ret = S3_OK;
  int size =  TEST_STORE_BUFFER_SIZE_COMMON;
  char *data = s3_malloc(size); char *data_cmp = s3_malloc(size);
  ASSERT_NE(NULL, data); ASSERT_NE(NULL, data_cmp);
  char *ret_buf = NULL;
  int offset = 128, rpc_offset = 128;
  CB_ARG = 0;

  /*
   * 超时测试
   * 需设置重试次数为2
   */
  ASSERT_EQ(2, TEST_STORE_BUFFER_TRY_LIMIT);

  int chunk_len = 20 * TEST_STORE_BUFFER_SIZE_COMMON + TEST_STORE_BUFFER_SIZE_COMMON / 2;
  int used_size = TEST_STORE_BUFFER_SIZE_COMMON;

  S3StoreBuffer *b = test_store_buffer_write_chunks_init(TEST_STORE_BUFFER_CNT_COMMON, chunk_len);

  // 第一次超时，第二次重试成功
  {
    memcpy(data, src_chunk+offset, used_size);
    S3StoreBufferSwapReq req = {
      .data = data,
      .size = size,
      .offset = offset,
      .used_size = used_size,
      .cb = test_store_buffer_cb_func,
      .cb_arg = &CB_ARG,
    };
    ret = s3_store_buffer_swap(b, &req, &ret_buf);
    ASSERT_EQ(S3_OK, ret);
    TEST_SWAP_ASSERT_AND_RESET; // will increment offset
    TestRpcCtx *ctx = test_rpc_ctx_get();
    ASSERT_NE(NULL, ctx);
    test_store_buffer_rpc_check_and_do_callback(ctx, NULL, NULL, 0, 0, S3_ERR_TIMEOUT);
    ASSERT_EQ(S3_OK, test_store_buffer_check_status(b));

    ctx = test_rpc_ctx_get();
    ASSERT_NE(NULL, ctx);
    memcpy(data_cmp, src_chunk + rpc_offset, ctx->arg->size);
    test_store_buffer_rpc_check_and_do_callback(ctx, &CHUNK_ID, data_cmp, ctx->arg->size, rpc_offset, S3_OK);
    rpc_offset += ctx->arg->size;
    ASSERT_EQ(S3_OK, test_store_buffer_check_status(b));

    ASSERT_EQ(1, CB_ARG);
  }

  // 第一次超时，第二次仍然超时
  {
    memcpy(data, src_chunk+offset, used_size);
    S3StoreBufferSwapReq req = {
      .data = data,
      .size = size,
      .offset = offset,
      .used_size = used_size,
      .cb = test_store_buffer_cb_func,
      .cb_arg = &CB_ARG,
    };
    ret = s3_store_buffer_swap(b, &req, &ret_buf);
    ASSERT_EQ(S3_OK, ret);
    TEST_SWAP_ASSERT_AND_RESET; // will increment offset
    TestRpcCtx *ctx = test_rpc_ctx_get();
    ASSERT_NE(NULL, ctx);
    test_store_buffer_rpc_check_and_do_callback(ctx, NULL, NULL, 0, 0, S3_ERR_TIMEOUT);
    ASSERT_EQ(S3_OK, test_store_buffer_check_status(b));

    ctx = test_rpc_ctx_get();
    ASSERT_NE(NULL, ctx);
    test_store_buffer_rpc_check_and_do_callback(ctx, NULL, NULL, 0, 0, S3_ERR_TIMEOUT);
    ASSERT_EQ(S3_ERR_TIMEOUT, test_store_buffer_check_status(b));
  }

  test_store_buffer_write_chunks_destroy(s3_g.store_buffer_mgr, b);


  /*
   * 其它错误测试：以CHKSUM为例
   */
  b = test_store_buffer_write_chunks_init(TEST_STORE_BUFFER_CNT_COMMON, chunk_len);
  {
    memcpy(data, src_chunk+offset, used_size);
    S3StoreBufferSwapReq req = {
      .data = data,
      .size = size,
      .offset = offset,
      .used_size = used_size,
      .cb = test_store_buffer_cb_func,
      .cb_arg = &CB_ARG,
    };
    ret = s3_store_buffer_swap(b, &req, &ret_buf);
    ASSERT_EQ(S3_OK, ret);
    TEST_SWAP_ASSERT_AND_RESET; // will increment offset
    TestRpcCtx *ctx = test_rpc_ctx_get();
    ASSERT_NE(NULL, ctx);
    test_store_buffer_rpc_check_and_do_callback(ctx, NULL, NULL, 0, 0, S3_ERR_CHKSUM);
    ASSERT_EQ(S3_ERR_CHKSUM, test_store_buffer_check_status(b));
  }
  test_store_buffer_write_chunks_destroy(s3_g.store_buffer_mgr, b);

  s3_free(data);
  s3_free(data_cmp);

  CHUNKS_CNT = 2;
}

// 每个thread代表一个写chunk
#define THREAD_NUM 3
int con_buffer_cnt = 1000;

static void test_concurrent_callback();

TEST(test_store_buffer, concurrent_callback) {
  CHUNKS_CNT = THREAD_NUM;
  ASSERT_EQ(1, THREAD_NUM <= 5); //不大于CHUNK_ID*个数

  int ret = S3_OK;
  int64_t rpc_offset = 0;
  int size =  TEST_STORE_BUFFER_SIZE_COMMON;
  char *data = s3_malloc(size); char *data_cmp = s3_malloc(size);
  ASSERT_NE(NULL, data); ASSERT_NE(NULL, data_cmp);
  char *ret_buf = NULL;
  int start_offset = 0, offset = 0;
  CB_ARG = 0;

  int chunk_len = con_buffer_cnt * TEST_STORE_BUFFER_SIZE_COMMON + TEST_STORE_BUFFER_SIZE_COMMON;
  S3StoreBuffer *b = test_store_buffer_write_chunks_init(con_buffer_cnt, chunk_len);

  test_set_chunk_data_zero(src_chunk, chunk_len);
  memset(data_cmp, 0, size);

  start_offset = size;
  for (offset = start_offset, rpc_offset = start_offset; offset < chunk_len; ) {
    int used_size = size;
    memcpy(data, src_chunk+offset, size);

    S3StoreBufferSwapReq req = {
      .data = data,
      .size = size,
      .offset = offset,
      .used_size = used_size,
      .cb = test_store_buffer_cb_func,
      .cb_arg = &CB_ARG,
    };
    ret = s3_store_buffer_swap(b, &req, &ret_buf);
    ASSERT_EQ(S3_OK, ret);
    TEST_SWAP_ASSERT_AND_RESET; // will increment offset
  }

  test_concurrent_callback();

  ASSERT_EQ(con_buffer_cnt, s3_atomic_load(&CB_ARG));
  ASSERT_EQ(S3_OK, s3_store_buffer_error(b));

  test_store_buffer_write_chunks_destroy(s3_g.store_buffer_mgr, b);

  s3_free(data);
  s3_free(data_cmp);

  CHUNKS_CNT = 2;
}

pthread_spinlock_t test_queue_lock;

static void *test_store_buffer_do_callback_concurrent() {
  while(1) {
    pthread_spin_lock(&test_queue_lock);
    TestRpcCtx *ctx = test_rpc_ctx_get();
    pthread_spin_unlock(&test_queue_lock);

    if (NULL != ctx) {
      ctx->arg->result = S3_OK;
      s3_store_buffer_rpc_do_callback(ctx->arg);
      test_rpc_ctx_destruct(ctx);
    } else {
      if (con_buffer_cnt == s3_atomic_load(&CB_ARG)) {
        break;
      }
    }
  }

  return NULL;
}

static void test_concurrent_callback() {
  pthread_t threads[THREAD_NUM] = {0};
  pthread_spin_init(&test_queue_lock, PTHREAD_PROCESS_PRIVATE);

  for (int i = 0; i < THREAD_NUM; ++i) {
    pthread_create(&threads[i], NULL, test_store_buffer_do_callback_concurrent, NULL);
  }
  for (int i = 0; i < THREAD_NUM; ++i) {
    pthread_join(threads[i], NULL);
  }

  pthread_spin_destroy(&test_queue_lock);
}
```  
