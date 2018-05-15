---
title: github_travis_coveralls自动集成自动部署
tags: [CICD]
---
# github_travis_coveralls自动集成自动部署
近期准备开发一个指数定投金额计算的小项目，源码托管在[github](https://github.com/dumingcode/my-fintech-datacenter)上。之前都是本地测试通过后，使用`XSHELL`将代码传输到部署服务器。后来自己研究了下`jenkins`，并借助`jenkins`实现将`hexo`博客自动部署，详见之前写的[文章](http://blog.buyasset.club/2018/04/21/2018-04-21-hexo%E4%BD%BF%E7%94%A8jenkins%E8%87%AA%E5%8A%A8%E9%83%A8%E7%BD%B2%E5%88%B0%E9%98%BF%E9%87%8C%E4%BA%91/)。但是首先`jenkins`需要自己部署，而且毕竟代码托管在`github`，干脆秉承一切都开源的精神，折腾一下将`github`,`travis`,`coveralls`结合起来，实现`CICD`。
## travis配置
### 注册配置travis
直接使用`github`的账号登陆`travis`，登陆之后如下图所示添加自己的`github repo`。   
![add repo](/images/travis_addrepo.png)   
添加完毕后，`setting`如下图所示。    
![add repo](/images/travis_setting.png)    

### travis自动集成
简单说这个功能就是实现代码上传travis后，能够travis自动执行测试过程。本人使用的是nodejs开发项目，所以先在`package.json`文件中定义了`npm test`命令，travis通过执行`npm test`命令完成自动构建。

#### travis mocha测试异步代码不退出
在测试过程中，遇到一个大坑折腾很久，项目中含有异步调用测试案例，导致travis测试通过后一直运行不退出，后来只能在`npm test`脚本命令中添加`--exit`。另外travis默认异步调用等待时间是2秒，所以需要在命令中使用`-t 10000`人为延长travis等待时间。具体`npm test`命令为：
```
 "name": "my-fintech-datacenter",
    "version": "1.0.0",
    "description": "data-center",
    "main": "schedule.js",
    "scripts": {
        "test": "mocha -t 10000 ./test/tasktest.js --exit"
       }
    }
```

#### redis服务设置秘钥
travis支持在测试前启用`mysql redis mongodb`等服务，但是我的代码中对`redis`设置了秘钥，而且犯懒了不太想改代码，然后找到了如下解决方案，在`travis.yml`配置文件中添加:`before_script: sudo redis-server /etc/redis/redis.conf --requirepass  $redis_password` ，注意命令中的`$redis_password`是在`travis setting`中设置的加密环境变量。    
后面还会讲到travis还会针对文件进行加密，比如一些项目配置文件以及SSH秘钥都需要进行文件加密。


## travis代码覆盖率集成
这个功能主要是借助[coveralls](https://coveralls.io)实现。首先`package.json`文件中,定义一条命令：
```
"test-cov": "./node_modules/istanbul/lib/cli.js cover ./node_modules/mocha/bin/_mocha -- --timeout 10000 -R spec ./test/  --exit"
```
测试依赖包如下所示：
```
  "devDependencies": {
        "chai": "^4.1.2",
        "coveralls": "^3.0.1",
        "istanbul": "^0.4.5",
        "istanbul-harmony": "^0.3.16",
        "mocha": "^5.1.1",
        "supertest": "^3.0.0"
    }
```
### coveralls网站配置
登陆[coveralls](https://coveralls.io)，`coveralls`也是关联`github`的账号直接登陆。如下图所示添加github中的repo。   
![add repo](/images/coveralls_repo.png)     
添加repo完毕后，需要记录下下图所示的`token`。   
![add repo](/images/coveralls_token.png)  
### travis.yml 配置
```
after_success:
  - npm run test-cov
after_script: cat ./coverage/lcov.info | ./node_modules/coveralls/bin/coveralls.js -repotoken $COVERALLS_TOKEN   
  
```
注意上述代码中的`$COVERALLS_TOKEN `，同样也是在travis setting中加密。
## 自动部署
在上面两步自动构建完毕之后，需要将测试通过的代码直接提交到部署服务器，然后重启部署服务器的服务。   
从travis构建服务器到部署服务器，需要使用SSH协议。travis服务器中需要存储有私钥(*_rsa files)，而远端部署服务器需要部署公钥(*_rsa.pub files)。    
但是私钥肯定是不能存储到git repo或者显示在travis的构建日志中。幸运的是，travis提供了对文件的加密功能，可以借助此功能，将加密后的私钥放到github repo中。  
**注**：此处说的ssh 公钥 私钥都是在部署服务器中生成的。私钥加密后传到github，公钥继续留在部署服务器。    
travis文件加密需要安装travis客户端，推荐linux上安装。   
目标（想登陆的机器）主机存公钥，源机器存私钥。

### travis客户端安装
```
yum install ruby ruby-devel
yum install gem
gem update --system
#添加源
gem sources --add https://gems.ruby-china.org/
gem install travis
travis login
输入github用户密码即可
```
### deploy server生成ssh key
登陆deploy server，执行下面的代码。**注意：**私钥需要经过travis加密，然后传到github repo 。对待**公钥则直接执行**`ssh-copy-id`命令。
```
ssh-keygen -t rsa -b 4096 -C 'build@travis-ci.org' -f ./deploy_rsa
travis encrypt-file deploy_rsa --add
ssh-copy-id -i deploy_rsa.pub <ssh-user>@<deploy-host>

rm -f deploy_rsa deploy_rsa.pub
```
travis加密文件的执行步骤如下所示：
```
[root@iz2ze1fd7d8ota0f9ysaazz .ssh]# [root@iz2ze1fd7d8ota0f9ysaazz .ssh]# travis encrypt-file id_rsa -r dumingcode/my-fintech-datacenter
encrypting id_rsa for dumingcode/my-fintech-datacenter
storing result as id_rsa.enc
storing secure env variables for decryption

Please add the following to your build script (before_install stage in your .travis.yml, for instance):

    openssl aes-256-cbc -K $encrypted_383bc2ea2d21_key -iv $encrypted_383bc2ea2d21_iv -in id_rsa.enc -out id_rsa -d

Pro Tip: You can add it automatically by running with --add.

Make sure to add id_rsa.enc to the git repository.
Make sure not to add id_rsa to the git repository.
Commit all changes to your .travis.yml.

```
**还有一点要注意** travis第一次登录远程服务器会出现 SSH 主机验证，这边会有一个主机信任问题。官方给出的方案是添加 addons 配置：
```
addons:
  ssh_known_hosts: your-ip
```

### 自动部署脚本
自动部署脚本如下所示。本人是采用的`rsync`命令从travis上将有变化的文件传输到部署服务器，这个命令比scp命令要好。`after_deploy`命令执行的是`pm2 restart`，要求nodejs服务至少之前在服务器上执行一次，要不restart要报错。
```
deploy:
  provider: script
  skip_cleanup: true
  script: rsync -r --delete-after --quiet $TRAVIS_BUILD_DIR $deploy_user@39.107.119.46:$DEPLOY_PATH
  on:
    branch: master

after_deploy:
  - ssh $deploy_user@your-ip "pm2 restart datacenter"
addons:
  ssh_known_hosts: your-ip
```
### 完整代码
下面贴上travis.yml和package.json文件的全部信息。项目github的地址为[github](https://github.com/dumingcode/my-fintech-datacenter)。
travis.yml文件:
```
sudo: false
language: node_js
node_js:
  - 8
before_install: 
  - openssl aes-256-cbc -K $encrypted_1fc90f464345_key -iv $encrypted_1fc90f464345_iv -in ./config/config.js.enc -out ./config/config.js -d
  - openssl aes-256-cbc -K $encrypted_383bc2ea2d21_key -iv $encrypted_383bc2ea2d21_iv -in id_rsa.enc -out ~/.ssh/id_rsa -d
  - chmod 600 ~/.ssh/id_rsa
before_script: sudo redis-server /etc/redis/redis.conf --requirepass  $redis_password
notifications:
  email:
    recipients:
    - jake1036@126.com
    on_success: always
    on_failure: always
script: 
  - npm test
after_success:
  - npm run test-cov
deploy:
  provider: script
  skip_cleanup: true
  script: rsync -r --delete-after --quiet $TRAVIS_BUILD_DIR $deploy_user@39.107.119.46:$DEPLOY_PATH
  on:
    branch: master

after_deploy:
  - ssh $deploy_user@39.107.119.46 "pm2 restart datacenter"
addons:
  ssh_known_hosts: 39.107.119.46 # 请替换成自己的服务器IP
after_script: cat ./coverage/lcov.info | ./node_modules/coveralls/bin/coveralls.js -repotoken $COVERALLS_TOKEN   
```
package.json文件内容：
```
{
    "name": "my-fintech-datacenter",
    "version": "1.0.0",
    "description": "data-center",
    "main": "schedule.js",
    "scripts": {
        "test": "mocha -t 10000 ./test/tasktest.js --exit",
        "test-cov": "./node_modules/istanbul/lib/cli.js cover ./node_modules/mocha/bin/_mocha -- --timeout 10000 -R spec ./test/  --exit"
    },
    "repository": {
        "type": "git",
        "url": "git+https://github.com/dumingcode/my-fintech-datacenter.git"
    },
    "keywords": [
        "datacenter"
    ],
    "author": "duming",
    "license": "MIT",
    "bugs": {
        "url": "https://github.com/dumingcode/my-fintech-datacenter/issues"
    },
    "homepage": "https://github.com/dumingcode/my-fintech-datacenter#readme",
    "dependencies": {
        "axios": "^0.18.0",
        "bunyan": "^1.8.12",
        "ioredis": "^3.2.2",
        "node-schedule": "^1.3.0"
    },
    "devDependencies": {
        "chai": "^4.1.2",
        "coveralls": "^3.0.1",
        "istanbul": "^0.4.5",
        "istanbul-harmony": "^0.3.16",
        "mocha": "^5.1.1",
        "supertest": "^3.0.0"
    }
}
```


## 参考   
感谢以下各位大神。  
[Travis CI 系列：自动化部署博客](https://segmentfault.com/a/1190000011218410)    
[SSH deploys with Travis CI](https://oncletom.io/2016/travis-ssh-deploy/)   
[Building Better npm Modules with Travis and Coveralls](https://strongloop.com/strongblog/npm-modules-travis-coveralls/)