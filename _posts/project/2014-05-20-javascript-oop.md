---
layout: post
title : 	javascript面向对象编程浅析
description : javascript的面向对象编程，包括构造函数、原型、继承、成员属性、成员方法等基础知识。
category : project
tags : [javascript, oop, prototype, 继承, 封装, 对象, 函数, 原型]
---


###什么是面向对象
面向对象最基本的三大特性是继承、封装、多态，javascript不是面向对象的语言，但能实现继承和封装两个特性。
js中的对象

###弱类型

定义变量类型：

布尔型bool、数值型、字符串类型、函数类型、数组类型、空类型、未定义类型
	
	var a = ""; //字符串
	var b = []; //数组
	var c = function(){} //function函数
	var d = {}; //对象
	

字面量:

字面量就类似于我们使用的json数据格式，分为字符串字面量、数组字面量、函数字面量、对象字面量。
	
	var cat = {
		//字符串字面量
		name : "cat",
		
		//数组字面量
		type : ["波斯猫", "加菲猫", "垂耳猫"],
		
		//函数字面量
		mark : function(){alert(this.name + this.type)},
		
		//对象字面量
		info : {from : "usa", "age" : "1"}
	}


###创建js对象
js对象只是一组名称/值对，可以使用熟悉的“.”（点）运算符或“[]”运算符，来获得和设置对象的属性，像词典。

code1：

	var userObject = new Object();
	userObject.lastLoginTime = new Date();
	alert(userObject.lastLoginTime); 
	
code2：

	var userObject = {}; // equivalent to new Object()
	userObject["lastLoginTime"] = new Date();
	alert(userObject["lastLoginTime"]);   
	
code3：

	var userObject = { 
		"lastLoginTime" : new Date() 
	};
	alert(userObject.lastLoginTime);
	
###创建js函数，函数也是对象

code：
	
	//1、普通函数
	function func(x) {
	    alert(x);
	}
	func("blah");
	
	//2、定义函数，在此创建函数对象，并赋给变量func
	var func = function(x) {
	    alert(x);
	};
	func("blah2");
	
	//3、用Funciton构造函数，不常用
	var func = new Function("x", "alert(x);");
	func("blah3");	
	

###使用对象初始化器创建对象
code1：

	var myConstructor = function(){
	}
	
	//添加静态属性
	//name属性和alertName()方法作为静态成员添加到了对象实例中
	myConstructor.name = "heiniu";
	myConstructor.alertName = function(){
		alert(this.name);
	}
	//执行,不需要new
	myConstructor.alertName();

code2：等价于以下代码，结构更简洁清晰。
	
	//name属性和alertName()方法作为静态成员添加到了对象实例中
	var myConstructor = {
		//静态属性
		name : "heiniu",
		
		//静态方法
		alertName : function(){
			alert(this.name);
		}
	}
	
	//执行,不需要new
	myConstructor.alertName();
**总结：**

- 优点：简洁明了；
- 缺点：创建对象的代码是一次性的

###使用构造函数创建对象

定义构造函数，而不是类

	//私有成员就是在构造函数中定义的变量和函数
	function myConstructor2(msg, name){
	
		//公有属性
		this.myMsg = msg;
		this.name = name;
		
		//私有属性
		var myVersion = "1.0" 
		
		//私有方法
		function alertMsg(){
			alert(this.myMsg)
			alert(myVersion)
		}
		alertVersion();//实例化时显示信息
		
		//特权方法，也是公用方法，在构造函数的作用域中使用this关键字定义的方法，尽量不要用，只用于需要访问私有成员的情况。
		this.appendToMsg = fucntion(string){
			this.myMsg += "heiniu_" + string;
			alertMsg();
		}
		
	}
	
	//静态属性和方法，静态成员是直接通过类对象访问的。
	myConstructor2.myYear = "2012";
	myConstructor2.now = function(){
		return new Date();
	}
	
	//公有方法，修改函数原型，即prototype属性。
	//一旦修改原型方法则立即应用到继承的对象和实例中，有风险。
	//原型方式会将新方法添加到myConstructor2的底层定义中，而不是myConstructor2实例自身。
	myConstructor2.prototype.alertMsg(){
		alert(this.myMsg);
	}
	myConstructor2.prototype.alertName(){
		alert(this.name);
	}
	
	var myObj = new myConstructor2("hello");

new操作符等价于

	//1、call()方法,每个函数对象都有一个名为 call 的方法，它将函数作为第一个参数的方法进行调用。第一个参数用作 this 的对象。其他参数都直接传递给函数自身。
	var myObj = {};
	myConstructor2.call(myObj,"hello", "heiniu")
	
	//2、apply()方法，有两个参数，用作 this 的对象和要传递给函数的参数的数组。
	var myObj = {};
	myConstructor2.apply(myObj,["hello", "heiniu"])
	
call的工作机制：

	var someuser = {
		name : "sjm",
		display : function(words){
			console.log(this.name + ' says ' + words);
		}
	};
	
	var myself = {
		name : "heiniu"
	};
	
	someuser.display.call(myself, 'fighting!');
	//结果：heiniu says fighting!
	
- someuser.display是函数的引用，即被调用的函数。
- myself是someuser.display被调用时的上下文对象。
- 'fighting!'是传入someuser.display的参数。
- 通过call将上下文对象改变为myself对象。
	
**总结：**

作为对象，函数还可以赋给变量、作为参数传递给其他函数、作为其他函数的值返回，并可以作为对象的属性或数组的元素进行存储等等。

call和apply的功能一致：以不同的对象作为上下文来调用某个函数，即允许一个对象去调用另一个对象的成员函数。

call和apply的差别：call以参数表来接受被调用函数的参数，而apply以数组来接受被调用函数的参数。

使用不同的引用来调用同一个函数时，this指针永远是这个引用所属的对象。

- 优点：可创建出多个规划好的对象，有若干个固定的属性、方法，并能初始化实例化。
- 缺点：复杂些，有上下文对象（this指针），即被调用函数所处的环境。
	
###使用原型和构造函数共同生成对象
什么是原型链？

继承方面,javascript中的每个对象都有一个内部私有的链接指向另一个对象 (或者为 null),这个对象就是原对象的原型. 这个原型也有自己的原型, 直到对象的原型为null为止. 这种一级一级的链结构就称为原型链.

prototype 对象的任何属性和方法都被传递给那个类的所有实例。原型链利用这种功能来实现继承机制。

	function ClassA() {
	}
	
	ClassA.prototype.color = "blue";
	ClassA.prototype.sayColor = function () {
	    alert(this.color);
	};
	
	function ClassB() {
	}
	
	//继承和扩展
	ClassB.prototype = new ClassA();
	ClassB.prototype.name = "";
	ClassB.prototype.sayName = function () {
	    alert(this.name);
	};
	
	//执行
	var objA = new ClassA();
	var objB = new ClassB();
	objA.color = "blue";
	objB.color = "red";
	objB.name = "John";
	objA.sayColor();
	objB.sayColor();
	objB.sayName();

使用原型与直接在构造函数内定义的属性 不同点：

- 构造函数内定义的属性继承方式与原型不同，子对象需要显示调用父对象才能继承构造函数内定义的属性。
- 构造函数内定义的任何属性，包括函数在内都会被重复创建，同一个构造函数产生的两个对象不共享实例。
- 构造函数内定义的函数有运行时闭包的开销，因为构造函数内的局部变量对其中定义的函数来说是可见的。

原型使用场合：

- 除非必须用构造函数闭包，否则尽量用原型定义成员函数，可减少开销
- 尽量在构造函数内定义一般成员，尤其是对象或数组，因为用原型定义的成员是多个实例共享的。

###原型链机制：

javascript分为三类对象：

- 1、用户创建的对象
- 2、构造函数对象
- 3、原型对象

code：

	function Foo(){
	}
	var foo = new Foo();
	var obj = new Object();


###公有、私有、特权、静态总结

- 私有和特权成员在函数的内部，他们会被带到函数的每一个实例中生成新的副本，因而将占用大量的内存。
 
- 公有的原型成员是对象的一部分，适用于通过new关键字实例化该对象的每一个实例。
 
- 静态成员所关联的是类本身，而不同的是大多数方法和属性所关联的是类的实例。

- 私有成员命名规范上用下划线表示，私有成员可避免安全隐患，防止使用者修改某个属性，导致对象内部数据的一致性受到破坏。如`_myPrivateProp`


###参考
- [http://msdn.microsoft.com/zh-cn/magazine/cc163419.aspx](http://msdn.microsoft.com/zh-cn/magazine/cc163419.aspx)
- [http://www.w3school.com.cn/js/pro_js_inheritance_implementing.asp](http://www.w3school.com.cn/js/pro_js_inheritance_implementing.asp)
