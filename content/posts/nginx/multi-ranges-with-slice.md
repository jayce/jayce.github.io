---
title: "Nginx Slice 模块支持 multi-ranges"
date: 2020-07-01T12:09:15+08:00
description:
draft: false
hideToc: false
enableToc: true
enableTocContent: false
tocPosition: inner
tocLevels: ["h2", "h3", "h4"]
tags:
- nginx
image:
---

## 起因
起因是对一个旧项目的技术更迭。这种项目的初期，自然是要花了很长的时间来做调研。因为都是 HTTP 协议，可以先抓包观察请求和响应的内容，从中找到异样的点。

偶然抓到了带 `Range` 的请求，且不是常见的 `Range: bytes=18099184-18165549` 格式。随问了，什么场景需要读取同一个资源的不同位置的数据呢？无果，只能先记下。然后，对于这种项目肯定要秉持一种原则，就是得支持，不然 --- 。

支持应该也不是很难，先来**感受一下：**
```txt
> GET ********** HTTP/1.1
> ....
> Range: bytes=18099184-18165549,392163-395061,107934-110397,98322-98498,
> 200261-202919,590655-596843,100554-103007,105471-107933,256379-259075,
> 115337-117812,171343-173950,103008-105470,267181-269885,512861-517579,
> 349450-352216,275312-278038,232237-234909,327379-330130,14248467-14300167,
> 221556-224221,330131-332882,125348-127866,435522-439086,291708-294446,
> 297186-299925,166145-168736,261776-264475,380585-383472,412567-415595,
> 117813-120318,313638-316383,9446656-9482385,269886-272595,415596-418738,
> 316384-319131,806912-814940,242936-245614,324629-327378,321880-324628,208239-210900
> ...
```

## 术语
> 这里的两个词是平时沟通中使用的，意思是把属性附加到请求上了，指的是一种请求。实际上在 HTTP 协议里面称为 single-part 或是 multiple-parts 。

* **single-range**
指 HEADER 中带有 “Range: bytes=0-2” 内容的请求。服务端如果可以满足字节范围，就响应 206 并发送对应 “Range” 所指的内容给客户端。书写格式为：`Range: <unit>=<range-start>-`。

* **multi-ranges**
与 single-range 一样，只不过 HEADER 变成了  “Range: bytes=0-2,3-4,10-20” 。同样的，如果服务端能够满足这些字节范围，就可以响应 206 ，但是把内容封装成 `multipart/byteranges;boundary=THIS_STRING_SEPARATES` 格式，再发送给客户端。书写格式为：`Range: <unit>=<range-start>-<range-end>, <range-start>-<range-end>, <range-start>-<range-end>`

两种情况的区别就是传输格式不同。single-range 的内容读到的就是实际内容。相反，读到 multi-rangs 的内容后还需要做一点点解码工作。当然这个解码工作其实非常简单，而现在 HTTP 非常流行，各种库或者框架应该都封装好了，应该也不用操心。

## 同源
好在以前项目是基于非常旧的 1.2.x 的 nginx 源码的，而新的技术栈基于 openresty ，本质上是一样的。

Nginx 的 range 模块是支持 `multi-ranges` 请求的，该模块在 body_filter 阶段把输出的内容按照 ranges 的范围进行重新拼装。不过目前只能对静态文件有效，并且其内容只能用一个 buf 来表示。

Nginx 的 slice 模块支持了一种分片机制，与缓存机制同时使用会有比较好的效果，等于可以缓存资源的某个片段。启用这个特性之后，对于 nginx 内部来讲一个资源可就变成多个相互独立的资源了。Nginx 在加入 slice 模块的时候对 range  模块也做了相应的修改，不过只支持了 `single-range` 。

我做调查的时候就在想，为啥当时不把 `multi-ranges` 一起给实现了呢？；使用的场景？没想出来；断点续传？也没有这样做续传的吧。

目前的情况是，slice 好用，得开；有这类请求，得支持。

## 事前分析
Nginx 作为代理，为了尽快让客户端得到反馈，设计了一个流式输出机制。这个机制可以把收到的 upstream 的数据同步发送给 downstream 。数据流必经之地就是 body_filter 阶段，想要对内容做手脚的模块就必须挂载到这个阶段。

这个阶段最重要的参数就是 `ngx_chain_t` 对象，它是一个把 `ngx_buf_t` 串联起来的链表结构。每个 body_filter 实际上就是在调整这个链表的长度或者是修改 buf 的内容。

Nginx 把输出内容的一些状态（什么时候没内容了）放到了 `ngx_buf_t` 结构体中，用位标记表示，实际在处理链表的时候，是要调整 buf 的标记位的。有个重要的标记：`buf->last_buf` ，其表示的就是链表中最后一个 buf ，不会再有内容了。

再看看 range 模块，实际上就是在遍历链表，找到 start 所在的 buf，把不满足 start 的 buf 删除，再找到 end 所在的 buf ，设置 `buf->last_buf` 并把之后的节点删除。另外，遍历过程中，需要把迭代过的 buf 标记位重置，这是因为可能会有别的模块设置了，比如 slice 模块。

再再看看 slice 模块加进来之后 range 模块有什么改动，对应的[提交](https://github.com/nginx/nginx/commit/8ba626ccd71cbd704c7c69928d1d6fe58fd0445f)记录。这次提交最核心的地方就是 `ctx->offset = r->headers_out.content_offset;` 这行代码，这是表示了响应内容的偏移量。主要用途就是支持 slice 的分片请求，因为这种响应内容是从某个固定的偏移开始的，那 range 模块必须要知道是从哪里开始的。

slice 还必须分析请求 HEADER 中的 Range ，以便知道应该从哪里开始，并把启始偏移量设置到 `r->headers_out.content_offset` ，这样 range 模块就知道上下文了。另外， slice 只是针对发送到上游的请求，所以无论 Range 的值是不是合法的，都会工作。对于不合法的 Range 由 range 模块来处理。

slice 的实现利用了内部子请求机制。子请求对于模块来说与正常请求没什么不同，什么意思？意思是子请求也有输出链。会把一个请求拆成了多个子请求，并且按顺序发送给 upstream ，且所有的 body_filter 模块都会处理子请求的内容。`slice_header_filter` 会调整 `r->headers_out.content_offset` 的值；`range_body_filter` 会根据 `r->headers_out.content_offset` 来调整主请求和子请求的输出链。`slice_body_filter` 要对输出链内容需要做调整，就是把最后一个 buf 的 last_buf 标记清除了，不然会提前中止主请求。其他处理（拼接输出链）交给子请求的一个核心模块来处理（`postpone`）。

**slice 模块在主请求中调整偏移量*
```c
    r->headers_out.status = NGX_HTTP_OK;
    r->headers_out.status_line.len = 0;
    r->headers_out.content_length_n = cr.complete_length; // 整个内容的长度
    r->headers_out.content_offset = cr.start; // content_offset 在 range 模块中使用
    r->headers_out.content_range->hash = 0;
    r->headers_out.content_range = NULL;

    .....

    rc = ngx_http_next_header_filter(r);

	  /* 跳过子请求，不需要计算 */
    if (r != r->main) {
        return rc;
    }

    /* 主请求中对齐 range 请求的范围
     * content_offset 被 range 模块修改了，改成了请求实际需要的启始位置
     * content_length_n 被 range 模块修改成了实际需要的内容长度
     * 也因此这里需要重新校对一下 start 和 end，避免发送多余的子请求
     */
    if (r->headers_out.status == NGX_HTTP_PARTIAL_CONTENT) {
        if (ctx->start + (off_t) slcf->size <= r->headers_out.content_offset) {
            ctx->start = slcf->size
                         * (r->headers_out.content_offset / slcf->size);
        }

        ctx->end = r->headers_out.content_offset
                   + r->headers_out.content_length_n;

    } else {
        ctx->end = cr.complete_length;
    }
```

## 改进思路
根据 slice 的实现分析，只处理了 single-range 的请求，那么要支持 multi-ranges 只需要把表示 range 的结构体变成数组就可行了吧？

```
 HTTP/1.1 206 Partial Content
 Date: Wed, 15 Nov 1995 06:25:24 GMT
 Last-Modified: Wed, 15 Nov 1995 04:58:08 GMT
 Content-Length: 1741
 Content-Type: multipart/byteranges; boundary=THIS_STRING_SEPARATES

 --THIS_STRING_SEPARATES
 Content-Type: application/pdf
 Content-Range: bytes 500-999/8000

 ...the first range...
 --THIS_STRING_SEPARATES
 Content-Type: application/pdf
 Content-Range: bytes 7000-7999/8000

 ...the second range
 --THIS_STRING_SEPARATES--
```

multi-ranges 与 single-range 不同点在于对内容的封装格式，而响应 HEADER 是事先在 range_header_filter 阶段就准备好了的，后面发送就好。要实现，意味着需要记录 multi-ranges 中每个 range 的处理状态，未处理、处理了一半、已处理？。

nginx 支持的 multi-ranges 非常灵活，Range 内容可以是 `-10, 10-, 0-10` 这种。加上 slice 之后 body_filter 阶段得到的输出链，可能只包含一点内容。

这就要 slice 模块根据 Range 值的顺序发起子请求，等某个 range 的内容满足之后，重置状态，再处理下一个，直到结束。range 模块实际也要按照相同的逻辑来处理，另外还要考虑内容封装的逻辑。

其实还可以考虑把 range 合并，这样发送给 upstream 的请求会少一些，不过这会让 range 模块的实现复杂一点。

摸一摸几个层次的上下文之后，想法是不是自然会出现？