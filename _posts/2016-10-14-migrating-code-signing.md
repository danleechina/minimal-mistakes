---
layout: single
author_profile: true
title: 将代码签名配置迁移到 Xcode 8
---

>原文地址 [Migrating Code Signing Configurations to Xcode 8](https://pewpewthespells.com/blog/migrating_code_signing.html)

这是 iOS 开发和部署的代码签名软件的指南。这里包含的信息可能有助于更好地理解代码签名的过程如何工作和实现，尤其是为那些构建必须由多个苹果开发者账号证书签署的应用程序和软件（例如：开发和企业）以及如何迁移您的现有的签名配置到 Xcode 8中工作。

# <a name="table-content"></a>目录

- [代码签名简介](#intro-code-sign)
	- [获取签名证书](#cert-sign-req)
	- [签名证书](#sign-cert)
	- [签名二进制文件](#sign-binary)
	- [Provisioning Profiles](#prov-prof)
	- [签名验证](#sign-vali)
	- [部署类型](#type-depl)
	- [理解“Fix issue”](#unde-fix-issu)
- [Xcode 7及以前的签名机制](#sign-in-xcode7-prior)
	- [签名方法](#sign-method)
		- [自动签名](#auto-sign)
		- [手动签名](#manu-sign)
	- [构建开发版本](#build-for-develop)
	- [构建分发版本](#build-for-distribute)
- [Xcode 8的签名机制](#sign-in-xcode8)
	- [签名方法](#sign-method-xcode8)
		- [自动签名](#auto-sign-xcode8)
		- [手动签名](#manu-sign-xcode8)
	- [构建开发版本](#build-for-develop-xcode8)
	- [构建分发版本](#build-for-distribute-xcode8)
- [兼容Xcode 8 和 Xcode 7](#work-in-both-worlds)


# <a name="intro-code-sign"></a> 代码签名简介

要理解代码签名在整个生态系统中所起的作用，您必须首先了解它的工作原理。本节介绍如何创建签名证书，以及代码签名在 iOS 生态系统中扮演的角色。为此，我们将介绍签署应用程序过程中的每个步骤。

给东西签名意味着能通过由已知的可信管理机构的验证。这对于苹果创建的 iOS 平台生态系统非常重要。在 iOS 设备上运行的所有软件必须由 Apple 信任的源进行签名。为了实现这一点，苹果公司通过苹果开发人员计划来批准相应的 iOS 开发人员获得授权在 iOS 设备上安装和运行代码。一旦您是此计划的成员之一，您将需要请求签名证书，以便您可以对您的 iOS 软件进行签名。

## <a name="cert-sign-req"></a> 获取签名证书

要获取签名证书，您首先需要创建 CSR 文件（Certificate Signing Request）。要创建CSR，您必须首先生成用于签署请求的私钥。为此，您可以运行以下命令：

`$ openssl genrsa -out MyPrivateSigningKey.key 2048`

这条命令将生成一个新的私钥并将其写入 `-out` 参数指定的路径的磁盘。参数 `2048` 告诉 `openssl` 使用2048位长的模数来生成RSA密钥对。这是在OS X上创建签名证书的标准方式。

一旦私钥被创建，它将被用于签署 CSR 文件。要生成新的 CSR 文件，您需要使用以下命令：

	$ openssl req -new \
	        -key MyPrivateSigningKey.key \
	        -out MyCertificateSigningRequest.csr \
	        -subj "/emailAddress=[Your email address]/commonName=[Your Name]/countryName=[Your country code]"

需要在请求中指定三个字段：

- 电子邮件地址
- commonName
- 国家的名字

这些字段的内容将用于向签名请求中添加元数据，以便您请求的证书可标识为您的。所以，如果我要创建一个新的CSR，命令将执行将如下所示：

	$ openssl req -new \
	        -key MyPrivateSigningKey.key \
	        -out MyCSR.csr \
	        -subj "/emailAddress=hello@pewpewthespells.com/commonName=Samantha Marshall/countryName=US"
        
这将创建一个 CSR，其中包含公钥和 `-subj` 指定的信息，然后数据将通过我的私钥进行签名，以验证它最初来自我。在创建证书签名请求后，就可以转到 Apple Developer Portal 并提交它以请求 Development 签名证书。

## <a name="sign-cert"></a>签名证书

证书颁发机构一旦获取到提交的 CSR 文件，它将使用 CSR 文件中的信息创建证书。生成的证书由证书颁发机构（Apple）签发并颁发给开发人员。开发人员使用该证书来签署他们想要安装到 iOS 设备上的软件，而不必让 iOS 设备明确地知道每个单独的开发人员。这是因为 iOS 设备知道（并信任）开发人员的证书的颁发机构（Apple），因此， iOS 设备知道它可以信任由Apple签名的证书。

总结：

	[1. 私钥] -----> [2. CSR] -----> [3. Apple Developer Certificate Authority]
	                                                            |         ^
	                                                            |         |
	                                                            |         |
	                                                            V         |
	[5. Signed Application] <--- [4. Developer Identity Certificate]      |										                                        |
		 |                                                                |
	     |                                                                |
	     |                                                                |
	     V                                                                |
	[6. iOS Device] ------------------------------------------------------
	
1. 开发人员的私钥用于签署 CSR 文件
2. CSR 文件包含开发者的信息和公钥
3. 苹果证书管理机构接收 CSR 文件，并为开发人员生成并签署身份证书
4. 开发人员接收身份证书，使用此证书及其私钥对应用程序进行签名
5. 由身份证书签名的软件被签名者信任。
6. 安装在iOS设备上的软件的证书针对签署开发人员证书的 CA 证书进行验证。验证的信任链：
	- 软件受开发者信任
	- 开发人员被苹果信任
	- 设备信任苹果

因此，设备信任来自开发者的软件。

## <a name="sign-binary"></a>签名二进制文件

构建 iOS 软件的一部分要求是，您必须签署所有要部署到iOS设备的软件。这是由于操作系统强制执行的严格的安全策略。 通过在可执行二进制文件上调用 `codesign`，Xcode 帮助我们将这一步整合到了构建应用程序的过程中。这将使用由 Apple 创建的身份证书相关联的私钥，生成二进制文件中代码内容的签名。然后将所生成的签名嵌入到可执行二进制文件中，以允许其被验证。这将确保应用程序中的代码不会被修改，除非嵌入式签名的验证失败。由于 iOS 应用程序不仅仅又可执行二进制文件组成，因此这种单一嵌入式签名不足以确保整个应用程序的完整性。

在 OS X 上，应用程序被视为具有 `.app` 扩展名的文件。这被称为“包（bundle）”。包是包含结构化格式和内容布局的目录。通过给目录文件添加扩展名，允许它们被系统识别以被另一个程序加载。当包有 `.app` 扩展名的时候，它会被系统程序 LaunchServices 加载。 LaunchServices 负责运行由其他软件或者用户点击启动的软件。因此，通常当您启动 Xcode 时，您可以在 Finder 中双击它，或者在 Dock 中点击它，或者要求 Spotlight 启动它。所有这些都是基于具有 `.app` 扩展名的包。在 OS X 上，应用程序包遵循以下结构：

	+ -o Foo.app 					|用户可见应用程序
	  + -o Contents 				|包含包的内容的目录
	    + -o Info.plist 			|文件，其中包含有关软件包的信息
	    + -o PkgInfo 				|创建包时生成的元数据文件
	    + -o MacOS 					|包含可执行二进制文件的目录
	    | + -o Foo 					|可执行二进制
	    + -o Resources 				|可用于应用程序的资源目录
	    | + -o Foo.icns 			|图标，显示为用户可见的应用程序
	    + -o Framework				|任何可执行文件使用的框架或动态库
	      + -o Sparkle.framework 	| OS X 软件中的常见框架示例

这被称作“深层应用程序包（deep application bundle）”，因为一切都存储在顶层 Foo.app 包目录中。

在 iOS 上，应用程序遵循稍微不同的结构，被称作“浅应用程序包（shallow application bundles）”

	+ -o Foo.app 							|用户可见应用程序
	  + -o Info.plist						|文件，其中包含有关软件包的信息
	  + -o Foo 								|可执行二进制
	  + -o Foo.icns 						|图标，显示为用户可见应用程序
	  + -o Framework						|任何可执行文件使用的框架或动态库
	    + -o AFNetworking.framework 		| iOS 软件中常见框架示例

正如你所看到的，不像 OS X 上的应用程序包那样有单独的目录，在 iOS 上使用更浅的方法来存储应用程序使用的必要资源和数据。

为了确保可执行二进制文件以及运行时使用的数据和资源未被修改， 一个新目录将被添加到应用程序包中。这个新目录名为 `_CodeSignature`，并且包含一个 `CodeResources` 文件。此文件是 plist 格式的，罗列出包中所有的文件，并为每个文件提供哈希值。这些哈希值用于验证包的内容保持不变。同时资源检查使得某些资源可以进行更新和删除，以防止应用程序使其自己的签名无效。

此过程允许开发人员立即执行构建并签名，并确保他们计划分发的二进制文件与最初构建的二进制文件相同。

## <a name="prov-prof"></a>Provisioning Profiles

除了使用证书验证应用的签名者的真实性，还有一种附加文件用于验证应用被允许安装在特定设备。这种文件被称为配置概要文件。Provisioning Profiles 是由 Apple 的 CA 加密签名的 plist 文件，以确保其在创建后不能被修改。这使得苹果完全控制在 iOS 上使用的部署机制。

Provisioning Profiles 包含用于启用应用程序的一些特定信息：

- 可用于对应用程序进行签名的证书
- 绑定 bundle identifier 与应用程序
- 部署方法（企业风格或基于设备UDID）
- 团队标识符
- 沙箱限制
- 可安装的过期日期

所有这些属性用于确定设备是否允许安装包含该 Provisioning Profiles 的应用程序。这限制了谁可以安装使用开发这证书签名的应用程序，并且任何被签名的应用程序的帐户能够被证明是从 Apple 开发人员门户创建的。

Provisioning Profiles 的目的是确保该种配置的安装是安全的，同时也不允许在任何时候更新该配置。在 Apple 开发人员门户中，您可以随时编辑和重新生成描述文件。这样做不会使以前的描述文件无效，它只会根据您修改的内容生成具有不同内容的新的描述文件。

## <a name="sign-vali"></a>签名验证

如前所述，iOS 操作系统限制所有可执行代码都必须使用被 Apple 信任的证书颁发机构签名。这种限制在应用还没有安装之前就已经开始了。

当开发 iOS 软件时，您将在您的计算机上构建和签名应用程序，然后 Xcode（或其他途径，如果您使用不同的IDE）将与通过 USB 连接的 iOS 设备通信。它将与 iOS 设备执行身份验证握手，然后通过 USB 接口使用本地安全套接字开始与之通信。在这时，iOS设备被告知我们要安装软件包，并开始将数据发送到设备。 iOS设备将接收此数据并在临时沙箱环境中重建应用程序。一旦应用程序已完全接收，操作系统将验证应用程序是否已签名并且能够安装到此设备上。

一旦确定应用程序来自可信任源，就会针对 Provisioning Profiles 文件进行验证，以确保应用程序可以在此设备上安装和运行。每个 iOS 设备都有唯一的标识符 UDID（Unique Device IDentifier）。该标识符由以下字符串的 SHA1 组成：

- 序列号
- ECID（也称为UniqueChipID，这是每个设备独有的）
- wifi 卡的 MAC 地址
- 蓝牙卡的 MAC 地址

此标识符包含足以唯一标识所有 iOS 设备的信息。当在苹果开发者门户网站上注册特定设备时，开发人员需要使用该标识符。每个开发者帐户只能注册的特定数量的设备。Provisioning Profiles 文件可以包含任何数量的标识符。这限制开发人员只能安装注册到其设备池的设备。如果设备的 UDID 与在描述文件中注册的任何标识符不匹配，则该应用程序将不会安装，因为它未获批准。如果应用程序的签名和描述文件都通过验证步骤，则会为该应用程序创建一个真正的沙箱容器，以驻留在其中。iOS 系统会为将要安装的应用程序创建一个新目录。从那里，它将能够访问有限的文件系统和资源。

此外，任何时候在 iOS 上启动应用程序，系统会执行额外的验证和代码签名验证步骤，以确保从安装时开始，可执行代码没有被修改。负责这个的过程称为 `amfid`，“Apple Mobile File Integrity Daemon” 的缩写。这是一种在后台运行的服务，可确保系统上运行的所有内容都已经被允许在系统上运行。最近有一个关于这个过程的性能问题，因为启动应用需要加载许多动态库。在iOS上运行任何代码的要求之一是所有可执行代码必须先验证为可信，然后才能通过动态链接器加载到内存中。这个过程会导致系统上的瓶颈，具体可以看这个 [issue](https://github.com/artsy/eigen/issues/586)。所以苹果建议开发人员限制应用程序使用的 dylib（动态链接库） 数量。

## <a name="type-depl"></a>部署类型

有两种方法将应用程序部署到iOS设备，主要是基于签名配置。

- 开发（Development） - 将应用程序部署到您的设备
- 分发（Distribution）- 将应用程序部署到其他人的设备

这两个配置很明显指的是它们依赖于用于执行应用程序签名的签名身份。

作为开发人员，要将应用程序部署到您的某个 iOS 设备，您必须具有与由Apple的开发人员证书颁发机构签名的身份证书配对的私钥。由该 CA 签名的身份证书将有权在有限数量的特定设备上安装，然后将调试器连接到该应用以使用它进行开发。这是系统上相对较高级别的权限，因此，Apple对单个开发人员帐户注册的设备数量有严格的限制。

第二种部署是为了软件分发的目的。一旦应用程序已经完成开发并准备好使用，它需要为分发进行签名。这种签名比开发部署时的签名更加严格，某种程度上与从 App Store 或企业渠道获取的应用程序一致。提交到 App Store 的应用程序必须先使用分发身份证书进行签名。这需要额外的 CSR 并提交给苹果。然而，这次它由不同的 CA 签署，Apple Distribution Certificate Authority （苹果分销证书颁发机构）。此外，为内部使用构建的应用程序会使用 Enterprise Distribution （企业分发）身份证书签名。

## <a name="unde-fix-issu"></a>理解“Fix issue”

为了帮助开发人员解决代码签名配置的问题， Xcode 引入了一个新的对话框。这是为了帮助开发人员避免繁琐的构建应用的配置过程。以下是简短的完整步骤：

	[1]<-----------\
	 |\             |
	 | \--NO------>[0]
	YES             ▲
	 |              |
	 V              |
	[2]             |
	 |\             ▲
	 | \--NO--------|
	YES             |
	 |              |
	 V              |
	[3]             |
	 |\             ▲
	 | \--NO--------|
	YES             |
	 |              |
	 V              |
	[4]             ▲
	 |\             |
	 | \--NO-------/
	YES
	 |
	 \------> Successful Build :)

1. 确定什么身份证书可用
2. 检查身份证书是否具有相应的私钥
3. 确定哪些描述文件可用
4. 检查是否有可用于在目标设备上安装的描述文件，其中包含找到的签名标识和应用程序的软件包标识符

如果任何这些的答案是否，那么你将最终转到步骤0的流程图，代表苹果开发者门户。然后会出现“Fix issue”对话框，点击确定之后会创建一个新的 CSR 文件，并如果没有证书的话会请求一个新的证书，并自动创建一个新的描述文件，将能够用于部署该应用程序。这个过程非常有用，它的缺点也很复杂。说实话，这种方法是适合不怎么理解代码签名机制并且不知道怎么解决的开发人员。不过，Xcode 的这个特征在自动代码签名配置的设置中起着非常重要的作用。

> 注意：当频繁更改描述文件时，最好使用开发描述文件进行所有调试工作，因为 Xcode 会在必要时重新生成这些描述文件。一旦在开发环境中完成配置，您应该手动更新和重新生成分发描述文件以适应任何新的更改。

[↑目录](#table-content)

***

>现在我们将讨论如何基于构建配置来配置 Xcode 来签名应用程序，以及以前和以后的工作方式。对于这一部分，我将使用 xcodebuild 和 xcconfig 文件作为我大多数的例子，因为这也是我正在使用的方式。主要针对为企业和 App Store 分发的应用程序，以及如何修改您现有的方式以便在Xcode 8中工作。这还包含一些操作技巧，比如如何配置签名用于多个 Xcode 版本的构建。
>
>我将首先介绍关于处理 iOS 构建软件和签名身份管理的一些基本概要：
>
> 1. 任何开发人员都不应该访问签名身份的私钥和用于分发的签名证书。
> 2. 在 CI 服务器上构建用于分发的应用程序（App Store 或 Enterprise），并提供给需要的用户。
> 3. 应在 xcconfig 文件中指定构建设置。这是为了防止对 Xcode 的过度编辑。
> 4. CI 服务器可以访问：
	- Provisioning Profiles（描述文件） 我们自己托管我们自己的 Jenkins 实例，这使得管理构建的描述文件非常容易，因为我们可以直接访问文件系统以添加任何新的描述文件。如果您使用的是其他系统，则必须在 repo 中包含描述文件，或者使用其他方法确保它们可用于执行构建。
	- 开发人员帐户 用于 Xcode 执行 Archive（存档）和导出 IPA 包操作，它可能需要访问开发帐户，以便它可以正确地签署以便上传。
>
>这些点是本指南的剩余部分将要写到的;以统一方式执行构建，同时限制对开发帐户的签名凭证的访问。

***

# <a name="sign-in-xcode7-prior"></a> Xcode 7及以前的签名机制
在Xcode 7及以前版本中，已有签署应用程序的方法，就是通过为 CODE_SIGN_IDENTITY 和 PROVISIONING_PROFILE 指定值来执行构建。这两个值是用于确定应用程序中创建嵌入式签名的签名配置。

## <a name="sign-method"></a>签名方法

`CODE_SIGN_IDENTITY` 和 `PROVISIONING_PROFILE` 需要在构建时指定值，以便能够构建一个 iOS 应用程序。有两种方法可以采取：自动签名和手动签名配置。两者遵循相同的核心逻辑路径，但在实践中可以被用于不同的目的。 Xcode 中代码签名背后的行为遵循了这套规则：

1. 确定已编译应用程序的软件包标识符
2. 确定部署应用程序的设备的 UDID
3. 根据 `PROVISIONING_PROFILE` 的值确定应使用的配置概要文件
4. 根据描述文件中支持的证书确定应使用的签名身份，以及 CODE_SIGN_IDENTITY 的值

步骤1和2不会失败，因为它们是构建和部署应用程序到 iOS 设备的核心。步骤3和4可能会失败，失败的时候会出现“Fix issue”对话框。如果开发人员具有签名身份，但缺少支持包中的标识符和目标 iOS 设备的描述文件，则步骤3将失败。 Xcode 将为你创建一个新的描述文件，支持现有的签名身份和目标设备并下载使用。如果开发人员没有有效的签名身份，则步骤4将失败。此时，Xcode 将执行所有需要的步骤为开发人员创建新的签名身份，并与新的描述文件一起下载。这是为了节省开发人员在确定两个潜在问题中哪一个是真正的根本原因的麻烦。

### <a name="auto-sign"></a>自动签名

在Xcode 7和以前，有一种“automatic signing”的方法，利用“Fix issue”对话框的方式，来解决将应用程序部署到设备用于开发目的的问题。要在您自己的签名配置中使用此行为，应指定以下值：

	CODE_SIGN_IDENTITY = iPhone Developer
	PROVISIONING_PROFILE =

要清楚，我们在这里做的设置是说，我们希望 Xcode 使用包含 iPhone Developer 的签名身份来签署应用程序，并且不使用特定的描述文件。这意味着只要团队中的每个开发人员都具有有效的开发签名身份（身份证书和私钥），他们就可以安全地部署到任何设备，而不必担心破坏签名配置。如上所述，每当尝试部署到某个开发人员没有配置描述文件的设备时，这将导致 Xcode 提示“Fix issue”。这还好，因为 Xcode 将下载该描述文件，您将能够部署到目标 iOS 设备。这可能会导致 Xcode 通过向构建设置中添加值来弄脏 .xcodeproj 文件。一旦 Xcode 下载了新创建的描述文件，那么就可以安全地删除该值，因为当开发人员再次执行构建时，Xcode 会自动找到该值。

对于只有应用程序开发团队的成员将使用的构建配置，这种行为是非常可取的。除了需要成为苹果开发者计划的成员之外，不需要对信息身份的要求。这是处理任何规模的团队的理想选择，因为它允许所有开发人员管理和维护自己的设备签名身份和描述文件，而不强制任何人维护包含每个人的证书和设备的描述文件。

### <a name="manu-sign"></a>手动签名

虽然自动签名是开发应用程序的理想选择，但它在处理开发或分发的特定需求时并不奏效。所以这时候必须使用手动签名配置。通常主要要求的是用于构建的特定描述文件。在这种情况下，您将必须如下配置构建设置：

	CODE_SIGN_IDENTITY = iPhone Distribution
	PROVISIONING_PROFILE = 2249294d-440a-427c-bbef-432326c6552b
	
这将告诉构建系统使用具有标识符 2249294d-440a-427c-bbef-432326c6552b 的描述文件部署此应用程序，并用 iPhone 分发身份证书签名。这迫使 Xcode 使用这些设置或在构建时返回错误，如果不能满足要求。这是执行 App Store，Enterprise 或任何类型的 Ad-hoc 分发的理想配置，以防止整个团队访问私钥或意外覆盖其自己的分发签名证书。

## <a name="build-for-develop"></a>构建开发版本

当构建用于开发的软件时，您希望在任何时间都不会出现代码编译的进程因为某些原因被终止掉。如前所述，用于此类型构建的最佳配置是“automatic”签名方法。这减少了维护的压力，并降低可能改变 Xcode 项目文件的可能性。

您应该首先为您的开发构建创建一个新的[构建配置](http://pewpewthespells.com/blog/managing_xcode.html#buildconf)。您可以重用 Xcode 为此自动创建的现有“Debug”配置。此外，您应该创建一个新的 xcconfig 文件来容纳要对此构建配置进行特定的设置。一旦你完成了这两件事，在 Xcode 项目编辑器中，你需要为应用程序的“Debug”配置指定该 xcconfig 文件。

现在，转到 xcconfig 文件，并确保它有以下项目：

	CODE_SIGN_IDENTITY = iPhone Developer
	PROVISIONING_PROFILE =
	
这将告诉 Xcode 为我们解决描述文件问题。重要的是要注意，给 `PROVISIONING_PROFILE` 的空值将在 Xcode 构建设置编辑器中显示为值 automatic。同样可以应用于 `CODE_SIGN_IDENTITY` 值，不是 `iPhone Developer`，而是值 automatic。这将告诉 Xcode 使用满足描述文件要求的任何类型的签名身份以及该描述文件匹配的 UDID 和软件包标识符。您应该注意，`CODE_SIGN_IDENTITY` 的 automatic 值与无选项明显不同。要理解这是如何工作的，构建设置给出的值用作针对所有已知结果的过滤器。因此，`CODE_SIGN_IDENTITY` 的值可以由以下命令表示：

	$ security find-identity -p codesigning -v | grep "$CODE_SIGN_IDENTITY"

因此，当 `CODE_SIGN_IDENTITY` 的值是 `iPhone Developer` 时，您将只使用与之匹配的标识，而当使用空字符串时，您将看到所有结果。

设置完成后，您将需要进入 Xcode 中的 [Scheme 编辑器](http://pewpewthespells.com/blog/managing_xcode.html#xcscheme)，并为您可以执行的各种类型的构建相关的操作选择“Debug”构建配置。当执行这些操作（Run，Test，Analyze，Profile， Archive）时，这些选项可以控制传递给构建系统的设置。这允许您创建与构建的方案直接相关的签名配置。

## <a name="build-for-distribute"></a>构建分发版本

配置为分发构建的应用程序几乎与为开发构建应用程序所需的配置步骤相同。设置新的构建配置以及能够使用该配置执行构建的 scheme 方案。这里的区别是，不是使用`PROVISIONING_PROFILE`构建设置的 automatic 值：使用特定 provisioning profile 描述文件的标识符。

这会稍微改变签名过程的行为，因为现在用于 app target 的包标识符必须与在指定的描述文件中命名的标识符相对应。因此，Xcode 构建系统将负责在磁盘上定位与描述文件中列出的签名身份之一匹配的签名身份。如果未找到证书或私钥，则会向开发人员发出错误，通知他们无法对 app 相应的 target 进行签名以将其部署到目标 iOS 设备。

需要进行这些更改，因为在构建和部署应用程序以进行分发时，应使用特定的分发描述文件和相应的签名身份。最好是在没有人可以修改的计算机上执行为分发创建的构建，例如持续集成服务器（CI）。这限制了对签名身份的访问，并允许以统一的方式执行所有构建。

按照先前为设置 scheme 所执行的相同步骤操作： 通过配置该 scheme 为每种类型的操作使用相同的构建配置。这允许该 scheme 的所有类型的构建产生以相同方式签名和配置的二进制。

[↑目录](#table-content)

***

# <a name="sign-in-xcode8"></a>Xcode 8的签名机制
随着 Xcode 8的引入，引入了新的管理 target 的签名配置的方法。这些新方法与 Xcode 7及以前的版本中已建立的行为有些冲突。如果您的签名配置遵循上一节中描述的模式，那么您可能已经知道在 Xcode 8 中某些方式已经不再适用了。

Xcode 8中，还有两个新的构建设置，用于帮助管理新的签名方法：`DEVELOPMENT_TEAM` 和 `PROVISIONING_PROFILE_SPECIFIER`。

- `DEVELOPMENT_TEAM` 该设置用于更好地控制签署二进制文件时使用的签名身份（尤其是对于属于多个团队的人员）。
- `PROVISIONING_PROFILE_SPECIFIER` 该设置用于指示应该用于给定 target 的签名方法的类型。要使用代码签名的手动方法的 target 不会使用此设置，而是将使用已弃用的 `PROVISIONING_PROFILE` 构建设置。如果设置了该值，则新的自动代码签名方式将接管。

## <a name="sign-method-xcode8"></a>签名方法

Xcode 8引入了一种称为“automatic signing”的新方法。这是替代现有的管理签名身份的半自动方法。这个更改可能会由于“manual”和“automatic”签名方法的分离发生一些变化，从而在一开始导致一些小问题。我强烈建议每个人尽快迁移到新的自动签名方法，因为它减少了签名身份和信息管理中的潜在灾难。

### <a name="auto-sign-xcode8"></a>自动签名

这是100％自动，并提示关闭 `DEVELOPMENT_TEAM`和 `CODE_SIGN_IDENTITY` 版本设置的值。这种方法可能和你以前理解为“Automatic”的签名配置有一些关键的区别。

- `DEVELOPMENT_TEAM`必须设置为有效的开发团队标识符。不能为空，这将很好地解决使用磁盘上任何团队的签名身份的问题。当您的开发团队的所有成员不属于由 Apple Developer Account 定义的同一“开发团队”时，这非常重要。这种情况经常出现在与承包商或小公司合作时，开发人员可能使用自己的开发人员帐户而不是公司帐户进行日常开发。为了避免重复编辑 Xcode 项目文件或 xcconfig 文件，公司应该将所有开发人员添加到他们的帐户，以便访问同一个开发团队。 （注意：这并不要求所有开发人员都可以访问用于签署非开发版本的私钥）。
- `CODE_SIGN_IDENTITY` 的值应该是通用的，例如“iPhone Developer”（没有名称说明符，以防止与其他团队成员冲突）。
- 对于要构建和部署到 iOS 设备的应用程序 target，不需要设置 `PROVISIONING_PROFILE` 或`PROVISIONING_PROFILE_SPECIFIER`。

启用自动签名时，会有一些元数据值写入 xcodeproj 文件，这些值指定每个 target 应该显示哪些开发团队的 team id 正在被使用。非直观地，这些值不是作为每个 target 的构建设置的一部分，而是作为在 xcodeproj 包内的 pbproj 文件中的 PBXProject 根对象上定义的TargetAttributes 字典的一部分。除了团队标识符（team id），TargetAttributes 中也设置了另一个键值对，此键值对向 Xcode（在支持它的版本中）指示 Xcode 目标编辑器的 UI 应反映给定 target 正在使用自动签名方式。这些值在必要时由 Xcode 自动更新，根据我的经验，这不会导致xcodeproj 文件中的混乱，因为开发人员没有更改这些设置。

### <a name="manu-sign-xcode8"></a>手动签名

新的手动方式管理签名配置是真正的100％手动。使用此方法时，必须指定 `CODE_SIGN_IDENTITY` 和 `PROVISIONING_PROFILE` 构建设置的值。虽然这听起来类似于Xcode 7的现有方法，但在这里有一些重要的区别。

- 现在可以通过名称而不是创建时提供的UUID指定 Provisioning profiles 描述文件。这使得更容易跟踪哪个描述文件正在被使用，并允许创建和使用具有相同名称的新描述文件，而无需更新任何现有配置或项目文件。
- 通过“Fix Issue”功能自动创建的描述文件不能在手动签名配置中使用。 （如果你希望不用告诉 Xcode 匹配 Xcode 自己用来生成新的描述文件的名称模式）。
- `PROVISIONING_PROFILE` 构建设置的“Automatic”设置不再有效。要让 Xcode 自动解决选择要用于部署的描述文件，您必须使用自动签名方法。

为了详细阐述使用手动签名的一个潜在的问题，以及为什么应该迁移到使用自动签名方法：手动配置意味着一切都是手动的，这包括描述文件创建。由于 Xcode 自动创建的描述文件在此模式下不起作用，因此有权访问 Apple Developer Account 的管理员或代理级别的人员必须登录并将设备的 UDID 添加到 Developer Portal。之后，必须创建一个新的 Provisioning profiles 描述文件，它允许在所有注册的设备上运行应用程序，并包括添加到该帐户的开发人员门户的所有团队成员的签名身份。当添加新的团队成员和新的开发/测试设备时，将需要不断地维护此描述文件和设备列表以便及时更新。

维护这样的描述文件将需要一个全职工作的人，根据您的团队的大小将迅速耗尽添加新设备的能力（添加的设备数量是有限的）。简单来说，在少数一些例外情况下，如果可以避免，没有理由继续使用手动方式进行签名，因为这将是任何一个开发人员的巨大负担。管理这样的描述文件基本上与开发工作无关，并且应该仅用于已有的遗留构建配置。

## <a name="build-for-develop-xcode8"></a>构建开发版本
> 本节将讨论在使用新的自动签名方法的情况下配置 Xcode 项目的方式。这种方式被我用于我维护的所有项目，并且是我推荐的作为开发人员应该使用的方法。
>
> 不过，如果您使用手动签名方法，那么您的构建应该已经被正确的配置，并且工作起来应该没有问题。如果您要迁移到新的自动签名方法，那么您应该阅读本博客的其余部分。

与以前配置代码签名的方法一样，设置开发构建将从构建配置和 scheme 开始。要确保开发构建已创建，您应该将 scheme 的操作（运行，测试，分析，Profile 和归档）配置为所有指向计划用于开发的同一构建配置。除此之外，`CODE_SIGN_IDENTITY` 要设置为“iPhone Developer”。这可能看起来不直观，却是应该采取的正确方法，以确保没有人会意外地修改 scheme 的预期设置。这相对于以前的为每个构建配置分配签名身份的方法有几个好处：

- 所有开发人员可以在所有将生成的模式下构建应用程序，但不能分发它。当试图调试代码中的崩溃或失败情况时，这是非常有用的。
- 更难以覆盖或打破所配置的所需签名身份。

除了此更改之外，您还需要将 `DEVELOPMENT_TEAM` 设置为您的团队拥有的标识符。如果您不知道这是什么，您可以使用您的 Apple 开发人员帐户的 Apple ID 登录开发者门户并在“Membership”部分查看。

还有，设置自动配置需要在 Xcode 目标编辑器的“General”选项卡的“Signing”部分中选中以启用。

## <a name="build-for-distribute-xcode8"></a>构建分发版本

为了在 Xcode 8中使用新的自动签名方法来创建要分发的应用程序的构建，您必须使用“Archive” scheme 操作。在 scheme 编辑器中指定构建配置，并且该构建配置的 `CODE_SIGN_IDENTITY` 值为 “iPhone Developer”， 然后“Archive”操作才将使用用于分发的签名身份和描述文件。因此，不必执行常规的“Build”操作，您需要调用“Archive”操作。这将进行一个构建并在完成之后打开 Xcode Organizer 窗口，其中列出了 Xcode 创建的所有“.xcarchive”档案。要在 CI 服务器上执行相同的过程，您必须调用两个 xcodebuild 命令：

	$ xcodebuild archive \
	             -workspace MyApp.xcworkspace \
	             -scheme MyApp-Enterprise \
	             -configuration Enterprise \
	             -derivedDataPath ./build \
	             -archivePath ./build/Products/MyApp.xcarchive
	
	$ xcodebuild -exportArchive \
	             -archivePath ./build/Products/MyApp.xcarchive \
	             -exportOptionsPlist ./export/exportOptions-Enterprise.plist \
	             -exportPath ./build/Products/IPA

这些命令将执行以下操作：

1. 创建 xcarchive 文件
	1. 设置执行的操作 “Archive”。
	2. 指定要在其中工作的 Xcode 文件，这必须是正确的 scheme。如果未指定 `-workspace` 标志，则 xcodebuild 将默认尝试使用 xcodeproj 文件来解析 scheme，即使 scheme 本身作为 xcworkspace 文件的一部分存储。这可能导致构建失败，无法构建 scheme 定义的隐式依赖关系。
	3. 指定要使用的构建配置。这是必要的，以便使用分配给它的正确配置来构建该 scheme。当从命令行调用 xcodebuild 时，如果没有指定使用哪个构建配置，Xcode 默认使用 Release 配置。
	4. 指定构建位置，对于我们的构建，我们要将它们放置在固定的构建目录中，以便 CI 服务器知道在哪里查找构建的输出。
	5. 指定用于创建 xcarchive 包的文件路径。
2. 创建 ipa 文件
	1. 告诉 xcodebuild，我们不是在构建一个 target，而是在导出一个现有的 xcarchive。
	2. 指定 xcarchive 软件包的路径。这是由 xcodebuild 的上一次调用生成的 xcarchive 文件包。
	3. 指定生成 ipa 文件时应使用的值。这些值存储在磁盘上的一个 plist 文件中， 该 plist 文件已经填充了所有的信息。下面会列出了一个示例， 解释了该 plist 的键值对以及它们的含义。
	4. 指定要将 ipa 文件导出到的目录。这将产生可用于分发的安装包。

> 注意：此过程与为 Enterprise 或 App Store 创建分发版本完全相同。 exportOptions.plist 中的“method”键将具有值“app-store”而不是“enterprise”（如果是 App Store 分发版本的话）。此外，您的 Enterprise 团队标识符或者开发团队标识符也将被用于“teamID”键。

`exportOptions.plist` 示例：

	<?xml version="1.0" encoding="UTF-8"?>
	<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
	<plist version="1.0">
	<dict>
	    <key>compileBitcode</key>
	    <false/>
	    <key>method</key>
	    <string>enterprise</string>
	    <key>teamID</key>
	    <string>G00DC0NF1G</string>
	    <key>uploadBitcode</key>
	    <true/>
	    <key>uploadSymbols</key>
	    <true/>
	    <key>manifest</key>
	    <dict>
	        <key>appURL</key>
	        <string>foo.bar/app</string>
	        <key>displayImageURL</key>
	        <string>foo.bar/display-image</string>
	        <ke>fullSizeImageURL</key>
	        <string>foo.bar/full-sized-image</string>
	    </dict>
	</dict>
	</plist>

`-exportOptionsPlist` 的文档 （译者注：可用使用 `xcodebuild --help` 查看）：

	compileBitcode : Bool
	
	    For non-App Store exports, should Xcode re-compile the app from bitcode? Defaults to YES.
	
	embedOnDemandResourcesAssetPacksInBundle : Bool
	
	    For non-App Store exports, if the app uses On Demand Resources and this is YES, asset packs 
	    are embedded in the app bundle so that the app can be tested without a server to host asset 
	    packs. Defaults to YES unless "onDemandResourcesAssetPacksBaseURL" is specified.
	
	iCloudContainerEnvironment
	
	    For non-App Store exports, if the app is using CloudKit, this configures the 
	    "com.apple.developer.icloud-container-environment" entitlement. 
	    Available options: 
	        * Development 
	        * Production
	    Defaults to Development.
	
	manifest : Dictionary
	
	    For non-App Store exports, users can download your app over the web by opening your 
	    distribution manifest file in a web browser. To generate a distribution manifest, the 
	    value of this key should be a dictionary with three sub-keys: 
	        * appURL
	        * displayImageURL
	        * fullSizeImageURL
	    The additional sub-key "assetPackManifestURL" is required when using on demand resources.
	
	method : String
	
	    Describes how Xcode should export the archive. Available options: 
	        * app-store
	        * ad-hoc
	        * package
	        * enterprise
	        * development
	        * developer-id
	    The list of options varies based on the type of archive. Defaults to development.
	
	onDemandResourcesAssetPacksBaseURL : String
	
	    For non-App Store exports, if the app uses "On Demand Resources" and 
	    "embedOnDemandResourcesAssetPacksInBundle" isn't YES, this should be a base 
	    URL specifying where asset packs are going to be hosted. This configures the 
	    app to download asset packs from the specified URL.
	
	teamID : String
	
	    The Developer Portal team to use for this export. Defaults to the team used to build the archive.
	
	thinning : String
	
	    For non-App Store exports, should Xcode thin the package for one or more device variants? 
	    Available options: 
	        * none (Xcode produces a non-thinned universal app)
	        * thin-for-all-variants (Xcode produces a universal app and all available thinned variants)
	        * (a model identifier for a specific device (e.g. "iPhone7,1"))
	    Defaults to <none>.
	
	uploadBitcode : Bool
	
	    For App Store exports, should the package include bitcode? Defaults to YES.
	
	uploadSymbols : Bool
	
	    For App Store exports, should the package include symbols? Defaults to YES.

[↑目录](#table-content)

***

# <a name="work-in-both-worlds"></a>兼容Xcode 8 和 Xcode 7
兼容Xcode 8 和 Xcode 7是可能的，这样当时机正确的时候开发人员和 CI 服务器可以更新到 Xcode 8。为此，我们将使用 xcconfig 文件来设置这样做的配置，因为在 Xcode 目标编辑器中设置是非常不必要的，并且会导致混乱。

首先，我们想要知道什么是可以改变签名配置的变量：

- Build Configuration，这将由 `CONFIGURATION` 值确定。
- Target 类型，这将由 `WRAPPER_EXTENSION` 值确定。 （应用程序的 target 将此设置为 `app`）
- Xcode 版本，这可以通过一些由 Xcode 设置的构建配置来确定。我们将使用 `XCODE_VERSION_MAJOR` 。

所以，我们将创建两个变量，这两个变量能够解释这三个设置将具有的不同的值：

	CONFIGURATION_AND_VERSION = $(CONFIGURATION)_$(XCODE_VERSION_MAJOR)
	WRAPPER_EXTENSION_AND_CONFIGURATION_AND_VERSION = $(WRAPPER_EXTENSION)_$(CONFIGURATION_AND_VERSION)

通过在 xcconfig 文件中定义这两个变量，我们可以使用 xcconfig 文件中的变量替换行为来更改其他构建设置的条件分配，以修改 Xcode 将在构建中使用的下列几个签名配置：

- `CODE_SIGN_IDENTITY`
- `PROVISIONING_PROFILE`
- `DEVELOPMENT_TEAM`

### `CODE_SIGN_IDENTITY`

将采用不同属性的第一个变量是 `CODE_SIGN_IDENTITY`。在[Xcode 7及以前的签名机制](#sign-in-xcode7-prior) 节描述的配置风格中，此构建设置的值因使用的构建配置而异。这种行为在 Xcode 8中改变了，所以我们需要使用一个只在 Xcode 的主要版本之间不同的变量，`XCODE_VERSION_MAJOR`。

`CODE_SIGN_IDENTITY = $(CODE_SIGN_IDENTITY_$(CONFIGURATION_AND_VERSION))`

这一行说，在构建时，分配给 `CODE_SIGN_IDENTITY` 的值将基于另一个基于构建配置和 Xcode 版本的值构造的变量。通过这样做，您可以自定义它的每个变体：

	CODE_SIGN_IDENTITY_Debug_0700 = iPhone Developer
	CODE_SIGN_IDENTITY_Enterprise_0700 = iPhone Distribution
	CODE_SIGN_IDENTITY_Production_0700 = iPhone Distribution
	
	CODE_SIGN_IDENTITY_Debug_0800 = iPhone Developer
	CODE_SIGN_IDENTITY_Enterprise_0800 = iPhone Developer
	CODE_SIGN_IDENTITY_Production_0800 = iPhone Developer

在这种情况下，我们配置变量，使得与设计用于在 Xcode 7中构建的行为保持相同，同时更新值以与 Xcode 8中的期望状态相对应。实践中如下：

	CODE_SIGN_IDENTITY = $(CODE_SIGN_IDENTITY_$(CONFIGURATION_AND_VERSION))
	
	// To resolve the value of CODE_SIGN_IDENTITY we must first resolve CONFIGURATION_AND_VERSION
	CONFIGURATION_AND_VERSION = $(CONFIGURATION)_$(XCODE_VERSION_MAJOR)
	
	// So, therefore...
	CODE_SIGN_IDENTITY = $(CODE_SIGN_IDENTITY_$(CONFIGURATION)_$(XCODE_VERSION_MAJOR))
	
	// Now that that variable has been resolved, we need to resolve CONFIGURATION and XCODE_VERSION_MAJOR
	XCODE_VERSION_MAJOR = // This is defined by Xcode, it will be 0700 in Xcode 7 and 0800 in Xcode 8
	CONFIGURATION = // This is defined by the Xcode build system, it is the string name of the build configuration being used
	
	// Therefore, when we are building a Debug build in Xcode 8, it will resolve as such:
	CODE_SIGN_IDENTITY = $(CODE_SIGN_IDENTITY_Debug_0800)
	
	// which means we have to look up the value of CODE_SIGN_IDENTITY_Debug_0800
	CODE_SIGN_IDENTITY_Debug_0800 = iPhone Developer
	
	// which means the original assignment of CODE_SIGN_IDENTITY resolves as
	CODE_SIGN_IDENTITY = iPhone Developer
	
这种基于环境条件在运行时解析构建设置的值的方法是 Xcode 构建系统本身用于许多常见构建设置的惯用方式。这是一个可靠的系统，以便将整个开发团队转换到新版本的 Xcode。

### `PROVISIONING_PROFILE`

可以采取类似的方法来分配应当使用的描述文件（provisioning profile）的值。但是，这次我们在执行赋值时需要更加小心。由于描述文件仅在部署可执行二进制文件而不是任何可执行代码时使用，因此我们必须将描述文件的分配限制为仅应用程序 target。这可以通过为每个 target 创建多个 xcconfig 并使用不同的 xcconfig 文件来实现，但是我们更容易管理单个信息集而不是多个信息集。

	PROVISIONING_PROFILE = $(PROVISIONING_PROFILE_$(WRAPPER_EXTENSION_AND_CONFIGURATION_AND_VERSION))

设置 `PROVISIONING_PROFILE` 的值取决于以下构建设置的值：

1. `WRAPPER_EXTENSION`
2. `CONFIGURATION `
3. `XCODE_VERSION_MAJOR`

基于 [Xcode 7及以前的签名机制](#sign-in-xcode7-prior) 描述的方法，只有两种情况会在构建中使用特定的描述文件：

- 在 Xcode 7中构建企业分发的应用程序
- 在 Xcode 7中构建用于生产（App Store）分发的应用程序

我们将设置 `PROVISIONING_PROFILE` 值来一次性处理这两种情况，同时在所有其他情况下设置为“Automatic”。这意味着在 Xcode 8中自动签名仍然可以像 Xcode 7中一样正常工作。

	PROVISIONING_PROFILE = $(PROVISIONING_PROFILE_$(WRAPPER_EXTENSION_AND_CONFIGURATION_AND_VERSION))
	
	// First, expand the variable WRAPPER_EXTENSION_AND_CONFIGURATION_AND_VERSION
	WRAPPER_EXTENSION_AND_CONFIGURATION_AND_VERSION = $(WRAPPER_EXTENSION)_$(CONFIGURATION_AND_VERSION)
	
	// Therefore...
	PROVISIONING_PROFILE = $(PROVISIONING_PROFILE_$(WRAPPER_EXTENSION)_$(CONFIGURATION_AND_VERSION))
	
	// Now do the same for CONFIGURATION_AND_VERSION
	CONFIGURATION_AND_VERSION = $(CONFIGURATION)_$(XCODE_VERSION_MAJOR)
	
	// Therefore...
	PROVISIONING_PROFILE = $(PROVISIONING_PROFILE_$(WRAPPER_EXTENSION)_$(CONFIGURATION)_$(XCODE_VERSION_MAJOR))
	
	// Now we know we are looking for variables with follow the pattern:
	// PROVISIONING_PROFILE_$(WRAPPER_EXTENSION)_$(CONFIGURATION)_$(XCODE_VERSION_MAJOR)
	
	// Based on the criteria already defined as needing the two provisioning profiles defined for Xcode 7:
	PROVISIONING_PROFILE_app_Enterprise_0700 = 0be1f9f5-2c59-4a11-b118-2e9d046e5026
	PROVISIONING_PROFILE_app_Production_0700 = 21510e33-dbe3-4209-9506-e907b5c87742
	
当此变量在构建中扩展时，对于不能为这两个值解析值的任何构建，我们将为 `PROVISIONING_PROFILE` 构建设置分配一个空值。这将导致它在构建时自动解决，这是 Xcode 8中所有类型构建和 Xcode 7中 Debug 构建配置的预期行为。

### `DEVELOPMENT_TEAM`

如前所述，`DEVELOPMENT_TEAM` 构建设置是 Xcode 8新引进的：这意味着它不需要有条件地为每个版本的 Xcode 设置，因为只有Xcode 8将使用它。分配给此构建设置的值将取决于您的具体情况。如果您只是为一个开发团队构建，那么您可以直接分配此值，如下所示：

`DEVELOPMENT_TEAM = H3LL0W0RLD`

但是，如果您正在与多个开发团队（例如开发账号和企业账号）合作，那么您可能需要使用 `CONFIGURATION` ，以有条件地根据构建配置更改它：

	DEVELOPMENT_TEAM_Debug = H3LL0W0RLD
	DEVELOPMENT_TEAM_Enterprise = G00DC0NF1G
	DEVELOPMENT_TEAM_Production = H3LL0W0RLD

这导致能够对 Xcode 构建系统进行分析，以便基于正在使用的构建配置采用不同团队的签名身份。这将采取下面的赋值形式：

	DEVELOPMENT_TEAM = $(DEVELOPMENT_TEAM_$(CONFIGURATION))

此构建设置的值很重要， 因为在 Building， Deploying， 和 Archiving 时 Xcode 8会用到。

### `PROVISIONING_PROFILE_SPECIFIER`

签名系统的最后一个组件是新的 `PROVISIONING_PROFILE_SPECIFIER` 构建设置。对于我们，它不需要设置为任何东西，因为自动签名将接管并为我们设置。这意味着您可以在 xcconfig 文件中定义它，以防止无意中修改该值：

```
 `PROVISIONING_PROFILE_SPECIFIER` =
```

### 总结

生成的 xcconfig 文件应如下所示：

	CONFIGURATION_AND_VERSION = $(CONFIGURATION)_$(XCODE_VERSION_MAJOR)
	WRAPPER_EXTENSION_AND_CONFIGURATION_AND_VERSION = $(WRAPPER_EXTENSION)_$(CONFIGURATION_AND_VERSION)
	
	CODE_SIGN_IDENTITY_Debug_0700 = iPhone Developer
	CODE_SIGN_IDENTITY_Enterprise_0700 = iPhone Distribution
	CODE_SIGN_IDENTITY_Production_0700 = iPhone Distribution
	CODE_SIGN_IDENTITY_Debug_0800 = iPhone Developer
	CODE_SIGN_IDENTITY_Enterprise_0800 = iPhone Developer
	CODE_SIGN_IDENTITY_Production_0800 = iPhone Developer
	CODE_SIGN_IDENTITY = $(CODE_SIGN_IDENTITY_$(CONFIGURATION_AND_VERSION))
	
	PROVISIONING_PROFILE_app_Enterprise_0700 = 0be1f9f5-2c59-4a11-b118-2e9d046e5026
	PROVISIONING_PROFILE_app_Production_0700 = 21510e33-dbe3-4209-9506-e907b5c87742
	PROVISIONING_PROFILE = $(PROVISIONING_PROFILE_$(WRAPPER_EXTENSION_AND_CONFIGURATION_AND_VERSION))
	
	DEVELOPMENT_TEAM_Debug = H3LL0W0RLD
	DEVELOPMENT_TEAM_Enterprise = G00DC0NF1G
	DEVELOPMENT_TEAM_Production = H3LL0W0RLD
	DEVELOPMENT_TEAM = $(DEVELOPMENT_TEAM_$(CONFIGURATION))
	
	PROVISIONING_PROFILE_SPECIFIER =

这个配置文件可以同时兼容 Xcode 7和8，并且又能利用每个不同的签名系统的优势。有关如何配置 xcconfig 文件的更多详细信息，请查看[本指南](http://pewpewthespells.com/blog/xcconfig_guide.html)。    

[↑目录](#table-content)

***