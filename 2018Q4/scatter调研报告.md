# Scatter调研报告

## 什么是Scatter

[Scatter](https://get-scatter.com/)是一套协议和实现，该协议是为了帮助用户在dApp上与链交互，同时能有效保护用户的私钥安全。另一方面，也能方便dApp的开发。目前支持Scatter的dApp已有多达几十种。

Scatter具有以下特点：

对于普通用户：

1. 方便用户签名交易，而且是支持不同链上的交易；
2. 用户完全掌控自己的数据。这些数据都是存在用户本地设备里的，避免泄露的风险；
3. 提供了单点登录(Singgle Sign-on)的方式，可以免密登录各种应用；

对于开发者：

1. 易于集成，支持跨平台，接口用法跟知名的库类似，如`eosjs`,`web3`；
2. 一次编码，各处使用。Scatter的sdk能够支持多个平台，开发者的代码可以运行在移动端和桌面端；
3. 多种链的支持。目前已知的有EOS，ETH，Tron；

### 两种集成方式

对于**web应用**，不管是桌面浏览器中打开的，还是移动应用中打开的web应用，都可以通过集成`scatter-js`库的方式跟用户的Scatter交互。这里的Scatter可以是浏览器上的插件（这种也被称为Scatter Classic，目前只有Chrome插件支持），也可以是某个实现了Scatter的协议钱包方，如Meet.one.

对于`桌面应用和原生应用`，通过开启本地websocket服务，与Scatter交互。

### 通信方式

Scatter与dApp通信的方式是通过websocket。用户的请求，通过scatter sdk包装后，通过websocket·发送出去，被Scatter接收后，再返回处理结果。Scatter在api封装上尽量做到了统一，所以不同链上应用的交互方式都是很接近的。

## Scatter-js

`Scatter-js` 是用于dApp和Scatter交互的轻量级js库。现在它包含5个部分：基础库`scatterjs-core`和三个链的插件库（scatterjs-plugin-eosjs， scatterjs-plugin-eosjs2， scatterjs-plugin-web3，scatterjs-plugin-tron）。

`Scatter-js`通过websocket的方式发送请求给`Scatter`。具体代码实现在[这里](https://github.com/GetScatter/scatter-js/blob/master/packages/core/src/services/SocketService.js)。

#### 基础用法

Scatter宣称可以使用一套代码，同时兼容多种平台。

1. 建立与Scatter的连接

```
// Optional!
const connectionOptions = {initTimeout:10000}

ScatterJS.scatter.connect("Put_Your_App_Name_Here", connectionOptions).then(connected => {
    if(!connected) {
        // User does not have Scatter installed/unlocked.
        return false;
    }
    
    // Use `scatter` normally now.
    ScatterJS.scatter.getIdentity(...);
});
```

2. 使用signature provider plugin

   ```
   // Using a proxy wrapper
   const eos = ScatterJS.scatter.eos(network, Eos, eosjsOptions);
   
   // Or using a hook provider.
   const eos = Eos({httpEndpoint:'', signatureProvider:ScatterJS.scatter.eosHook(network)});
   
   ```

3. 调用合约（签名并发送交易）

   ```
   // ---------------------------------------
   // EOS
   // ---------------------------------------
   const contract = await eos.contract('hello', {requiredFields});
   const result = await contract.hi(...);
   
   // ---------------------------------------
   // Ethereum
   // ---------------------------------------
   
   web3.eth.sendTransaction({
       from: account.address,
       to: '0x11f4d......',
       value: '1',
       requiredFields,
       fieldsCallback
   })
   
   const options = {from:account.address, abi, requiredFields, fields => {
       console.log('fields', fields);
   }};
   contract.methods.hello(...args).send(options)
   ```


## ScatterWebExtension

Scatter官方实现的[Chrome](https://github.com/GetScatter/ScatterWebExtension)插件。能够管理用户的身份和私钥，用于对dapp授权登录和签名交易等。

## Scatter Desktop

Scatter官方实现的[桌面应用](https://github.com/GetScatter/ScatterDesktop/releases)。它在本地启动了websocket服务，用于接收scatter的请求。具体实现在[这里](https://github.com/GetScatter/ScatterDesktop/blob/master/src/services/SocketService.js)

## RIDL

Scatter 推出的去中心化声誉平台。

## 移动端集成方案

虽然Scatter声称使用它们的SDK，可以用一套代码兼容多个平台，接口也是一样的。但在调研其相关钱包方后发现，事情没有那么简单。我们在调研了meet.one,Tron的钱包后发现，它们作为Provider，与其上dApp通信的方式是通过postMessage。

### Meet.one的方案

钱包方Meet.one提供了`DApp JS-Bridge SDK`, 但代码没有开源。它的[README.md](https://github.com/meet-one/JS-SDK)中详细说明了dApp的集成方式。它的js sdk将dapp的请求包装成字符串形式的uri，然后可以将该uri发送给钱包meet.one，但是发送方式可以有多种，它列了出了两种：

* ```
  window.location.href = uri;
  window.postMessage(uri);
  ```

> meet.one声称它的接口协议支持Scatter的协议。

在发出uri后，钱包方需要拦截消息并处理请求，然后处理的结果如何返回给dApp方呢？它的文档中给出的方法是在dapp里添加`message`事件监听，然后在钱包方需要返回数据时，触发`message`事件，将结果传回给dapp。

### Tron wallet的方案

[Tron wallet](https://github.com/TronWatch/Mobile-Wallet) 是Tron的社区开发组织TronWatch开发的移动端钱包，基于`react-native`开发，通过观察其源码发现，它与dapp通信的方式也是通过postMessage。不过，它通过向webview里注入js方法，让dapp可以直接通过调用`callTronWallet`方法，发送消息给钱包。

### TronLink

[TronLink](https://github.com/TronWatch/TronLink) 是Tron的社区开发组织TronWatch开发的浏览器插件钱包，类似Ontology的chrome插件钱包Cyano Wallet. TronLink内部通过自己实现的两个工具包，可以支持跟dapp间的双向通信。其双向通信的实现是基于事件监听和插件的通信机制，简单来讲，每个消息带有独特的messageID， 在收到消息时，返回的是Promise包装的结果，wallet在处理完消息，会发生reply的消息，对应的messageID的resolve会被调用。

## Cyano dapi移动端扩展

Cyano Wallet是本体开发者社区开发的浏览器插件钱包，也能够实现dapp和provider(钱包)的双向通行，其实现也是基于postMessage机制。由于dapp开发和插件钱包开发都是用js开发，但在移动端，开发语言和环境不同，导致cyano wallet的方式不能直接适用。我们重新定义了不同场景下dapp和provider交互的协议，有以下三种：

* 扫描二维码
* provider中打开dapp
* dapp唤醒provider

[详情见](https://github.com/ontio-cyano/CEPs/blob/master/CEP1.mediawiki)

针对常见的第二种场景，我们提供了用于dapp端的库`cyanobridge`和provider的不同语言的sdk。

在dapp端可以使用跟调用Cyano Wallet基本类似的接口。调用方式也是基于postMessage的双向通信。[具体实现详见](https://github.com/ontio-cyano/cyano-dapi-mobile)



