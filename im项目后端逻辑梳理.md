# im项目后端逻辑梳理

## 接口层

- 10000端口

### auth

登录相关

#### auth/user_register

- 用户注册
- 调用authRPC的user_register函数和user_token函数



#### auth/user_token

- 用户登录，获取token
- token负载为uid和platform
- 调用authRPC的user_token函数



### chat





## 逻辑层

### rpcAuth

#### user_register

- 调用`user_model`的`UserRegister`方法，增加一个用户

#### user_token

- 调用`user_model`的`FindUserByUID`方法，能查到则生成token返回







## 数据层

### user_model

| uid  | name | icon | gender | mobile | birth | email | ex   | create_time |
| ---- | ---- | ---- | ------ | ------ | ----- | ----- | ---- | ----------- |
|      |      |      |        |        |       |       |      |             |

#### UserRegister

- Create() 增

#### FindUserByUID

- 查找uid=?的用户

