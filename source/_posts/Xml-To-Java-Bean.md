title: 利用注解、反射将Xml转JavaBean
categories: Java
date: 2015-09-22 10:50:28
tags: xml转javabean
description: xml转javabean 反射xml转javabean 自定义javabean转xml 非规则xml反射 注解xml
---

上一篇文章谈到了javabean转xml的问题，现在反过来，利用反射将xml转javabean。这里主要用到两个技术点，一个是利用dom4j根据xpath来查找节点，另一个就是java reflect方面的知识点。
- 关于xpath相关的知识点可以看一下[w3cschool](http://www.w3school.com.cn/xpath/)的教程
- 关于java reflect方面的知识可以简单地看一下官方文档，我的建议是直接找个例子做一下，不会再查api文档，用多了就熟了。

<!-- more -->
回到主题，继续上篇文章的内容，将xml转回javabean，下面看一下代码，然后看一下代码后面的重点讲解：

```
package com.qhee.trade.utils;

import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;
import java.util.ArrayList;
import java.util.Collection;
import java.util.HashSet;
import java.util.LinkedHashSet;
import java.util.LinkedList;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.TreeSet;
import java.util.Vector;
import java.util.concurrent.ConcurrentHashMap;

import org.apache.commons.lang.StringUtils;
import org.dom4j.Document;
import org.dom4j.DocumentHelper;
import org.dom4j.Node;

import com.qhee.trade.EtransException;

public class XmlUtils {
	/****************************************Xml转Bean***************************************/
	
	/**
	 * Xml转Bean
	 * @param type
	 * @param xml
	 * @return
	 * @throws Exception
	 */
	public static <T> T toBean(Class<? extends T> type, String xml) throws Exception{
		Document document = DocumentHelper.parseText(xml);
		T object = type.newInstance();
		Field[] fields = type.getDeclaredFields();
		for (int i = 0; i < fields.length; i++) {
			assembleObjectField(object, fields[i], document);
		}
		return object;
	}
	
	/**
	 * 装配Bean属性
	 * @param object
	 * @param field
	 * @param document
	 * @throws Exception
	 */
	private static <T> void assembleObjectField(T object,Field field,Document document) throws Exception{
		String className = field.getType().getName();
		if(Collection.class.isAssignableFrom(field.getType())){
			assembleCollection(object, field, document);
		}else if(className.startsWith("com.qhee")){
			assembleCustom(object, field, document);
		}else{
			assembleGeneric(object, field,document);
		}
	}
	
	/**
	 * 装配基本类型属性
	 * @param object
	 * @param field
	 * @param document
	 * @throws Exception
	 */
	private static <T> void assembleGeneric(T object,Field field,Document document) throws Exception{
		String fieldName = field.getAnnotation(com.qhee.etrans.annotation.Field.class).name();
		if(field.isAnnotationPresent(com.qhee.etrans.annotation.Head.class)){
			processValueForGeneric(object, field, document.selectSingleNode("/message/head/field[@name='"+fieldName+"']").getText());
		}else if(field.isAnnotationPresent(com.qhee.etrans.annotation.Body.class)){
			processValueForGeneric(object, field, document.selectSingleNode("/message/body/field[@name='"+fieldName+"']").getText());
		}
	}
	
	/**
	 * 装配自定义类属性
	 * @param object
	 * @param field
	 * @param document
	 * @throws Exception
	 */
	private static <T> void assembleCustom(T object,Field field,Document document) throws Exception{
		Field[] fields = field.getType().getDeclaredFields();
		Object customObject = field.getType().newInstance();
		String position = "head";
		if(field.isAnnotationPresent(com.qhee.etrans.annotation.Body.class)){
			position = "body";
		}
		for (Field customObjectField : fields) {
			String fieldName = customObjectField.getAnnotation(com.qhee.etrans.annotation.Field.class).name();
			processValueForGeneric(customObject, customObjectField, document.selectSingleNode("/message/"+position+"/field[@name='"+fieldName+"']").getText());
		}
		field.set(object, customObject);
	}
	
	/**
	 * 装配数组类型属性
	 * @param object
	 * @param field
	 * @param document
	 * @throws Exception
	 */
    @SuppressWarnings("unchecked")
	private static <T,E> void assembleCollection(T object,Field field,Document document) throws Exception{
		Type type = field.getGenericType();
		if(type instanceof ParameterizedType){
			Type[] types = ((ParameterizedType) type).getActualTypeArguments();
			Class<E> innerClazz = (Class<E>) types[0];
		    Collection<E> listObject = null;
			if(Collection.class.isAssignableFrom(field.getType())){
				if(List.class.isAssignableFrom(field.getType())){
					if(ArrayList.class.isAssignableFrom(field.getType())) listObject = new ArrayList<E> ();
					if(LinkedList.class.isAssignableFrom(field.getType())) listObject = new LinkedList<E> ();
					if(Vector.class.isAssignableFrom(field.getType())) listObject = new Vector<E> ();
					if(listObject == null) listObject = new ArrayList<E> ();
				}else if(Set.class.isAssignableFrom(field.getType())){
					if(HashSet.class.isAssignableFrom(field.getType())) listObject = new HashSet<E> ();
					if(LinkedHashSet.class.isAssignableFrom(field.getType())) listObject = new LinkedHashSet<E> ();
					if(TreeSet.class.isAssignableFrom(field.getType()))listObject = new TreeSet<E> ();
					if(listObject == null) listObject = new LinkedHashSet<E> ();
				}
			}
            assembleListElements(object,listObject, field,innerClazz, document);
		}
	}
	
    /**
     * 装配数组元素
     * @param listObject
     * @param field
     * @param innerClazz
     * @param document
     * @throws Exception
     */
    @SuppressWarnings("unchecked")
	private static <E,T> void assembleListElements(T object,Collection<E> listObject,Field field,Class<E> innerClazz,Document document) throws Exception{
        Field[] innerFields = innerClazz.getDeclaredFields();
        List<Node> list = null;
		if(field.isAnnotationPresent(com.qhee.etrans.annotation.Head.class)){
			list = document.selectNodes("/message/head/field-list[@name='"+field.getAnnotation(com.qhee.etrans.annotation.Field.class).name()+"']/field-list");
		}else if(field.isAnnotationPresent(com.qhee.etrans.annotation.Body.class)){
			list = document.selectNodes("/message/body/field-list[@name='"+field.getAnnotation(com.qhee.etrans.annotation.Field.class).name()+"']/field-list");
		}
		if(list != null){
			for (Node node : list) {
			  //E innerObject =  innerClazz.newInstance(); //该句代码会发生异常，见下方解析
			    Constructor<?>[] c = innerClazz.getDeclaredConstructors();
			    E innerObject = (E)c[0].newInstance(object);
				for (int i = 0; i < innerFields.length; i++) {
					Field innerField = innerFields[i];
					if(innerField.isAnnotationPresent(com.qhee.etrans.annotation.Field.class)){//过滤掉this$0属性
						processValueForGeneric(innerObject, innerField,  node.selectSingleNode("field[@name='"+innerField.getAnnotation(com.qhee.etrans.annotation.Field.class).name()+"']").getText());
					}
				}
				listObject.add(innerObject);
			}
		}
		field.set(object, listObject);
	}
	
	/**
	 * 基本类型转换
	 * @param entity
	 * @param field
	 * @param value
	 * @throws Exception
	 */
	private static void processValueForGeneric(Object entity,java.lang.reflect.Field field,Object value) throws Exception{
		if(field == null) return ;
		boolean access=field.isAccessible();
		try{
			if(String.class.isAssignableFrom(field.getType())){
				field.set(entity, value);
			}else 
				if(Integer.class.isAssignableFrom(field.getType()) ){
					field.set(entity, value == null ? null:Integer.valueOf(value.toString()));
				}else if(field.getType() == int.class){
					field.set(entity, value == null ? 0:Integer.valueOf(value.toString()));
				}
				
				else if(Long.class.isAssignableFrom(field.getType()) ){
					field.set(entity, value == null ? null:Long.valueOf(value.toString()));
				}else if(field.getType() == long.class){
					field.set(entity, value == null ? 0L:Long.valueOf(value.toString()));
				}
				
				
				else if(Float.class.isAssignableFrom(field.getType())){
					field.set(entity, value == null ? null:Float.valueOf(value.toString()));
				}else if(field.getType() == float.class){
					field.set(entity, value == null ? 0F:Float.valueOf(value.toString()));
				}
				
				else if(Double.class.isAssignableFrom(field.getType()) ){
					field.set(entity, value == null ? null:Double.valueOf(value.toString()));
				}else if(field.getType() == double.class){
					field.set(entity, value == null ? 0D:Double.valueOf(value.toString()));
				}
			
			
				else if(Byte.class.isAssignableFrom(field.getType())) {	
					field.set(entity, value == null ? null:Byte.valueOf(value.toString()));
				}else if(field.getType() == byte.class){
					field.set(entity, value == null ? (byte)0:Byte.valueOf(value.toString()));
				}
				
				else if(Boolean.class.isAssignableFrom(field.getType())){
					field.set(entity, value == null ? null:Boolean.valueOf(value.toString()));
				}else if(field.getType() == boolean.class){
					field.set(entity, value == null ? false:Boolean.valueOf(value.toString()));
				}
		}catch(Exception ex){
			ex.printStackTrace();
		}finally{
			field.setAccessible(access);
		}
		//日期的bean属性都用字符串表示
	}
}

```

都是一些api的运用，代码可能不太完善仅供参考一下，下面说一下注意的地方和重点的代码段：

- 关于内部类反射实例化的问题：

上面的代码我注释掉了这句代码

```
//E innerObject =  innerClazz.newInstance(object);
```

因为这段代码会发生下面的异常

```
Exception in thread "main" java.lang.InstantiationException: com.qhee.etrans.entity.CCBBalanceResponse$CCBBalanceItem
...
Caused by: java.lang.NoSuchMethodException: com.qhee.etrans.entity.CCBBalanceResponse$CCBBalanceItem.<init>()
```

异常原因说的很清楚，是因为没有这样的初始化方法，我们可以用下面的代码看一下内部类里面就是有什么样的初始化方法

```
Constructor<?>[] c = innerClazz.getDeclaredConstructors();
```

当我们用断点看一下这个构造方法数组，你会发生里面只有一个有参的构造方法，参数就外部类的实例，因此我们用代码的代码来实例化内部类就不会报错了

```
 Constructor<?>[] c = innerClazz.getDeclaredConstructors();
 E innerObject = (E)c[0].newInstance(object);
```

- 关于内部类this$0属性的问题

获取内部类的属性代码

```
Field[] innerFields = innerClazz.getDeclaredFields();
```

当我尝试用上面的方法获取内部类的属性，并for循环反射设置属性的值的时候，发现多了一个this$0的属性，反射设置值的时候就报了异常，这是一个指向外部类实例的属性。因此在代码中需要将此属性过滤掉以免发生异常

```
if(innerField.isAnnotationPresent(com.qhee.etrans.annotation.Field.class)){//过滤掉this$0属性
	...
}
```

- 关于获取集合里面的泛型实际类型的问题

```
Type type = field.getGenericType();
if(type instanceof ParameterizedType){
	Type[] types = ((ParameterizedType) type).getActualTypeArguments();
	Class<E> innerClazz = (Class<E>) types[0];
}
```

嗯，运用上面的代码就可以获取到集合泛型的实际类型了。

原创文章，欢迎转载，无需注明出处。
