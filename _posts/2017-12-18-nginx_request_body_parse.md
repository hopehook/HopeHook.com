---
date: 2017-12-18
layout: post
title: nginx access_log request_body 中文字符解析方法
thread: 2017-12-18-nginx_request_body_parse.md
categories: 经验
tags: nginx
---

#### 问题

nginx 在获取 post 数据时候，request_body 如果是中文，日志内容是一堆乱码。

<pre>
"{\\x22id\\x22:319,\\x22title\\x22:\\x22\\xE4\\xBD\\xB3\\xE6\\xB2\\x9B\\xE9\\x98\\xB3\\xE5\\x85\\x89\\xE9\\x87\\x91\\xE5\\xA5\\x87\\xE5\\xBC\\x82\\xE6\\x9E\\x9C\\xE7\\x8E\\x8B\\xEF\\xBC\\x8822\\xE6\\x9E\\x9A\\xEF\\xBC\\x89\\x22,\\x22intro\\x22:\\x22\\xE8\\xB6\\x85\\xE9\\xAB\\x98\\xE8\\x90\\xA5\\xE5\\x85\\xBB\\xE8\\x83\\xBD\\xE9\\x87\\x8F\\xE6\\x9E\\x9C\\xEF\\xBC\\x8C\\xE4\\xB8\\x80\\xE5\\x8F\\xA3\\xE4\\xB8\\x8B\\xE5\\x8E\\xBB\\xE6\\xBB\\xA1\\xE6\\x98\\xAF\\xE7\\xBB\\xB4C\\x22,\\x22supplier_id\\x22:23,\\x22skus\\x22:[{\\x22create_time\\x22:\\x222017-08-09 22:09:32\\x22,\\x22id\\x22:506,\\x22item_id\\x22:319,\\x22item_type\\x22:\\x22common\\x22,\\x22price\\x22:21800,\\x22project_type\\x22:\\x22find\\x22,\\x22sku_title\\x22:\\x22\\x22,\\x22update_time\\x22:\\x222017-08-09 22:09:32\\x22}],\\x22images\\x22:[\\x22GoodsCommodity/5b3b8558-7d0c-11e7-95d6-00163e0a37a7\\x22],\\x22project_type\\x22:\\x22find\\x22}"
</pre>


#### 解决思路

思路一: 在 nginx 层面解决，中文不进行转义，避免解析。

思路二: 在程序层面解决，想办法解析出来。


#### 具体方法

思路一可以参考 http://www.jianshu.com/p/8f8c2b5ca2d1 ，可以知道 nginx 到底做了些什么，
这个不是本文重点，直接跳过，我们看看思路二。

从思路一得到启发，既然 nginx 遇到中文字符，会处理成 \x22 这样的16进制内容。
那么我们只要遇到 \x22 这种形式的内容，翻译回来即可。


* nginx 转义处理的代码片段

<pre>
static uintptr_t ngx_http_log_escape(u_char *dst, u_char *src, size_t size)
{
    ngx_uint_t      n;
    /* 这是十六进制字符表 */
    static u_char   hex[] = "0123456789ABCDEF";

    /* 这是ASCII码表，每一位表示一个符号，其中值为1表示此符号需要转换，值为0表示不需要转换 */
    static uint32_t   escape[] = {
        0xffffffff, /* 1111 1111 1111 1111  1111 1111 1111 1111 */

                    /* ?>=< ;:98 7654 3210  /.-, +*)( '&%$ #"!  */
        0x00000004, /* 0000 0000 0000 0000  0000 0000 0000 0100 */

                    /* _^]\ [ZYX WVUT SRQP  ONML KJIH GFED CBA@ */
        0x10000000, /* 0001 0000 0000 0000  0000 0000 0000 0000 */

                    /*  ~}| {zyx wvut srqp  onml kjih gfed cba` */
        0x80000000, /* 1000 0000 0000 0000  0000 0000 0000 0000 */

        0xffffffff, /* 1111 1111 1111 1111  1111 1111 1111 1111 */
        0xffffffff, /* 1111 1111 1111 1111  1111 1111 1111 1111 */
        0xffffffff, /* 1111 1111 1111 1111  1111 1111 1111 1111 */
        0xffffffff, /* 1111 1111 1111 1111  1111 1111 1111 1111 */
    };
    
    while (size) {
         /* escape[*src >> 5],escape每一行保存了32个符号，
         所以右移5位，即除以32就找到src对应的字符保存在escape的行，
         (1 << (*src & 0x1f))此符号在escape一行中的位置，
         相&结果就是判断src符号位是否为1，需不需要转换 */
        if (escape[*src >> 5] & (1 << (*src & 0x1f))) {
            *dst++ = '\\';
            *dst++ = 'x';
            /* 一个字符占一个字节8位，每4位转成一个16进制表示 */
            /* 高4位转换成16进制 */
            *dst++ = hex[*src >> 4];
            /* 低4位转换成16进制*/
            *dst++ = hex[*src & 0xf];
            src++;

        } else {
            /* 不需要转换的字符直接赋值 */
            *dst++ = *src++;
        }
        size--;
    }

    return (uintptr_t) dst;
}
</pre>

函数参数: dst 是存在转义后的字符串; src 是原字符串; size 是 sizeof(src); 返回值不用管。
程序逻辑: 
ngx_http_log_escape 函数拿到用户传过来的字符串 src，按照一个字节一个字节处理，遇到不是 ASCII 码表
中的字符，该字符的高4位和低4位分别转成两个16进制数(0123456789ABCDEF)，并用 \x 开头表示。



* 解析处理的 ruby 代码

<pre>
request_body = "{\\x22id\\x22:319,\\x22title\\x22:\\x22\\xE4\\xBD\\xB3\\xE6\\xB2\\x9B\\xE9\\x98\\xB3\\xE5\\x85\\x89\\xE9\\x87\\x91\\xE5\\xA5\\x87\\xE5\\xBC\\x82\\xE6\\x9E\\x9C\\xE7\\x8E\\x8B\\xEF\\xBC\\x8822\\xE6\\x9E\\x9A\\xEF\\xBC\\x89\\x22,\\x22intro\\x22:\\x22\\xE8\\xB6\\x85\\xE9\\xAB\\x98\\xE8\\x90\\xA5\\xE5\\x85\\xBB\\xE8\\x83\\xBD\\xE9\\x87\\x8F\\xE6\\x9E\\x9C\\xEF\\xBC\\x8C\\xE4\\xB8\\x80\\xE5\\x8F\\xA3\\xE4\\xB8\\x8B\\xE5\\x8E\\xBB\\xE6\\xBB\\xA1\\xE6\\x98\\xAF\\xE7\\xBB\\xB4C\\x22,\\x22supplier_id\\x22:23,\\x22skus\\x22:[{\\x22create_time\\x22:\\x222017-08-09 22:09:32\\x22,\\x22id\\x22:506,\\x22item_id\\x22:319,\\x22item_type\\x22:\\x22common\\x22,\\x22price\\x22:21800,\\x22project_type\\x22:\\x22find\\x22,\\x22sku_title\\x22:\\x22\\x22,\\x22update_time\\x22:\\x222017-08-09 22:09:32\\x22}],\\x22images\\x22:[\\x22GoodsCommodity/5b3b8558-7d0c-11e7-95d6-00163e0a37a7\\x22],\\x22project_type\\x22:\\x22find\\x22}"

new_request_body = ''
pt = 0
while pt < request_body.length do
    # 如果是中文, 转码
    if request_body[pt] == '\\' and request_body[pt + 1] == 'x' then
        word = (request_body[pt + 2] + request_body[pt + 3]).to_i(16).chr
        new_request_body = new_request_body + word
        pt = pt + 4
    # 如果是英文, 不处理
    else
        new_request_body = new_request_body + request_body[pt]
        pt = pt + 1
    end
end
puts '翻译结果:'
puts new_request_body
</pre>

上面的 ruby 代码可以直接运行，运行结果如下:

<pre>
{
"id": 319,
"title": "佳沛阳光金奇异果王（22枚）",
"intro": "超高营养能量果，一口下去满是维C",
"supplier_id": 23,
"skus": [
    {
        "create_time": "2017-08-09 22:09:32",
        "id": 506,
        "item_id": 319,
        "item_type": "common",
        "price": 21800,
        "project_type": "find",
        "sku_title": "",
        "update_time": "2017-08-09 22:09:32"
    }
],
"images": [
    "GoodsCommodity/5b3b8558-7d0c-11e7-95d6-00163e0a37a7"
],
"project_type": "find"
}
</pre>

* 解析处理的 c 代码版本

<pre>
#include <stdio.h>
#include <string.h>

unsigned int hexToDec(unsigned char ch)
{
    // "0123456789ABCDEF"
    if (ch >= 'A' && ch <= 'F')
    {
        return ch - 'A' + 10;
    } else
    {
        return ch - '0';
    }
}

void unescape(unsigned char* dst, unsigned char* src, unsigned int src_size)
{
    while (src_size)
    {
        if (*src == '\\' && *(src+1) == 'x')
        {
            unsigned char high_4_bit = (hexToDec(*(src+2))) << 4;  // xxxx 0000
            unsigned char low_4_bit = (hexToDec(*(src+3))) & 0xf;  // 0000 xxxx
            *dst++ = high_4_bit + low_4_bit;
            src += 4;
            src_size -= 4;
        } else
        {
            *dst++ = *src++;
            src_size--;
        }
    }
}

void main()
{
    unsigned char src[] = "{\\x22id\\x22:319,\\x22title\\x22:\\x22\\xE4\\xBD\\xB3\\xE6\\xB2\\x9B\\xE9\\x98\\xB3\\xE5\\x85\\x89\\xE9\\x87\\x91\\xE5\\xA5\\x87\\xE5\\xBC\\x82\\xE6\\x9E\\x9C\\xE7\\x8E\\x8B\\xEF\\xBC\\x8822\\xE6\\x9E\\x9A\\xEF\\xBC\\x89\\x22,\\x22intro\\x22:\\x22\\xE8\\xB6\\x85\\xE9\\xAB\\x98\\xE8\\x90\\xA5\\xE5\\x85\\xBB\\xE8\\x83\\xBD\\xE9\\x87\\x8F\\xE6\\x9E\\x9C\\xEF\\xBC\\x8C\\xE4\\xB8\\x80\\xE5\\x8F\\xA3\\xE4\\xB8\\x8B\\xE5\\x8E\\xBB\\xE6\\xBB\\xA1\\xE6\\x98\\xAF\\xE7\\xBB\\xB4C\\x22,\\x22supplier_id\\x22:23,\\x22skus\\x22:[{\\x22create_time\\x22:\\x222017-08-09 22:09:32\\x22,\\x22id\\x22:506,\\x22item_id\\x22:319,\\x22item_type\\x22:\\x22common\\x22,\\x22price\\x22:21800,\\x22project_type\\x22:\\x22find\\x22,\\x22sku_title\\x22:\\x22\\x22,\\x22update_time\\x22:\\x222017-08-09 22:09:32\\x22}],\\x22images\\x22:[\\x22GoodsCommodity/5b3b8558-7d0c-11e7-95d6-00163e0a37a7\\x22],\\x22project_type\\x22:\\x22find\\x22}";
    unsigned char dst[512];
    memset(dst, 0, sizeof(dst));
    unescape(dst, src, sizeof(src));
    printf("%s\n", dst);
}
</pre>
<br></br>

* 题外话

前面是针对性的解析了 request_body，使得中文可以正常显示出来。假如 nginx access_log 输出的是一个 json，要完整解析，它的日志怎么做呢？

nginx access_log 格式定义如下：
<pre>
 log_format  logstash   '{ "@timestamp": "$time_local", '
                        '"@fields": { '
                        '"status": "$status", '
                        '"request_method": "$request_method", '
                        '"request": "$request", '
                        '"request_body": "$request_body", '
                        '"request_time": "$request_time", '
                        '"body_bytes_sent": "$body_bytes_sent", '
                        '"remote_addr": "$remote_addr", '
                        '"http_x_forwarded_for": "$http_x_forwarded_for", '
                        '"http_host": "$http_host", '
                        '"http_referrer": "$http_referer", '
                        '"http_user_agent": "$http_user_agent" } }';   

 access_log  /data/log/nginx/access.log  logstash;
</pre>
 
完整解析的 ruby 代码实例：
 
<pre>
#!/usr/bin/ruby
# -*- coding: UTF-8 -*-
require 'json'

# nginx access log 日志实例
event = {}
event['message'] = "{ \"@timestamp\": \"17/Dec/2017:00:07:58 +0800\", \"@fields\": { \"status\": \"200\", \"request\": \"POST /api/m/item/add_edit?time=1513440478479 HTTP/1.1\",  \"request_body\": \"{\\x22id\\x22:319,\\x22title\\x22:\\x22\\xE4\\xBD\\xB3\\xE6\\xB2\\x9B\\xE9\\x98\\xB3\\xE5\\x85\\x89\\xE9\\x87\\x91\\xE5\\xA5\\x87\\xE5\\xBC\\x82\\xE6\\x9E\\x9C\\xE7\\x8E\\x8B\\xEF\\xBC\\x8822\\xE6\\x9E\\x9A\\xEF\\xBC\\x89\\x22,\\x22intro\\x22:\\x22\\xE8\\xB6\\x85\\xE9\\xAB\\x98\\xE8\\x90\\xA5\\xE5\\x85\\xBB\\xE8\\x83\\xBD\\xE9\\x87\\x8F\\xE6\\x9E\\x9C\\xEF\\xBC\\x8C\\xE4\\xB8\\x80\\xE5\\x8F\\xA3\\xE4\\xB8\\x8B\\xE5\\x8E\\xBB\\xE6\\xBB\\xA1\\xE6\\x98\\xAF\\xE7\\xBB\\xB4C\\x22,\\x22supplier_id\\x22:23,\\x22skus\\x22:[{\\x22create_time\\x22:\\x222017-08-09 22:09:32\\x22,\\x22id\\x22:506,\\x22item_id\\x22:319,\\x22item_type\\x22:\\x22common\\x22,\\x22price\\x22:21800,\\x22project_type\\x22:\\x22find\\x22,\\x22sku_title\\x22:\\x22\\x22,\\x22update_time\\x22:\\x222017-08-09 22:09:32\\x22}],\\x22images\\x22:[\\x22GoodsCommodity/5b3b8558-7d0c-11e7-95d6-00163e0a37a7\\x22],\\x22project_type\\x22:\\x22find\\x22}\", \"request_time\": \"0.041\", \"body_bytes_sent\": \"702\", \"remote_addr\": \"100.120.141.124\", \"http_x_forwarded_for\": \"-\", \"http_host\": \"api.dev.domain.com\", \"http_referrer\": \"https://test.dev.domain.com/\", \"http_user_agent\": \"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_5) AppleWebKit/603.2.4 (KHTML, like Gecko) Version/10.1.1 Safari/603.2.4\" } }"

message = event['message']
# 避免转义的request_body被解析parse
message = message.gsub('\\x', '\\\\\\x')
message_obj = JSON.parse(message)
request_body = message_obj['@fields']['request_body']
if request_body != '-' then
    # 如果 request_body 有 json 内容, 进行转码处理, 然后解析 parse
    new_request_body = ''
    pt = 0
    while pt < request_body.length do
        # 如果是中文, 转码
        if request_body[pt] == '\\' and request_body[pt + 1] == 'x' then
            word = (request_body[pt + 2] + request_body[pt + 3]).to_i(16).chr
            new_request_body = new_request_body + word
            pt = pt + 4
        # 如果是英文, 不处理
        else
            new_request_body = new_request_body + request_body[pt]
            pt = pt + 1
        end
    end
    new_request_body_obj = JSON.parse(new_request_body)
    message_obj['@fields']['request_body'] = new_request_body_obj
end
    
event['message_json'] = JSON.generate(message_obj)

puts '翻译结果:'
puts event['message_json']

</pre>

这个代码可以应用于 logstash indexer 的配置文件 filter 部分，实现 elk 对 nginx access_log 的解析。
