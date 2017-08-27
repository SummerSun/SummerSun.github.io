---
layout: post
title:  "It Just Not Works"
---

首先，请理解这种中英文混杂的表达方式。如有不适，且忍着吧。

和 Travis CI 大战几十回合，终于在第二十几回合的时候把 nuget package 成功的 publish 到了 myget.
<!--excerpt-->

## Round 1: script in .travis.yml

```.travis.yml
script:
- npm adduser --registry=https://myget.org/F/generator-azure-iot-edge-module/npm/ <<!
  $NPM_USER
  $NPM_PSW
  $NPM_EMAIL
  !
- npm publish
```
对 bash script 不了解，从 stackoverflow 上看到这种 <<! 的方式 来避免 interactive，直接送进去 credential.
但是 NoNoNo，script 并不知道你是在换行还是在干啥，通通放到一行上跑，而且这样写一点也不美丽。

## Round 2: publish.sh

那写个脚本直接跑吧。

```publish.sh
npm adduser --registry=https://myget.org/F/generator-azure-iot-edge-module/npm/ <<!
$1
$2
$3
!
npm publish
```

```.travis.yml
script:
- bash publish.sh $NPM_USER $NPM_PSW $NPM_EMAIL
```
测试结果和 Travis Agent 一毛一样的错误，可我不知道问题出在哪里，成功 logged in 了呀， npm whoami 也知道我是谁呀。

可一到 publish，就啥不知道了。
Google 大法 npm ERR! code ENEEDAUTH 也没找到啥线索，因为错误很清楚，就是没有 Auth.
可为什么 logged in 刚刚成功了, npm config get registry 也只有我自己 set 的值。摊手，耸肩，摇头。
难道是 bash 定义参数和传参数的时候，单引号和双引号的问题？

测试的时候遇到过，
```
1. NPM_USER="vschina"
2. NPM_PSW="guesswhat"
3. NPM_EMAIL="**@**.com"

4. NPM_USER=vschina
5. NPM_PSW=guesswhat
6. NPM_EMAIL=**@**.com

7. bash test.sh $NPM_USER $NPM_PSW $NPM_EMAIL

8. bash test.sh "" "" ""
9. bash test.sh '' '' ''
```
组合拳 1-2-3-7, logged in succeed, but publish failed.

组合拳 4-5-6-7 just works.

直拳 8 logged in succeed, but publish failed.

直拳 9 just works.

可是 .travis.yml script 直接把环境变量传参数执行，应该就是直接把 value 塞进去，不会有双引号单引号什么事，我也很绝望，天上掉下来一个大拿帮帮我吧。

log from travis ci build:
```
2.89s$ bash publish.sh $NPM_USER $NPM_PSW $NPM_EMAIL
e: ail: (this IS public) ail: (this IS public) Logged in as [secure] on https://www.myget.org/F/generator-azure-iot-edge-module/npm/.
npm ERR! Linux 4.11.6-041106-generic
npm ERR! argv "/home/travis/.nvm/v0.10.48/bin/node" "/home/travis/.nvm/v0.10.48/bin/npm" "publish"
npm ERR! node v0.10.48
npm ERR! npm  v2.15.1
npm ERR! code ENEEDAUTH
npm ERR! need auth auth required for publishing
npm ERR! need auth You need to authorize this machine using `npm adduser`
npm ERR! Please include the following file with any support request:
npm ERR!     /home/travis/build/Azure/generator-azure-iot-edge-module/npm-debug.log
```

## Round 3: deploy in .travis.yml

Travis CI 可以直接 [deploy nuget package](https://docs.travis-ci.com/user/deployment/npm/)
> however if you have a publishConfig.registry key in your package.json then Travis CI publishes to that registry instead.

对应到 package.json 应该是这样的格式:
```package.json
"publishConfig": {"registry": "https://www.myget.org/f/generator-azure-iot-edge-module/npm/"}
```
从本地 .npmrc copy auth token：

```.travis.yml
deploy:
  provider: npm
  email: $NPM_EMAIL
  api_key: $NPM__AUTH_TOKEN
  skip_cleanup: true
  on:
    branch: master
```
然而我还是看到了熟悉的 error:
```
NPM API key format changed recently. If your deployment fails, check your API key in ~/.npmrc.
http://docs.travis-ci.com/user/deployment/npm/
~/.npmrc size: 48
npm ERR! publish Failed PUT 401
npm ERR! Linux 4.11.6-041106-generic
npm ERR! argv "/home/travis/.nvm/v0.10.48/bin/node" "/home/travis/.nvm/v0.10.48/bin/npm" "publish"
npm ERR! node v0.10.48
npm ERR! npm  v2.15.1
npm ERR! code E401
npm ERR! You must be logged in to publish packages. : generator-azure-iot-edge-module
npm ERR! 
npm ERR! If you need help, you may report this error at:
npm ERR!     <https://github.com/npm/npm/issues>
npm ERR! Please include the following file with any support request:
npm ERR!     /home/travis/build/Azure/generator-azure-iot-edge-module/npm-debug.log
```

## Round 4: script 直接写脚本吧
```.travis.yml
script:
- npm config set registry https://www.myget.org/F/generator-azure-iot-edge-module/npm/
- echo 'npm adduser <<!' >> publish.sh
- echo -e $NPM_USER >> publish.sh
- echo -e $NPM_PSW >> publish.sh
- echo -e $NPM_EMAIL >> publish.sh
- echo -e '!' >> publish.sh
- bash publish.sh
- npm publish
```
Just Works.

Travis CI 可以在 settings 里设置不在 build log 里直接打印环境变量，所以没有加密，直接写文件，应该不会有太大的 security 的问题。

前面三个回合，来回测试了很多次，依旧没有找到能 work 的方法。
github npm isuues，travis ci issues，都是想办法怎么 Auth，暂时没找到已经 auth 成功但是依旧不能 publish 的情况。

先 mark 一下这些回合，等我找到解决的方法再来更新。
