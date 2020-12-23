
插一句，如果一个`struct`类型变量，若是属性都是小写开头的话（private），那么`json.Marshal`方法是序列化不出来的。([Go的json解析：Marshal与Unmarshal](https://blog.csdn.net/zxy_666/article/details/80173288))


#### 建造者模式理念

1. 为什么需要建造者模式？

   建造者模式是构造方法的一种替代方案，目的是通过抽象一层工程方法来更规范的构建复杂对象。

   什么叫复杂的对象？按照`单一职责原则`来说，一个对象不能包含太多的职责，也就不能包含太多的`属性`与`行为方法`,那么在满足单一职责原则基础上，或者实际应用中由于各种情况，也许程序开发中不能遵守原则，有时候会有负责对象的产生：**不少的对象属性与行为方法，其中属性又是其他负责对象，常规构造对象过程过于臃肿，不便于拓展，那么就需要设计模式来重构**

   

   核心理念：`将复杂对象的表现层与构造过程剥离开来，使相同的构造过程能创建不同的表现层。`

   对象构建过程，本质就是赋予对象行为逻辑与设置属性的值，那么Builder建造者模式，在应用上来看，是将**对象的属性与创建分离，抽象对象创建过程，根据不同场景复用创建过程**。

   

   

2. 建造模式有哪些组成？

   核心重要的就是两部分：`产品实体属性与方法 + 抽象的构建过程`

   - 抽象建造者（builder）：描述具体建造者的公共接口，一般用来定义建造细节的方法，并不涉及具体的对象部件的创建。【**可复用**】

   - 产品（Product）：描述一个由一系列部件组成较为复杂的对象。

   - 具体建造者（ConcreteBuilder）：描述具体建造者，并实现抽象建造者公共接口。

   - 指挥者（Director）：调用具体建造者来创建复杂对象（产品）的各个部分，并按照一定顺序（流程）来建造复杂对象。

     

     

3. 建造者有哪些应用套路？

   - **为了分离对象的属性与创建过程**

     一旦将属性与对象创建过程接耦，那么对象的创建过程就不再强依赖所有的属性，创建过程就能更灵活的，具体建造者就可以根据实际的对象属性组合的创建不同的产品。

     

   - **相同的构造流程可以产出不同的表现层**

     因为对象构建过程是抽象的！那么有相同构建过程的产品，都可以复用改构建过程，从而可以组合创建不同的产品。

4. 建造者有哪些优势缺点？

   - 优点：

     1、建造者独立，易扩展。 2、便于控制细节风险。

   - 缺点：

     1、产品必须有共同点，范围有限制。 2、如内部变化复杂，会有很多的建造类。



#### 建造者实践应用

了解了设计模式建造者模式理念，那么就可以通过高级语言来实践应用。

###### Go应用

1. 定义建造者最终的产品实体Product(产品都有相同的构建过程)

   ```go
   type Order struct {
   	OrderId string
   	Price Money
   	UserId int64
   	GoodName string
   	OrderType string
   
   	// ...
   }
   
   // 外卖订单Product
   type FootOrder struct {
   	OrderId string
   	Price Money
   	UserId int64
   	GoodName string
   	OrderType string
   
   	// ...
   }
   ```

   

2. 抽象建造者Builder

   ```go
   	  // 抽象建造者Builder，描述具体建造者的公共接口，一般用来定义建造细节的方法
      type Builder interface {
      	WithOrderId(orderId string) Builder
      	WithPrice(p Money)  Builder
      	WithUid(uid int64)  Builder
      	WithGoodName(gn string)  Builder
      	WithType(otype string) Builder
      	Build() Builder
      }
   ```

   

3. 不同产品复用相同构建过程的实现者

   ```go
    		func (od Order) WithOrderId(oid string) Builder {
      		if len(oid) > 0 {
      		od.OrderId = oid
      	}
      		return od
     	 }
      	func (od Order) WithPrice(m Money) Builder {
      		od.Price = m
      		return od
      	}
      
     	 func (od Order) WithUid(uid int64) Builder {
      		od.UserId = uid
      		return od
      	}
      
      	func (od Order) WithGoodName(gn string) Builder {
      		od.GoodName = gn
      		return od
      	}
      
      	func (od Order) WithType(otype string) Builder {
      		od.OrderType = otype
      		return od
      	}
      
      	func (od Order) Build() Builder{
         // TODO 可以在创建的对象前置处理.....
      		return od
     	 }
      
      	// 产品2: 外卖订单Order的具体构建过程
      	func (fo FootOrder) WithOrderId(oid string) Builder {
      	if len(oid) > 0 {
      			fo.OrderId = oid
      	}
      		return fo
      	}
        func (fo FootOrder) WithPrice(m Money) Builder {
      		fo.Price = m
      		return fo
      	}
      
      	func (fo FootOrder) WithUid(uid int64) Builder {
      		fo.UserId = uid
      		return fo
      	}
      
      	func (fo FootOrder) WithGoodName(gn string) Builder {
      		fo.GoodName = gn
      		return fo
      	}
      
      	func (fo FootOrder) WithType(otype string) Builder {
      		fo.OrderType = otype
      		return fo
      	}
      
      	func (fo FootOrder) Build() Builder{
        	// TODO 可以在创建的对象前置处理.....
      		return fo
      	}
      
   ```

   

4. 构建产品构建指挥者Director

   ```go
   // 构造Director，用来具体构建对象
      func NewOrderBuilder() Builder {
      	return &Order{}
      }
      
      func NewFootOrderBuilder() Builder {
      	return &FootOrder{}
      }
   ```

   

5. 使用Usage

   ```go
    func Test_builder(t *testing.T) {
      		// 使用相同构建过程，创建支付订单对象
      		payOrder:= NewOrderBuilder().WithOrderId("EG001").
      		WithPrice(20.40).
      		WithUid(10010).
      		WithGoodName("商品").
      		WithType("支付订单").Build()
      
        // 使用相同构建过程，创建外卖单对象
      		footOrder:= NewFootOrderBuilder().WithOrderId("F001").
      		WithPrice(10.40).
      		WithUid(10010).
      		WithGoodName("酸奶").
      		WithType("商品订单").Build()
      
      		sorder,_ := json.Marshal(payOrder)
      		forder,_ := json.Marshal(footOrder)
      
      		//fmt.Println(reflect.TypeOf(order).Kind())
      		//fmt.Println(reflect.ValueOf(payOrder))
      		fmt.Printf(string(sorder))
      		fmt.Println("")
      		fmt.Printf(string(forder))
      }
   ```

   

6. 用插件自动生成的建造者模式代码

    改种方式是通过组合产品对象product与对应的建造者来创建对象。

   ```go
      // Order builder pattern code
      type OrderBuilder struct {
      		order *Order
      }
      
      func NewOrderBuilder() *OrderBuilder {
      		order := &Order{}
      		b := &OrderBuilder{order: order}
      		return b
      }
      
      func (b *OrderBuilder) OrderId(orderId string) *OrderBuilder {
      		b.order.OrderId = orderId
      		return b
      }
      
      func (b *OrderBuilder) Price(price float64) *OrderBuilder {
      		b.order.Price = price
      		return b
      }
      
      func (b *OrderBuilder) UserId(userId int64) *OrderBuilder {
      		b.order.UserId = userId
      		return b
      }
      
      func (b *OrderBuilder) GoodName(goodName string) *OrderBuilder {
      		b.order.GoodName = goodName
      		return b
      }
      
      func (b *OrderBuilder) OrderType(orderType string) *OrderBuilder {
      		b.order.OrderType = orderType
      		return b
      }
      
      func (b *OrderBuilder) Build() (*Order, error) {
      		return b.order, nil
      }
      
      // usage
      	payOrder:= NewOrderBuilder().WithOrderId("EG001").
      	WithPrice(20.40).
      	WithUid(10010).
      	WithGoodName("商品").
     	WithType("支付订单").Build()
   ```

   

---



###### Java应用

参考了该篇文章例子：[Java设计模式14：建造者模式](https://blog.csdn.net/weixin_38230747/article/details/88560118)



1. 具体复杂对象产品定义

   ```java
   public class Actor {
   	String type;
   	String sex;
   	String face;
   	String costume;
   	String hairstyle;
       //此处已省略getter合setter构建的方法
    
   }
   ```

   

2. 抽象对象建造者,定义抽象的产品创建过程

   ```java
   public abstract class ActorBuilder {
   	Actor actor = new Actor();
    
   	public abstract void buildType();
    
   	public abstract void buildSex();
    
   	public abstract void buildFace();
    
   	public abstract void buildCostume();
    
   	public abstract void buildHairstyle();
    
   	public Actor createActor() {
   		return actor;
   	};
   }
   ```

   

3. 不同产品具体建造者实现（实现具体建造者ConcreteBuilder）

   ```java
   	public class AngelBuilder extends ActorBuilder {
   		@Override
   		public void buildType() {
   			actor.setType("天使");
   		}
    
   		@Override
   		public void buildSex() {
   			actor.setSex("AngelSex");
   		}
    
   		@Override
   		public void buildFace() {
   			actor.setFace("AngelFace");
   		}
    
   		@Override
   		public void buildCostume() {
   			actor.setCostume("AngelCostume");
   		}
    
   		@Override
   		public void buildHairstyle() {
   			actor.setHairstyle("AngelHairStyle");
   		}
     
     	// 另外一种产品实现
     public class GhostBuilder extends ActorBuilder {
    
   		@Override
   		public void buildType() {
   			actor.setType("精灵");
   		}
    
   		@Override
   		public void buildSex() {
   			actor.setSex("GhostSex");
   		}
    
   		@Override
   		public void buildFace() {
   			actor.setFace("GhostFace");
   		}
    
   		@Override
   		public void buildCostume() {
   			actor.setCostume("GhostCostume");
   		}
    
   		@Override
   		public void buildHairstyle() {
   			actor.setHairstyle("GhostHairStyle");
   		}
    
   	}
   ```
   
   

4. 构建产品构建指挥者Director

   ```java
   public class Client {
   	public static void main(String[] args) {
   		ActorBuilder ab = new AngelBuilder();//无配置文件
   		ActorController ac = new ActorController();
   		Actor angel;
   		angel = ac.construct(ab);
   		System.out.println(angel.getType() + "的外观：");
   		System.out.println("性别：" + angel.getSex());
   		System.out.println("面容：" + angel.getFace());
   		System.out.println("服装：" + angel.getCostume());
   		System.out.println("发型：" + angel.getHairstyle());
   	}
   }
   ```

   


