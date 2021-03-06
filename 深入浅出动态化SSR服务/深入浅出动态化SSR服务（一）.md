深入浅出动态化SSR服务（之一） - 开发工具篇
----

### 目录
* 开发工具篇 🔥
* SSR服务篇
* 架构篇

### 简述
在前端还未系统化之时的刀耕火种时代，已经有非常多的生成页面的工具，其可视化的方式极大的赋能了非技术人员，加快了业务的迭代速度。如今，随着前端技术的发展和复杂化，我们看到越来越多以组件为基础的页面级可视化生成工具已经出现在了各大领域。

依靠`Vue`/`React`等UI组件框架和逐渐易用化的`Webpack`等编译工具的出现，编写一套符合自己业务需求的前端可视化页面生成工具并不复杂。但大部分的页面可视化系统大都以完全纯前端思维来打造，对其论述也仅仅是大体的做法和原理剖析，并没有以前端工程化的角度来阐述其中的困难与细节。

在 《深入浅出动态化SSR服务》系列文章中，我们将以较为深入的前端工程化角度来讲述如何打造一款动态化且支持SSR的页面可视化系统。希望大家能以此为参考，更深入的思考前端工程化的实践，并能在业务当中有相关的提升。

本系列文章分为三个部分：
1. 开发工具篇： 当前篇我们会较为深入和系统地讲述`sis`的编写过程，其中涉及到技术选型考量，编译前端相关的知识。同时我们会适当讲述不直接采用`Vue CLI`等工具链的原因及其相关的不足点，更多的内容在《SSR服务篇》中我们还会通过测试结果进行更深入分析和讲述。
2. SSR服务篇：在这一篇中我们会探讨当前SSR服务性能和稳定性相关的问题，在压力测试工具以及`Node`相关Profile帮助下优化我们的SSR服务，达到单机服务高稳定性、高性能、动态化的要求。
3. 架构篇：在这一篇中我们会讲述此页面可视化系统的架构层面的一些思考和实践，从更全局和整体的角度来探讨如何保证系统的高稳定和高性能化要求。

在此，我非常感谢百度的FEX团队产出的 [FIS工具](http://fis.baidu.com/)，其中非常多关于前端工程化的思考都被融入到了`sis`以及`sis-ssr`中。同时我也非常感谢与我亦师亦友的 [FIS作者/前端工程化先行人 - 张云龙](https://github.com/fouber) 从全民直播至今对我整体技术方向的指引和多维度方法论的培养。

### 项目背景
如今市面上的页面可视化系统主要以纯前端技术为主，并没有对结果页面进行SSR同构。究其原因可以总结为：纯前端技术的整体系统实现较为容易，对于其产出结果只需要由CDN来进行加速，即可完成抵抗单页面大流量的需求。我们可以看到目前市面上类似的运营/营销产品可视化系统大体都是如此。

但是SSR同构服务的引入是有其积极意义的，其核心的两点在于：
1. SEO友好
2. 首屏渲染的加速

对于很多站点而言，其在SEO上的需求与大型厂商的系统有很大的不同：有非常多站点的产品页面仍然需要Google等搜索引擎进行收录。这点我们可以从开放的 [robots.txt](https://www.coupang.com/robots.txt) 文件看到：

```html
User-agent: Googlebot
Crawl-delay: 1
Allow: /vp/products/
Allow: /vm/products/
Disallow: /*.css$
Disallow: /*.js$
```

在这里可能有部分读者会问：“Google不是可以对SPA页面进行收录么？”，其实不然，我们在Google的相关解释中可以找到这样一句话：
> Note: that as of now, only can index synchronous JavaScript applications just fine.

因此，当我们的页面如果存在异步请求然后再进行页面渲染的话，Google仍然是无法进行有效收录的。那么基于`Headless`的预渲染服务（Prerendering）又会如何呢？由于`Headless`需要启动完整的浏览器核心进行渲染，因此对于服务端性能是极大的消耗（尽管可以创造`Headless`的对象池进行重用，但渲染时性能仍然有较大的服务器端消耗）。针对较少的营销页面还可以，但是在需要整站SSR这个场景下，这并不能发挥很好的作用。

其次，特别是对于电商类产品而言，根据Amazon的页面加载延迟与收入关系的实践数据（每增加100ms网站加载延迟将导致收入下降1%），我们需要非常强力的支持进行首屏渲染的加速，降低内容到达时间(time-to-content)，保证更好的用户体验和更高的用户留存。

与此同时，我们也应该支持非常灵活的页面可视化搭建平台来应付大量的日常化运营需求，并且也应该具有开放的能力和通用性，能很好的支撑公司其他团队的业务。在此背景下，我们稍加总结就能清晰的得到这套系统需要达成的目标，即：
* 组件化及可视化
* 能够进行SSR服务的渲染
* 动态化，测试及发布不涉及核心的SSR渲染服务
* 足够灵活，其他团队能很好的接入及使用
* 能够保障服务安全，并做到业务隔离

### 技术选型
对于SSR服务而言，存在两种思路体系可以选择：
1. `HandleBars`等以纯字符串渲染引擎为主的思路
2. `Vue`/`React`等现代UI框架以Virtual DOM为主的思路

`HandleBars`等字符串渲染引擎为主的思路优点在于：性能。但是对于现代的前端开发来说，难以地很好的利用`NPM`生态，对开发不是很友好。而以`Vue`/`React`等现代UI框架的Virtual DOM的思路，能够很好的利用生态，但是缺点也是相当明显的，即：由于编译阶段会产生非常多的Virtual DOM对象，因此在渲染性能和内存占用相比`HandleBars`等字符串渲染引擎的思路而言并不占优。

当然，在我们技术选型时，我们一般首先以**开发友好**为准则进行，毕竟效率即是一切。因此`Vue`/`React`的方案是我们理所当然会去选择的。但是，`Vue`/`React`等现代UI框架的所有出发点总归是以纯前端为主，后端渲染为辅的思路在做支持，因此其所包含的工具链（`Vue CLI`等）支持并不能很好的满足我们自身的需求，举个例子：

> 团队A和团队B互不干涉的分别基于此系统开发两个项目C1与C2，此时，C1和C2项目都引用相同版本的诸如Vue、Lodash等公共依赖。现将C1与C2进行相关的同构打包，在不做编译工具调整的情况下，会产出D1和D1两个发布包。

我们可以看到，在这个例子之下，D1和D2必定会包含相同的公共依赖。这对于开放且动态化的SSR服务而言是极度不友好的，因为代码包体越大，无关代码越多，那么服务本地初始化的IO、CPU和内存占用成本势必会随之增加，这点我们在之后的《SSR服务篇》可以更深入地分析得到。其次，对于浏览器端的而言，因为静态资源的加载时间被增加了，也会增加更多的用户操作响应时间。考虑这样一个场景：

> D1发布结果包含了100个组件，但是对于某一生成页面而言，只需要对2个组件进行重复渲染。

在这个场景中，不管是服务端还是浏览器端，我们都需要浪费大量的IO、CPU和内存在剩下的98个组件代码之中。当然，在这些场景里面，类似`Webpack`之类的编译工具仍然可以通过开发者标注`Dynamic Load`的方式来进行按需加载，但这也造成了非常大的开发负担，我们希望整个过程是编译工具自动完成的。因此，直接使用`Vue`/`React`等现代UI框架的现有工具链是完全无法满足我们的系统要求的。我们需要对现有工具链进行替代或改进。

总而言之，最后我们确定的技术选型为：
* `Vue`
* `ElementUI`（可视化后台所需）
* 改进的编译工具(`sis`)

需要注意的是，**整个系统设计上实际与UI库/框架是无关的**，但我建议我们仍然需要在开发及生产期间固定你的技术选型，以此来避免因为无技术选型造成的项目不可维护及混乱，可以把这个看做是一个内部强制的约定。当然接下来的内容我们都会以`Vue`来进行讨论，如果有通用渲染服务的需求，可以在此基础上进行参考。

同时对于页面可视化系统来说，开发应该只是关注于组件的开发，而较少考虑外部系统的逻辑，因此我们一般采用如下的项目目录结构：

 ![目录结构](https://static.petera.cn/img1.png) 

在这个基础上，我们所有最终的编译关注点应是在`components`目录，这是我们编译的目标目录。而其他文件主要是用作本地开发时所用，在最终的结果中并不引入。

### 资源的加载分析
在上面我们已经分析过，直接使用`Vue CLI`等工具链并不能很好的满足当前系统的需求，我们需要更灵活的**按需加载**的编译支持，并且希望这个支持是不需要开发干涉的。反观现有的编译工具而言，其更多是在部署之前进行相关的代码静态分析并整体打包。如果我们需要更灵活的**按需加载**，那么唯一的方式是在运行时能够获得当前页面所需要的组件代码然后整合后运行，如图：

 ![页面](https://static.petera.cn/img2.png) 

从图中的逻辑我们可以看到，当页面仅需要ComponentA组件时，我们仅需要查询表中的ComponentA及其依赖的加载地址从而返回，然后由浏览器动态的完成整个加载，即可达成我们的要求。在整个过程中并不需要拉取ComponentB的代码，从而减少了加载和代码运行的耗时。

那么我们如何拿到这个依赖关系呢？实际上对于现代的编译工具而言，在进行编译时期就已经产出了对应的依赖关系了。我们只需要对其进行一些加工即可满足我们的需求。在编写`sis`的过程中，我会选择了 [Parcel](https://parceljs.org/) 来充当这一角色，其原因在于：
* `Parcel`遵循0配置的原则，开箱即用，使用简单
* `Parcel`有非常灵活的编译相关的接口，很容易进行编译工具的二次开发
* `Parcel`默认编译即采用多核编译及产出编译Cache，编译速度极快

`Parcel`相较于`Webpack`而言，在轻、重度使用上都会更胜一筹。

> 其中最主要的原因是我比较喜欢的`Parcel`的`代码即配置`而不是`Webpack`的`配置优先`的原则。当然你也可以使用`Webpack`拿到相关的信息进行二次加工，其做法并无太大差异。

我们拿一个简单的`A.vue`代码举例，代码如下：
![代码](https://static.petera.cn/img10.png)

使用`Parcel`拿到相关的资源依赖关系十分简单，其代码如下:
![代码](https://static.petera.cn/img11.png)

其中`assets`是一个Bundler对象，其结构经过必要简化仅保留我们关心的数据后，大致如下（如需要更详细和准确的结构信息，请参考`Parcel`文档）：
![代码](https://static.petera.cn/img12.png)

现在我们已经拿到了对应的依赖关系，接下来需要的工作就是将此树形结构的依赖转换成我们所期望的样子：
![代码](https://static.petera.cn/img13.png)

实际上，对于`sis`而言，其处理过程包含5个阶段，其分别是：
1. transform: 将`Parcel`的依赖数据JSON化，方便我们输出查看及调试
2. analysis: 分析对应的依赖结果，展平整个依赖数据，并且更改对应一些节点信息
3. checker: 检查对应代码是否符合强制的规范要求
4. optimize: 对依赖结果进行合并分析优化，并添加`AMD`模块化代码
5. output: 输出对应的代码及依赖关系JSON

在这里我们仅介绍5个阶段中的4个，而不对耦合具体业务需求的checker进行介绍。同时我们也主要以介绍思路为主，而简化了大部分的实现，实际上`sis`处理了非常多繁琐的Corner Case来保证编译的正确性。

### 简化资源结构
首先，为了调试的简便性，我们先将`Parcel`嵌套的Bundler对象简化为JSON数据，其代码如下：
![代码](https://static.petera.cn/img14.png)

经过此函数的处理，我们将Parcel的Bundler对象嵌套简化成了JSON数据，其结果为：
![代码](https://static.petera.cn/img15.png)

需要注意的是，这一步并不是必须的，如果为了编译时的性能，我们可以直接针对嵌套的Bundler对象进行后续处理。

### 展平资源结构
为什么需要将树形结构进行展平？其中的原因很简单的，就是为了后续更容易分析。在之后的optimize阶段我们需要大量的在依赖对象中进行跳转和改写，对树形结构展平，性能会更好，同时更容易达成这一目标。

将树形结构进行展平的代码很容易，代码如下：
![代码](https://static.petera.cn/img16.png)

此处`getVersion`函数作用是获取到当前依赖的版本号，其根据文件所在路径向上查找最近的`package.json`文件中的`version`字段获得。而`md5`函数是将对应的generated字段进行`JSON.stringify`后`md5`，然后返回一个7位长度的字符串。

通过此函数我们可以将对应的树形结构成功进行展平，其结果如下：
![代码](https://static.petera.cn/img17.png)

一切准备就绪，我们可以进入`sis`最有乐趣的optimize阶段了！

### 依赖的合并优化

回顾我们上面的讲述，为了保证可视化系统`按需加载`，我们依靠`Parcel`输出了依赖的JSON。从示例上我们可以看到，如果需要加载`A.vue`，那么我们只需要扫描整个对象，拿到`A.vue`、`lodash`、`Buffer`等代码的访问路径就行了。这看起来相当完美，但在实际应用中，这仍然远远不够。

我们知道对于浏览器来说，其静态资源的并发请求是存在限制的，在日常开发中我们并不会有这么简单的依赖关系。假如我们将`ElementUI`引入到A组件中然后编译，我们会发现，整个JSON文件有230多项依赖信息，即使我们在A组件中仅仅添加入一个`ElementUI`的`Button`组件，其所需要动态加载的文件数量就高达80多个！这显然不是我们想要的结果。针对这个问题我们会很自然地想到`合并`，现在的问题便会转化为：“我们如何知道哪些模块需要被合并呢？”。

假设我们有一个较为复杂的依赖关系，如图所示：

 ![依赖关系图1](https://static.petera.cn/img22222_meitu_5.png) 

其中包含循环引用（G-F-E-G）以及自引用（E-E)，那么我们怎么确定该合并哪些模块呢？当然，可能有读者会说：“ 你这个图错了，实际开发中不可能存在循环引用和自引用的！”。但事实上这在静态分析中是经常会发生的事情。

例如，当我们针对上面的简单示例使用`Parcel`编译后我们会发现，`Buffer`模块竟然自引用了自己！究其原因是因为`Parcel`为了统一Browser和Node端的代码而添加了`polyfill`代码。在编译时`Parcel`发现`lodash`引用的`Buffer`库中有这么一代码时：
![代码](https://static.petera.cn/img18.png)

就帮我们引入了`Buffer`的`polyfill`模块，从而造成了自引用。而至于循环引用，当我们编写了类似的如下代码：
![代码](https://static.petera.cn/img19.png)

不难发现，此中存在循环依赖为：C-B-A-C，但当我们使用Node执行此代码后会发现其能很正确的进行执行并输出：
![代码](https://static.petera.cn/img20.png)

从静态分析的角度来说，类似的代码在我们的依赖分析中很容易就会形成自引用与循环引用。针对以上问题，我们首先需要破坏依赖表中的自引用和循环引用（至于为什么这样做能够仍然保证正确的运行，就留给读者自行思考了）。其代码如下：
![代码](https://static.petera.cn/img21.png)

执行后，我们就进一步得到了如下的依赖关系：

 ![依赖关系图2](https://static.petera.cn/img4.png) 

根据上图稍加分析我们可以知道，在这里的依赖关系中，F节点实际上可以与G节点合并生成G+F节点，完成一次合并。其合并的条件为F的**入度为1**，更简单的说法是，F的父亲节点唯一。因此我们可以得到第一次合并的结果：

 ![依赖关系图3](https://static.petera.cn/img5.png) 

按照此规律我们依次进行合并，其过程如下：

 ![依赖关系图4](https://static.petera.cn/img6.png) 

1. 将G+F节点与C节点合并（G+F节点入度为1）

 ![依赖关系图5](https://static.petera.cn/img7.png) 

3. 将G+F+C节点与E节点合并（E节点入度为1）

 ![依赖关系图6](https://static.petera.cn/img8.png) 

4. 将G+F+C+D节点与E节点合并，合并结束。

通过这个算法，我们可以将引用数为1的模块尽可能合并，以此减少静态资源加载所需的请求数。最终我们的JSON将会形成如下的形式：
 ![代码](https://static.petera.cn/img22.png) 

其中我们对于每个合并的单节点新增了`pkg`字段，同时也形成了一个`__pkg0`节点记录了所有引用的模块。我们再次对相同的`ElementUI`项目进行测试，发现其依赖项从230项减少到了51项，大部分模块都被正确的合并了。

但与此同时我们也发现还有少部分模块因为被多个组件公用，从而被独立划分出来。当然此类文件也有很多显著的特点，例如：文件体积不大（大部分集中在1K以下），并且大部分都是非常基础的功能代码实现（如merge/filter/map等逻辑处理）。针对这些小文件，我们仍然可以在分析过程中再次进行合并。但在实际的`sis`实现中并没有这么做，而是选择使用Combo服务帮助我们完成了这部分功能，相关内容我们在《架构篇》会讲述。

### 给资源加上模块系统

`Parcel`提供给我们的数据中，其`generated`包含了已经过babel等编译过的具体代码，其结构大致如下：
 ![代码](https://static.petera.cn/img23.png)

针对这些转移的代码，我们需要给其加上`AMD`的相关模块外层代码，例如:
 ![代码](https://static.petera.cn/img24.png)

同时，我们可以自行编写一个40行的AMD实现来完成模块的定义执行（当然你也可以使用开源的实现），其大体实现如下：
 ![代码](https://static.petera.cn/img25.png)

至此，所有模块内容的分析和改写工作就已完成，我们只需要根据JSON依赖表将各项代码输出到文件就完成了整个开发工具的工作。与此同时，JSON依赖表也应被同时输出，供我们之后的服务端和浏览器端进行运行时分析并提供加载依据。

### 更多功能
`sis`作为一个开发工具，除了进行最终编译功能外，还应该增加一些方便开发的功能，如图：
 ![代码](https://static.petera.cn/33333.png	)

这些功能除了加快初始化项目的创建外，还沉淀了一些项目的**最佳实践**以便帮助新人能够更快的融入到项目开发之中。


### 总结与期待

在本篇中，我们详细讲述了项目的背景及对应的目标，并分析了如何依靠依赖表的方式优化对应的加载性能以及达到SSR目标中的动态化。当然整个过程我们主要是以讲述思想为主，抛开了非常多繁琐复杂的相关工作细节。

在下一篇文章中我们会继续详细讨论如何使用依赖表来进行SSR的**按需加载**的动态化，同时对`sis-ssr`和目前传统实现的SSR服务进行相关压力测试并分析相关的测试结果，从而清楚此种方式在单机性能上的边界。当然我们也对`sis-ssr`在可靠性上的提升做更多的介绍。敬请期待 《深入浅出动态化SSR服务（之二） - SSR服务篇》。
