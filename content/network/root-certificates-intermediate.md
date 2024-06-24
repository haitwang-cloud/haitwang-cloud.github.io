+++
title = '根证书与中间证书的区别'
date = 2024-06-23T09:42:42+08:00
draft = false
+++

> 本文是 [The Difference in Root Certificates vs Intermediate Certificates](https://www.keyfactor.com/education-center/the-difference-in-root-certificates-vs-intermediate-certificates/)的中文翻译版本，内容有删减


如果你正在为你的网站申请SSL证书，你可能会遇到两个术语——根证书和中间证书，这两个术语其实很容易被混淆。


首先，根证书和中间证书之间的基本区别就在于根。一个**根证书颁发机构**在主流浏览器的信任存储中有自己的受信任根证书。另一方面，中间证书颁发机构或子证书颁发机构会颁发一个中间根证书，因为它们在浏览器的信任存储中没有根证书。因此，中间证书颁发机构的根指向了信任方的根。

![](/pics/Root-CA-01.png)

# SSL 是如何工作的？

## 公钥基础设施(Public Key Infrastructure,PKI) 
 
[SSL](https://www.keyfactor.com/blog/what-is-ssl/) 基本上采用了私钥-公钥对的实现。网站数据根据私钥进行加密，只有拥有正确[公钥](https://www.keyfactor.com/resources/what-is-pki/)的客户端才能访问该网站。公钥以证书文件的形式分发给客户端，每当用户尝试访问网站时都会使用该证书文件。

如果网站的证书无效，意味着该网站尚未经过适当验证，存在潜在威胁。网站可以选择各种不同的认证级别，每个级别具有不同的信任级别和验证过程。

证书(Certificates)由合适的认证机构签发，这些机构会对申请SSL证书的网站进行调查，以确定其是否真实。每个证书都有一个到期日期，过期后将被视为无效，必须进行更新以继续使用SSL保护功能。

正如你所看到的，SSL是一种机制，允许用户知道他们正在连接的网站是否经过认证机构的适当验证，并确实是他们想要访问的真实网站。没有SSL，就不会有加密，你也会面临数据泄露和安全漏洞的风险。

## 数字签名 （Digital Signatures）

[数字签名](https://www.keyfactor.com/blog/digital-signature-vs-digital-certificate/)可以被视为验证特定证书文件的数字信息。这类似于我们如何通过合格的公证员公证实体文件。在 SSL 证书链中，每个组件（无论是证书还是公钥）都将由适当的证书签名。。

浏览器在验证 SSL 证书时，会首先检查证书是否由受信任的 CA 颁发。如果证书是由受信任的 CA 颁发的，浏览器会检查证书的数字签名是否有效。如果数字签名有效，浏览器会检查证书链中的所有证书是否有效。如果证书链的所有证书都有效，浏览器会认为该 SSL 证书是有效的。

## 证书链 (Certificate Chains)

公钥或证书颁发过程包括许多与安全相关的服务，以确保公钥/私钥对的正确使用。在此过程中会使用一系列可能无法被最终用户访问的证书。这个证书链也被称为**信任链**或**认证路径**。

每个证书链都包含一个证书列表，其中包括最终的用户证书和一个或多个 CA（证书颁发机构）证书，以及一个自签名证书。每个证书具有以下属性：

    
- 证书颁发者的详细信息。除了证书链中的最后一个证书外，这个值将与链中下一个证书的主体相匹配。
- 用于对链中下一个证书进行签名的秘钥。再次强调，链中的最后一个证书不需要进行此秘钥签名过程。
- 正如前面所述，证书链的最后一个证书与其他证书不同。它被称为信任锚定点，始终由可信实体提供，如有效的认证机构（CA）。通常被称为**CA证书**。只有经过验证且高度受信任的认证机构才能颁发此类根证书，因为它们是证书链中的信任锚定点
  
如果证书链无法追溯到有效的根证书，则该证书链将被视为无效，并且最终用户应用程序不会将相应的网站视为来自受信任的来源。

## 证书层级(Certificate Hierarchy)

证书链 或 信任链 定义了您的实际 SSL 证书与受信任 CA 之间的链接。任何 SSL 证书要想被视为有效，都必须追溯到有效的 CA。

当您存储您尝试连接的新网站的证书时，您可以查看证书以获取更多详细信息并获得证书层次结构。您拥有的第一个证书将是根证书，其次是中间 CA，然后最终证书应指向有效的 CA。

### 根证书（Root Certificate）

这是由CA颁发的数字证书文件，用于所有使用SSL保护的网站。你的**浏览器**会下载这个文件并将其存储在信任存储中。所有根证书都受到颁发它们的CA的严密保护。

### 中间证书（Intermediate Certificate）

这些证书在证书层级中位于根证书之后。它们就像根证书的分支，充当根证书和向公众颁发的公共**服务器证书**之间的中间者。大多数**证书链**通常只有一个中间证书，但也有可能会有多个中间证书。

### 服务器证书(Server Certificate)
最终用户证书 是 CA 向需要实施 SSL 协议的域名颁发的证书。

由于证书链用于验证最终用户证书是否来自受信任的 CA，因此链中的每个证书都包含一个数字签名，该签名对应于链中下一个证书，并最终指向信任锚证书。到达信任锚证书意味着可以信任最终用户证书，因为它可以追溯到适当的 CA 颁发的信任锚。

## 根程序(Root Program)

正如你所看到的，根证书是信任链中最重要的部分，因为它用于验证最终用户证书。一个**根证书计划**有助于管理根证书及其公钥，存储于设备上的名为根存储区的特定位置。

这个根存储区的位置和实现方式可能因设备操作系统或其他第三方应用程序而异。一些知名的根证书计划提供方包括：

- Microsoft（微软）
- Apple（苹果）
- Google（谷歌）
- Mozilla（火狐）
- 
尽管这些根证书计划的实际实施可能存在差异，但它们都遵循CA/B论坛基线要求所规定的严格指南和规定。一些根证书计划可能还会对验证证书文件施加额外的限制。

# 为什么根证书很重要？

![](/pics/Root-CA-02.png)

根证书是SSL协议中最关键的部分，因为任何使用其**私钥**信息签名的证书都会被所有浏览器信任。因此，会特别谨慎确保一个有效的CA确实颁发了根证书。

根证书建立了信任因素，为网站增添了可信度。一个有效的CA必须经历多项验证和合规程序，以被视为足够值得信赖，从而颁发根证书。因此，通过根证书建立了CA的信任锚定点，直接关联到使用CA提供的签名安全证书的网站。

# 为什么使用中间证书？
![](/pics/Root-CA-03.png)
虽然根证书本身足以实现SSL安全，但实际上大多数CA会使用中间证书。这是因为获得颁发CA所需的基本资格涉及实践中的实际考量。

在大多数情况下，CA开始颁发交叉证书。这些数字证书由一个CA颁发，与另一个知名CA颁发的根证书的公钥相连接。

刚开始运营的初级认证机构可能尚未具备颁发根证书所需的必要资格。因此，它将使用另一个知名CA的服务，并将其证书与有效的根证书相连接，从而形成信任链。一个受信任的根证书将通过交叉证书与多个其他中间证书相连接，从而允许用户为其SSL实施获取有效的信任链。

一旦CA获得必要的验证并被认为有资格颁发自己的根证书，它将用自己的根证书替换信任锚定点。相应的根证书将被添加到根存储区。

因此，中间证书有助于弥合中间CA和受信任根证书之间的差距。它们用于帮助不断壮大的CA公司站稳脚跟并建立客户群。经过适当验证后，它们将颁发自己的根证书，完成信任链，无需其他CA的帮助。

以下是使用中间证书的一些更多原因：

- 中间证书有助于控制正在使用的根证书数量，并有助于减轻安全风险和欺诈。随着越来越多的用户实施SSL网站，根CA的数量也会增加。但拥有太多的根CA可能会导致严重的安全隐患，中间证书旨在解决这个问题。它们为根CA提供了一种方式，可以将部分证书颁发责任委派给中间CA，提供可替代根证书的中间证书。

- 中间证书可以大量复制，而不会影响安全框架并有助于建立信任链。

- 它们有助于实现可扩展的SSL网络。 

  

几乎所有**受信任的CA**都使用中间证书，因为它增加了额外的安全层并有助于优雅地管理安全事件。*在安全攻击发生时，只需吊销中间证书，而无需吊销根证书及其所有相关证书。只需吊销涉及的中间证书，就会影响与该中间证书在同一链中的一组证书，从而最小化安全事件的成本和影响*。

仅吊销涉及的中间证书，只会影响与该中间证书在同一链中的一组证书，从而最小化安全事件的成本和影响。

# 什么使根证书特殊？

信任锚定点或根证书是X.509格式的一种特殊类型SSL证书。它可以单独存在，也可以用于颁发其他中间证书。以下是使根CA特殊的一些特征。

*   根CA的生命周期比常规TLS/SSL证书长得多。它可以高达25年，而常规证书通常限制在1到2年的寿命。

*   每个受信任的CA可能有多个根证书，每个根证书的属性可能不同，如使用的数字签名。可以从根存储应用程序中查看证书属性。

*   通过根证书对公钥进行签名，并验证其他证书。如果中间证书链无法追溯到根证书，将被视为无效。

  
被授权颁发根证书的CA必须遵循严格的合规和规则，以达到让他们能够颁发根证书的地位。根CA必须在两个特定背景下符合资格，才能获得颁发根证书的授权。

### 社会信任(Social trust)

- 社会信任指的是CA必须接受的各种审计、规则、法规以及公众审查，以便被视为可信赖。它也可以适用于CA在市场上的品牌和形象认知。

### 技术信任（Technical trust）

*   技术信任指的是根CA在其证书实施和安全措施方面的技术强度和安全性。只有技术上强大的CA才能获得所需的社会信任，以长期发挥作用。

# 根证书和中间证书之间的区别


判断证书是根证书还是中间证书非常简单，可以直接查看根存储中的详细信息来推断。以下是如何查看证书详细信息的方法：

1. 点击根存储中的证书，打开以查看详细信息。
2. 转到证书验证路径。您将能够看到证书路径中的各个级别。您还可以收集更多详细信息，如颁发证书的CA、颁发证书的CA以及证书的生命周期。

以下是显示证书为根证书的标识：

- 证书路径只包含一个级别。
- 颁发给和颁发自的值指向同一CA。
- 证书的有效期超过两年。根证书的有效期通常长达25年，而中间CA只有一到两年的有效期。

Windows根存储应用程序使区分这些证书变得更容易，它将它们列在不同的类别中。您可以在"受信任的根证书颁发机构"中找到根证书，而中间证书则在"中间证书颁发机构"下。

# 总结


虽然链式根证书实现是最常用的，但它也存在一定的实施复杂性。

链式根证书的安装可能是一个复杂的过程，因为中间证书必须加载到使用链中证书的每台服务器和应用程序上。
链式根证书严重依赖它们链接到的信任锚定点。中间证书无法对根证书进行控制。如果根CA倒闭，依赖它的中间CA也将不得不关闭。
中间证书实施中的另一个复杂性是证书的生命周期。通常，根证书的寿命比较长，为25年。而中间证书必须始终将其过期日期设置为比其信任锚定点的过期日期更短。这可能增加安装的复杂性，因为如果根证书接近其到期日期，所有中间证书必须在该日期之前过期。

