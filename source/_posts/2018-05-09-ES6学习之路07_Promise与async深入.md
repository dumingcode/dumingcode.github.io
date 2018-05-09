---
title: ES6学习之路07_Promise与async深入
tags: [ES6,nodejs]
---
# ES6-学习之路-07 async Promise深入
继续深入对js异步操作的学习，主要针对async和await。
## Promise方式
首先继续回顾Promise，Promise针对异步回调的处理方式，是通过链式，每一步resole data，并将数据传递给下一个then函数。  
```
new Promise(function(resolve, reject) {
  // if OK
  resolve(data);
  // if ERROR
  reject(error);
}).then(function(data) {
  return turtle;
}).then(function(turtle) {
  return turtleToRat
}).catch(function(error) {
  // handle an error
});
```
fetch函数针对大多数浏览器都适用并且这个函数会返回一个Promise，下面是一个具体示例，我们会根据用户名访问github的用户信息和用户名下的仓库。
```
var fetch = require('node-fetch')

function fetchGitProfile(username) {
    return fetch(`https://api.github.com/users/${username}`)
        .then((data) => data.json())
        .then(({ bio, company, followers, following, repos_url }) => ({
            bio,
            company,
            followers,
            following,
            repos_url
        }));
}

function includeGitRepos(user) {
    return fetch(user.repos_url)
        .then((data) => data.json())
        .then((data) => data.map(({ name, stargazers_count }) => ({
            name,
            stargazers_count
        })))
        .then((repoList) => {
            return {
                ...user,
                repoList
            };
        });
}

function log(data) {
    console.log(data);
}

fetchGitProfile('dumingcode')
    .then(includeGitRepos)
    .then(log);
```
以上代码也许仅仅比回调函数写的好理解一些，而且还需要深入理解Promise代码。

## 使用async/await
async/await 由ES7引入，这两个关键词需要共同使用，await必须在async函数中使用，绝对不能单独使用。一个有趣的特性是，这两个关键字适配Promise。如果一个函数返回结果是Promise你可以用await去resole，或者针对async函数的返回使用then去解析。     
### 基本用法
```
async function resolveMyData() {
  const data = await fetchData('/a');
  return await fetchMoreData('/b/' + data.id);
}
```
改写上述github代码：
```
var fetch = require('node-fetch')

async function fetchGitProfile(username) {
    const userinfo = await fetch(`https://api.github.com/users/${username}`)
    let { bio, company, followers, following, repos_url } = await userinfo.json()
    return { bio, company, followers, following, repos_url }
}

async function includeGitRepos(repoUrl) {
    const repo = await fetch(repoUrl)
        .then((data) => data.json());

    return repo.map(({ name, stargazers_count }) => ({
        name,
        stargazers_count
    }));
}

async function resolveGithubProfile() {
    const profile = await fetchGitProfile('duming');
    const repoList = await includeGitRepos(profile.repos_url);
    console.log({
        ...profile,
        repoList
    });
};

resolveGithubProfile();
```
上述代码中`includeGitRepos`中的fetch API后跟着一个then函数，fetch函数仍然会返回一个Promise。
### 并行异步调用
没必要针对每一个函数都使用一个await函数，可以并行地调用，并逐个处理每个请求的返回值:
```
async function resolveGithubProfileParallel() {
    let dumingPromise = fetchGitProfile('dumingcode')
    let octocatPromise = fetchGitProfile('octocat')
    const duming = await dumingPromise;
    const octocat = await octocatPromise; // this will complete in the same time as rkotzePromise.
    return [duming, octocat, "done!"];
}

resolveGithubProfileParallel().then((data) => {
    console.log(data)
})
```

### 使用promise all
```
Promise.all([fetchGitProfile('rkotze'), fetchGitProfile('octocat')]).then(function(values) {
    console.log(values);
});

输出（注意输出的是一个数组）：

[ { bio: 'Software engineer. Currently into ES6, ReactJS, NodeJS and Elixir.',
    company: '@findmypast',
    followers: 24,
    following: 52,
    repos_url: 'https://api.github.com/users/rkotze/repos' },
  { bio: null,
    company: 'GitHub',
    followers: 2238,
    following: 5,
    repos_url: 'https://api.github.com/users/octocat/repos' } ]

```

### 异步测试框架
优先研究一下Mocha  
[Mocha](https://mochajs.org)  
[Jest](https://facebook.github.io/jest/)  
[Jasmine](https://jasmine.github.io/) 
### codewar挑战
如果能通过这个挑战，说明你掌握了js异步，点击链接
[code war异步挑战](https://www.codewars.com/kata/jokes-youve-been-awaiting-for-dot-dot-dot-promise/javascript)





## 参考
感谢大神们，让我站在你们的肩膀上。   
[promises-async-await-testing](https://www.richardkotze.com/coding/promises-async-await-testing)