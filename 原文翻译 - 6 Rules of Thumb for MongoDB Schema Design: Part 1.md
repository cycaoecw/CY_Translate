原文作者: William Zola, Lead Technical Support Engineer at MongoDB

[原文链接](https://www.mongodb.com/blog/post/6-rules-of-thumb-for-mongodb-schema-design-part-1 "6 Rules of Thumb for MongoDB Schema Design: Part 1")

“虽然我有很多SQL的经验，但是对于MongoDB，我还是只是个菜鸟。我想知道对于“1对N”的关系要设计数据模型呢？”这是我在MongoDB公司里最常碰到的一个问题。

我不会有一个简短的答案，因为有各种各样的方法，所以没有一个唯一的答案。MongoDB有丰富的和细微差别的语法来表达在SQL世界了的“1对N”关系。让我帮你了解怎样去选择那个“1对N”关系模型更适合你的需求。

这里有太多的东西需要说， 我会把这些分成3部分。在第一部分，我会讲的是3种最基本的“1对N”关系模型。在第二部分，会涉及更高层次的架构设计，包括反标准化和双向引用。同样在第一部分，我会回顾所有的选择，然后给你一些从这上千种（真的是上千）选择中考虑使用哪个“1对N”关系模型的建议。

很多初学者都认为在MongoDB中，只有一种方法去模型化“1对N”关系。就是把子文档嵌套到父文档里面，但是这个不是对的。因为不是因为你可以嵌套文档就一定要嵌套文档。

在开始设计一个MongoDB架构的时候，你要问几个在设计SQL的时候不会问的问题：这些关系的基数是多少？说正式点就是，你需要搞清楚这些“1对N”关系的更多细微差别。例如，这是1对少数数量的关系，还是1对中等数量的关系，或者是1对海量数量的关系？你就要根据这些条件使用不同的模型来表现它们的关系。


**基本模型：1 vs 少数数量**

一个"1 vs 少数数量"的列子是个人地址。这是一个很好的情景去使用嵌套文档：把多个地址放到一个数组里面，再把这个数组放到Person这个实体里：

[引用代码](https://gist.github.com/amyberman3/691ac932f11a94c3f300#file-gistfile1-txt "gistfile1.txt")

```
> db.person.findOne()
{
  name: 'Kate Monster',
  ssn: '123-456-7890',
  addresses : [
     { street: '123 Sesame St', city: 'Anytown', cc: 'USA' },
     { street: '123 Avenue Q', city: 'New York', cc: 'USA' }
  ]
}
```
这个设计具有嵌套设计的所有优点和缺点。主要的优点就是不需要执行另外的查询去获得嵌套信息的详情。但是缺点也很明显，就是没有办法把这些嵌套信息作为一个单独的实体来访问。

一个例子就是，如果你正在构造的是任务跟踪系统，每个人都有一系列的任务。把任务嵌套到Person里面的话，实现类似于"查看明天所有的任务"的查询会比其他的架构设计更难实现。我将会在下一个文章介绍一个更加适合这个情景的设计。

**基本模型：1 vs 中等数量**

一个"1 vs 中等数量"的列子就是产品的替换零件的订单系统。每个产品会有几百个零件可以替换，但不会有几千个(它们有不同规格的螺丝，垫圈，密封垫等等)。这个场景非常好的对应这个系统。你可以把ObjectIDs 放到一个零件数组里面，再把数组放到Product这个document里。(在这个例子里，因为方便阅读我使用的是2-byte 的ObjectIDs ，但是在真实环境中是使用12-byte的ObjectIDs)。

每个零件都有它自己的document:
[代码引用](https://gist.github.com/amyberman3/fc40a0e3222c8fed43f1#file-gistfile1-txt "gistfile1.txt")

```
> db.parts.findOne()
{
    _id : ObjectID('AAAA'),
    partno : '123-aff-456',
    name : '#4 grommet',
    qty: 94,
    cost: 0.94,
    price: 3.99
}
```

每个产品也会有自己的document, 而且每个产品都包含了一个纪录了该产品的替换零件的ObjectID 的数组.
[代码引用](https://gist.github.com/amyberman3/9b07ff94031ca2af6694#file-gistfile1-txt "gistfile1.txt")

```
> db.products.findOne()
{
    name : 'left-handed smoke shifter',
    manufacturer : 'Acme Corp',
    catalog_number: 1234,
    parts : [     // array of references to Part documents
        ObjectID('AAAA'),    // reference to the #4 grommet above
        ObjectID('F17C'),    // reference to a different Part
        ObjectID('D2AA'),
        // etc
    ]
```
这样你就可以使用应用层面的连接(application-level join)去获取某个产品的全部可替换零件:
[代码引用](https://gist.github.com/amyberman3/4065561e7c817af4bd19#file-gistfile1-txt "gistfile1.txt")

```
 // Fetch the Product document identified by this catalog number
> product = db.products.findOne({catalog_number: 1234});
   // Fetch all the Parts that are linked to this Product
> product_parts = db.parts.find({_id: { $in : product.parts } } ).toArray() ;
```
为了执行更加高效，你需要建立在产品的目录号码(products.catalog_number)上面建立索引(index). 需要注意的是，替换零件id(parts._id)上总是需要一个索引(index)以保证查询永远的高效。

这种类型的引用嵌套也有各自优缺点。每个部分都是相对独立的文档(document)， 所以很容易去独立查找和修改。但是使用这种架构的代价就是需要使用二次查询才能得到产品的替换零件的详情(但是这个会再denormalizing 的第二部分得到解决)。

作为附加的奖励，这个架构可以让你有独立的替换零件组(Parts)给到各种不同的产品(Products)。这就意味着这个"1 vs N"的结构可以变成一个"N vs N"的结构，而且不需要任何连接表!
