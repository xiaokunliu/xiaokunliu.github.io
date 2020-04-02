---
title: python微服务设计
category: Python
date: 2020-03-12 18:36:59
tags: Python
---

<!-- more -->
##### Nameko + API Swagger

###### 创建项目

```bash
## 安装微服务框架
pip install nameko==2.5.4.4
## 安装api框架
pip install nameko-swagger==1.2.7
## 创建项目
nameko-admin createproject demo
```

> 项目目录结构

```
demo/
    .tox/   
    bin/
        run.sh
    conf/
        config.yaml
    logs/
    service/
        demo.py
    spec/
        v1/
            api.yaml
    test/
    ... ##
```

###### 默认配置

```yaml
## api.yaml
paths:
  /demo/health:
    get:
      summary: 获取服务健康状态
      tags:
        - demo
      operationId: get_health
      responses:
        default:
          description: |
            返回结果：{"code": "xxx", "msg": "xxx", "data": {}, "time": xxx, "why": "xxx"}

            其中`code`字段可能返回的错误码包括：
            * "OK" 操作成功，结果见`data`字段
            * "INTERNAL_SERVER_ERROR" 内部错误，具体原因见`why`字段
          schema:
            $ref: '#/definitions/HealthResponse'
```

```python
## demo.py
@swagger.unmarshal_request
@http('GET', '/v1/demo/health')
def http_get_health(self, request):
    return dict(code='OK', msg='', data={}, time=int(time.time()))
```

###### 配置在路径path的作用域中

```yaml
## api.yaml添加path的路径参数配置
/demo/path/index/{uid}:
    get:
      summary: 根据请求参数在path中进行查询
      tags:
        - demo
      operationId: index
      parameters:
        - name: uid
          in: path        # Note the name is the same as in the path
          description: 用户uid
          required: true
          type: integer
      responses:
        default:
          description: |
            返回结果：{"code": "xxx", "msg": "xxx", "data": {}, "time": xxx, "why": "xxx"}

            其中`code`字段可能返回的错误码包括：
            * "OK" 操作成功，结果见`data`字段
            * "INTERNAL_SERVER_ERROR" 内部错误，具体原因见`why`字段
          schema:
            $ref: '#/definitions/HealthResponse'
```

```python
## demo.py
@swagger.unmarshal_request
@http('GET', '/v1/demo/path/index/<int:uid>')
def path_index(self, request, uid):
    logger.info(request)
    logger.info(uid)
    return dict(code='OK', msg='', data={r"msg": "path index success..."}
```

###### 配置在路径query作用域中

```yaml
## api.yaml
/demo/query/index:
    get:
      summary: 根据请求参数在query中查询
      operationId: query_index
      parameters:
        - name: cid
          in: query
          description: 用户频道Id
          required: true
          type: integer
      responses:
        default:
          description: |
           返回结果：{"code": "xxx", "msg": "xxx", "data": {}, "time": xxx, "why": "xxx"}

            其中`code`字段可能返回的错误码包括：
            * "OK" 操作成功，结果见`data`字段
            * "INTERNAL_SERVER_ERROR" 内部错误，具体原因见`why`字段
          schema:
            $ref: '#/definitions/HealthResponse'
```

```python
## demo.py
@swagger.unmarshal_request
@http('GET', '/v1/demo/query/index')
def query_index(self, request, cid):
    logger.info(request)
    logger.info(cid)
    return dict(code='OK',
                msg='',
                data={"cid": cid},
                time=int(time.time()))
```

###### 配置在query中查询多个

```yaml
## api.yaml
/demo/query/many:
    get:
      summary: 根据请求参数在query中查询
      operationId: query_many
      parameters:
        - name: cid
          in: query
          description: 用户频道Id
          required: true
          type: integer
          minimum: 1
        - name: uid
          in: query
          description: 用户id
          required: false
          type: integer
          minimum: 1
        - name: channelid
          in: query
          required: false
          type: string
      responses:
        default:
          description: |
           返回结果：{"code": "xxx", "msg": "xxx", "data": {}, "time": xxx, "why": "xxx"}

            其中`code`字段可能返回的错误码包括：
            * "OK" 操作成功，结果见`data`字段
            * "INTERNAL_SERVER_ERROR" 内部错误，具体原因见`why`字段
          schema:
            $ref: '#/definitions/HealthResponse'
```

```python
## demo.py
@swagger.unmarshal_request
@http('GET', '/v1/demo/query/many')
def query_many(self, request,
               cid,
               uid=0,
               channelid=0):
    logger.info(request)
    logger.info(cid)
    logger.info(uid)
    logger.info(channelid)
    return dict(code='OK',
                msg='',
                data={"cid": cid,
                      "uid": uid,
                      "channelid": channelid},
                time=int(time.time()))
```

###### 在post的表单中查询

```yaml
## api.yaml
/demo/post:
    post:
      summary: 根据请求参数在post的表单中查询
      operationId: do_post
      parameters:
        - name: cid
          in: formData
          description: 用户频道Id
          required: true
          type: integer
          minimum: 1
        - name: uid
          in: formData
          description: 用户id
          required: false
          type: integer
          minimum: 1
        - name: channelid
          in: formData
          required: false
          type: string
      responses:
        default:
          description: |
           返回结果：{"code": "xxx", "msg": "xxx", "data": {}, "time": xxx, "why": "xxx"}

            其中`code`字段可能返回的错误码包括：
            * "OK" 操作成功，结果见`data`字段
            * "INTERNAL_SERVER_ERROR" 内部错误，具体原因见`why`字段
          schema:
            $ref: '#/definitions/HealthResponse'
```

```python
## demo.py
@swagger.unmarshal_request
@http('POST', '/v1/demo/post')
def do_post(self, request,
               cid,
               uid=0,
               channelid=""):
    logger.info(request)
    logger.info(cid)
    logger.info(uid)
    logger.info(channelid)
    return dict(code='OK',
                msg='do post message',
                data={"cid": cid,
                      "uid": uid,
                      "channelid": channelid},
                time=int(time.time()*1000))
```

###### 配置在post中查询多个

```yaml
## api.yaml
/demo/post/query:
    post:
      summary: 根据请求参数在post的表单以及query的路径中查询
      operationId: query_post
      parameters:
        - name: cid
          in: query
          description: 用户频道Id
          required: true
          type: integer
          minimum: 1
        - name: uid
          in: query
          description: 用户id
          required: false
          type: integer
          minimum: 1
        - name: channelid
          in: formData
          required: false
          type: string
      responses:
        default:
          description: |
           返回结果：{"code": "xxx", "msg": "xxx", "data": {}, "time": xxx, "why": "xxx"}

            其中`code`字段可能返回的错误码包括：
            * "OK" 操作成功，结果见`data`字段
            * "INTERNAL_SERVER_ERROR" 内部错误，具体原因见`why`字段
          schema:
            $ref: '#/definitions/HealthResponse'
```

```python
## demo.py
@swagger.unmarshal_request
@http('POST', '/v1/demo/post/query')
def query_post(self, request,
               cid,
               uid=0,
               channelid=""):
    logger.info(request)
    logger.info(cid)
    logger.info(uid)
    logger.info(channelid)
    return dict(code='OK',
                msg='query post message',
                data={"cid": cid,
                      "uid": uid,
                      "channelid": channelid},
                time=int(time.time() * 1000))
```

###### 配置在body中

```yaml
## api.yaml
/demo/post/body:
    post:
      summary: 根据请求参数在body中查询
      operationId: query_body
      parameters:
        - in: body
          name: jsonParameters
          schema:
            $ref: '#/definitions/RequestJson'
      responses:
        default:
          description: |
           返回结果：{"code": "xxx", "msg": "xxx", "data": {}, "time": xxx, "why": "xxx"}

            其中`code`字段可能返回的错误码包括：
            * "OK" 操作成功，结果见`data`字段
            * "INTERNAL_SERVER_ERROR" 内部错误，具体原因见`why`字段
          schema:
            $ref: '#/definitions/HealthResponse'

## 定义的requestJson格式
definitions:
  RequestJson:
    description: 请求参数在body中的json格式
    type: object
    properties:
      cid:
         description: 用户的cid
         type: integer
      uid:
         description: 用户的uid
         type: integer
      channelid:
         description: 频道Id
         type: string
    required:
      - cid
```

```python
## demo.py
@swagger.unmarshal_request
@http('POST', '/v1/demo/post/body')
def query_body(self, request, jsonParameters):
    u"""
    jsonParameters名称要和api.yaml上声明的name一致,请求的body的json格式为
    {
        cid:"",
        uid:"",
        channelid:""
    }
    ## 使用这种可以传递一个对象的json数据而避免使用过长的参数列表来进行request请求
    """
    return dict(code='OK',
                msg='query post message',
                data=jsonParameters,
                time=int(time.time() * 1000))
```