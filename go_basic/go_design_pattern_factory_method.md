
#### 工厂方法模式理论

1. 为什么需要工厂方法模式？

   工厂方法模式也是一种创建型模式，产生的目的就是为了“创建对象”。那么为什么需要这种模式来创建对象，创建哪些场景的对象呢？工厂方法模式也是为了适应软件设计`开闭原则，依赖倒置原则`的，目的就是通过一种工程化方式，设计可拓展的程序。**面向接口开发。不需要太多的关注实现细节**

   > 定义一个用于创建对象的接口，让子类绝对实例化哪一个类，工厂方法使一个类的实例化延迟到其子类。
   > 总是引用接口而非实现类，能允许变换子类而不影响调用方，即尽可能面向抽象编程。

   在OOP中，通常都会有`抽象`的概念，通常在高级语言中，会通过内建的一些关键字进行表达，例如`abstrct,interface...`,目的就是让开发者通过这些抽象，去设计高可拓展，稳定的程序。


  例如在Java中，如果不是面向接口（抽象）去创建产品，那么在对象的创建过程，和对象的使用过程都会有局限性，因为太过于具体而不利于拓展。

   ```java
   // 例如有两个实体
   public class ProductA {
     
   }
   
   public class ProductB {
     
   }
   
   // 要打印所有的商品
   public Class ProductPrinter {
     // 为了突出目的，不用重写
     public void printA(ProductA pa){
       
     }
     public void printB(ProductB pb){
       
     }
     
     // ...  public void print(ProductB pb){
       
    //}
   }
   
   // 常规创建对象
   public static void main(String[] args) {
     ProductA a = new ProductA(); 
     ProductB b = new ProductB();
     // ProductC c = new ProductC();
     
     ProductPrinter printer = new ProductPrinter();
     printer.printA(a); // 分别调用两个方法
     printer.printB(b);
   }
   ```

   

   可以看到，两种相似的产品，需要分别使用产品对象进行创建，在产品打印类中，若是想要打印所有产品，还得分别定义多个方法。不利于拓展，并且当有新的产品添加的时候，代码改动地方会很多，有风险。

   当使用了面向接口编程，类似使用了工厂方法来创建对象，那么，代码可拓展型，稳定性就会变得很高：

   ```java
   // 例如有两个实体,Product是所有产品的抽象
   public class ProductA implements Product {
     
   }
   
   public class ProductB  implements Product {
     
   }
   
   public Class ProductPrinter {
     // 面向抽象作为入参传入,不用关心细节
     public void print(Product p){
       
     }
   }
   
   // 工厂方法，根据不同需求创建不同对象
   public class ProductFactoryMethod {
     private Product product;
     
     // 根据不同需求工厂方法newProduct创建不同的对象
     public Product newProduct(string productType){
       if "pa".equals(productType){
         product = new ProductA();
       }else if "pb".equals(productType){
         product = new ProductB();
       }
       return product;
     }
     
   }
   
   
   
   // 客户端调用代码复用性高，不用对变动进行大改
   public static void main(String[] args) {
     ProductFactoryMethod factory = new ProductFactoryMethod();
     ProductA a = factory.newProduct("pa")
     ProductB b = factory.newProduct("pb")
     // ProductC c = new ProductC();
     
     ProductPrinter printer = new ProductPrinter();
     printer.print(a);
     printer.print(b);
   }
   
   ```

   

   

2. 工厂方法有哪些组成？

   - 抽象产品类Product：负责定义产品的共性，实现对事物最抽象的定义
   - Creator为抽象类创建类：是抽象工厂，具体如何创建产品类是有具体的实现工厂ConcreteCreator完成的。

   基本就是两大类：产品对象Product + 抽象工厂，都是面向抽象编程。

   具体根据不同实现方式，其实在Java中还区分划出了`静态工厂方法，抽象工厂模式`等等。

3. 工厂方法有哪些应用的套路？

   抽象好产品Product，产品不一定非要是`struct`类型的，也可以是`interface`类型的。

   定义好抽象工厂，抽象工厂通常是`interface`类型的，接口内包含返回Product的方法（返回通常是interface，而不是struct)

   > 抽象的Product通常是struct+interface组合形成的抽象对象。

   

4. 工厂方法的优缺点？

   - 优点

     把初始化实例时的工作放到工厂里进行，使代码更容易维护。 更符合面向对象的原则，面向接口编程，而不是面向实现编程。

     

     

   - 缺点

     每次增加一个产品时，都需要增加一个具体类和对象实现工厂，使得系统中类的个数成倍增加，在一定程度上增加了系统的复杂度，同时也增加了系统具体类的依赖。要新增产品类的时候，就要修改工厂类的代码，违反了开放封闭原则。



#### 工厂模式实践应用

##### Go中应用

> 在golang中，工厂模式的应用最核心的一点就是：需要将常规的封装（属性+方法）进行拆分。
>
> struct（属性定义） + interface（方法定义） => struct需要实现interface的方法。

- 简单工厂模式
  简而言之，就是通过客户端传入内容，工厂进行判断，返回对应的产品对象。

  1. 定义产品抽象，**重点是对产品共同行为进行抽象**。
     假设设定抽象产品是：不同渠道的充值订单。根据不同渠道标示manner进行不同操作，创建不同订单对象。

     ```go
     // 根据不同的支付渠道manner创建不同的订单对象Product
     type PayOrder struct {
     	Id            int64   `json:"id"`
     	Order         string  `json:"order"`
     	Uid           int64   `json:"uid"`
     	Peer          int64   `json:"peer"`
     	Money         float64 `json:"money"`
     	Lc            string  `json:"lc"`
     	Cv            string  `json:"cv"`
     	Ua            string  `json:"ua"`
     	Option        string  `json:"option"`
     	MannerId      string `json:"manner_id"` // 区分渠道
     }
     ```

     针对不同支付订单，抽取相同共性的行为，若是没有的话，可以自定义一个`Get`方法，目的就是抽象出一个`interface`, 与`struct`结合抽象才能比较方便利用设计模式。

     ```go
     // 定义一个所有Product的抽象
     type OrderInterface interface {
     	Get() interface{}
     }
     
     func (po *PayOrder) Get() interface{}{
       return po
     }
     
     ```

  2. 定义抽象工厂，根据不同的内容创建不同对象
   通常简单工厂模式，就是`通过单个工厂类，去创建多个不同的对象Product`,改种弊端就是若是新增产品，那么必须在工厂类中需要修改，新建分支。
  
     > 这里需要注意的一点是：工厂方法创建的对象不是传统Java中的Product，在golang中，工厂方法返回的是Product中具有相同行为的抽象，即为方法=> interface

     ```go
   	// 静态工厂对象
     	type OrderStaticFactory struct {
     
     	}
     
     	func newOrderFactory() *OrderStaticFactory {
     		return &OrderStaticFactory{}
     	}
     
     	// 定义简单工厂创建不同的product的方法,这里返回的不是Product，而是Product's 方法
     	// 创建对象的静态工厂
     	func (of *OrderStaticFactory) createOrder(param BaseReq) (OrderInterface,int,error) {
     		manner := param.Manner
     		// lc := param.Lc
     		// 可以做一些公共的基础逻辑校验，例如通用参数校验，session校验等
     		// orderCreateCommonCheck(param) ....
     		// sessionValid(param) ...
     		// 尽量减少分支，只是根据一级manner进行划分分支
     		switch manner {
     		case "alipay":
             // createAlipayOrder(param)
             // 内部service再根据具体manner，lc进行创建订单
     		case "weixin":
     				// createWechatOrder(param)
     				// 内部service再根据具体lc进行创建订单
     		case "apple":
     				// createAppleOrder(param)
     		case "googlepay":
     				// createGooglePayOrder(param)
     		}
     		return &PayOrder{},0,nil
     		}
     
     ```
  
  3. 使用方式Usage
  
     ```go
     	func main() {
       	// 创建工厂对象，并根据传入内容进行创建具体的Product
         	param := BaseReq{}
   	  	payOrder,code,errr := newOrderFactory().createOrder(param)
         	fmt.Printf(payOrder.Get().Order)
   	}
     ```
  
     
  
- 工厂方法模式

  与简单工厂模式不同，工厂方法模式：`每个具体工厂类只负责创建对应的产品`,也就是产品Product与Factory工厂是一一对应的。好处就是，不像简单工厂模式，新建产品Product还是需要修改单一的工厂方法，而工厂方法模式因为产品与工厂是一一对应，则不需要改动之前的产品代码。

  > 每一种产品Product都会有具体的Factory与之对应，具体的Factory来创建具体的产品。
  >
  > ```
  > ////// 工厂方法模式 ////
  > // WechatOrderFactory --> WechatOrder
  > // AlipayOrderFactory --> AlipayOrder
  > // AppleOrderFactory --> AppleOrder
  > // GooleOrderFactory --> GoogleOrder
  > ```

  1. 定义抽象产品Product，不同的具体产品需要继承该产品。

     在golang中，对抽象产品Product的继承，就是通过`将一个struct，作为属性定义在子类struct中`.通过**struct+interface**组合成了`封装`的概念。定义抽象工厂方法，并针对不同的子产品，定义不同具体工厂实现去创建产品。

     ```go
     		// 定义一个所有Product的行为抽象
     		type OrderInterface interface {
     			Get() interface{}
     		}
     
     		// 定义不同的支付订单product
     		type AlipayProduct struct {
     			PayOrder
     		}
     		func (ap *AlipayProduct) Get() interface{}{
     			return ap
     		}
     
     		type WechatProduct struct {
     			PayOrder
     		}
     		func (wp *WechatProduct) Get() interface{}{
     			return wp
     		}
     
     		type AppleProduct struct {
     			PayOrder
     		}
     		func (ap *AppleProduct) Get() interface{}{
     			return ap
     		}
     
     		type GoogleProduct struct {
     			PayOrder
     		}
     		func (gp *GoogleProduct) Get() interface{}{
     			return gp
     		}
     
     ```
     
  2. 因为工厂方法的概念，就是需要根据抽象的工厂方法，创建具体的工厂方法，来创建具体的子产品。

     ```go
  		// 定义抽象的工厂方法，注意此处返回的是Product的行为抽象,抽象工厂interface，内部有抽象方法返回抽象的产品Product
     		type AbstrctOrderFactory interface {
     			createOrder(param BaseReq) OrderInterface
     		}
     		// 分别定义不同具体工厂创建对象
     		type WechatOrderFactory struct {
     		}
     
     		func (wof *WechatOrderFactory) createOrder(param BaseReq) OrderInterface {
     			// createWehchatOrder(param)
     			return &WechatProduct{}
     		}
     		type AlipayOrderFactory struct {
     		}
     		func (wof *AlipayOrderFactory) createOrder(param BaseReq) OrderInterface {
     			// createAlipayOrder(param)
     			return &AlipayProduct{}
     		}
     		type AppleOrderFactory struct {
     		}
     
     		func (wof *AppleOrderFactory) createOrder(param BaseReq) OrderInterface {
     			// createAppleOrder(param)
     			return &AppleProduct{}
     		}
     		type GoogleOrderFactory struct {
     		}
     
     		func (wof *GoogleOrderFactory) createOrder(param BaseReq) OrderInterface {
     			// createGoogleOrder(param)
     			return &GoogleProduct{}
     		}
     ```
     
     
     
  3. Usage使用方式
     和标准定义工厂方法模式略有不同，此处是直接创建具体的工厂对象，进而来创建具体的子产品。
     因为抽象工厂在golang中是interface类型的，不能直接实例化的。

     ```go
     
     	func Test_Fm(t *testing.T) {
     		param := BaseReq{}
     		// 创建微信订单对象
     		wechatFactory := &WechatOrderFactory{}
     		wxOrder := wechatFactory.createOrder(param).Get()
     		fmt.Println(wxOrder)
     
     		// 创建支付宝订单
     		alipayFactory := &AlipayOrderFactory{}
     		alipayOrder := alipayFactory.createOrder(param).Get()
     		fmt.Println(alipayOrder)
     
     		// 创建apple订单
     		appleFactory := &AppleOrderFactory{}
     		appleOrder := appleFactory.createOrder(param).Get()
     		fmt.Println(appleOrder)
     
     		// 创建google订单
     		googleFactory := &GoogleOrderFactory{}
     		googleOrder := googleFactory.createOrder(param).Get()
     		fmt.Println(googleOrder)
     	}
     ```

  

  ##### 总结

  根据上面创建型过程，是否在不是传统意义的设计模式上，将两种结合，第一个步进行外部渠道的路由，第二步则进行具体渠道订单的创建。

  ```go
  	func (of *OrderStaticFactory) createOrder(param BaseReq) (OrderInterface,int,error) {
  		manner := param.Manner
  		// lc := param.Lc
  
  		// 可以做一些公共的基础逻辑校验，例如通用参数校验，session校验等
  		// orderCreateCommonCheck(param) ....
  		// sessionValid(param) ...
  
  
  		// 尽量减少分支，只是根据一级manner进行划分分支
  		switch manner {
  		case "alipay":
        	alipayFactory := &AlipayOrderFactory{}
  				alipayOrder := alipayFactory.createOrder(param).Get()
        	return alipayOrder
  		case "weixin":
  				wechatFactory := &WechatOrderFactory{}
  	    	wxOrder := wechatFactory.createOrder(param).Get()
        	return wxOrder
  		case "apple":
  		  	appleFactory := &AppleOrderFactory{}
      		appleOrder := appleFactory.createOrder(param).Get()
  		  	return appleOrder
  		case "googlepay":
  	   		googleFactory := &GoogleOrderFactory{}
  	    	googleOrder := googleFactory.createOrder(param).Get()
        	return googleOrder
  		}
  		return &PayOrder{},0,nil
  	}
  
  
  
  	// client usage
  	func main() {
      // 创建工厂对象，并根据传入内容进行创建具体的Product
     	 param := BaseReq{}
  	 	 payOrder,code,errr := newOrderFactory().createOrder(param)
      	fmt.Printf(payOrder.Get().Order)
  	}
  
  	// 或者能够更简化，其实根据渠道分支判断逻辑总是少不了，无法是通过其他映射关系，字段来选择。
  	// 只是在程序的粒度上又进行变换,这样可以缩小变换的范围
  	func (of *OrderStaticFactory) createOrder(param BaseReq) (OrderInterface,int,error) {
  		manner := param.Manner
  		// lc := param.Lc
  
  		// 可以做一些公共的基础逻辑校验，例如通用参数校验，session校验等
  		// orderCreateCommonCheck(param) ....
  		// sessionValid(param) ...
  
  
  		// 尽量减少分支，只是根据一级manner进行划分分支
    	mannerFactory := getFactoryByManner(param.manner)
    	return mannerFactory.createOrder(param).Get(),0,nil
  	}
  
  
  	func getFactoryByManner (manner string) AbstrctOrderFactory {
    	switch manner {
  		case "alipay":
        	return &AlipayOrderFactory{}
  		case "weixin":
        	return &WechatOrderFactory{}
  		case "apple":
  		  	return &AppleOrderFactory{}
  		case "googlepay":
  	   		return &GoogleOrderFactory{}
  		}
    	return nil
  	}
  
  ```

  




