title: 利用注解、反射将JavaBean转Xml
categories: Java
date: 2015-09-14 14:46:45
tags: JavaBean转Xml
description: JavaBean转Xml 自定义javabean转xml 非规则xml反射 注解xml
---

最近对接建行商贸通项目，遇到一个JavaBean和Xml互转的问题。大家通常用的的Bean转Xml应该都是直接将Bean的属性名作为Xml的标签，属性值作为Xml的标签值。比如微信公众平台的消息：
```
 <xml>
 <ToUserName><![CDATA[toUser]]></ToUserName>
 <FromUserName><![CDATA[fromUser]]></FromUserName> 
 <CreateTime>1348831860</CreateTime>
 <MsgType><![CDATA[text]]></MsgType>
 <Content><![CDATA[this is a test]]></Content>
 <MsgId>1234567890123456</MsgId>
 </xml>
```

在这种情况下已经有一些现成的工具可以转化，比如说`org.simpleframework.xml.core.Persister.Persister()`，示例如下：

<!-- more -->

```
Serializer  serializer = new Persister();
ByteArrayOutputStream out = new ByteArrayOutputStream();
try {
	    serializer.write(o, out);
}
catch (Exception e) {
    logger.error(e.getMessage(), e);
}
 
String xmlData;
try {
	xmlData = out.toString("GB2312");
} 
catch (UnsupportedEncodingException e) {
		logger.error(e.getMessage(), e);
}
return xmlData;
```

而这次项目遇到的情况是下面这样子的：

```
<?xml version="1.0" encoding="gb2312"?>
<message>
    <head>
		<field pin="false" name="version"><![CDATA[100]]></field>
		<field pin="false" name="type"><![CDATA[0210]]></field>
		<field pin="false" name="chanl_no"><![CDATA[30]]></field>
		<field pin="false" name="chanl_sub_no"><![CDATA[3001]]></field>
		<field pin="false" name="chanl_date"><![CDATA[20110227]]></field>
		<field pin="false" name="chanl_time"><![CDATA[083941]]></field>
		<field pin="false" name="ectip_date"><![CDATA[20110227]]></field>
    </head>
    <body>
		<field pin="false" name="MCH_NO"><![CDATA[1100000001]]></field>
		<field pin="false" name="MCH_NAME"><![CDATA[贵金属交易所]]></field>
		<field-list name="ARRAY_3FC006">
	        <field-list name="0">
	            <field pin="false" name="ACCT_NAME62"><![CDATA[测试公司]]></field>
	            <field pin="false" name="SIT_NO"><![CDATA[101987654321002]]></field>
	            <field pin="false" name="CURR_COD"><![CDATA[01]]></field>
	            <field pin="false" name="CURR_IDEN"><![CDATA[0]]></field>
	            <field pin="false" name="ACCT_BAL">14281.91</field>
	            <field pin="false" name="AVL_BAL">14281.91</field>
	        </field-list>
		</field-list>
    </body>
</message>
```

这情况不是直接用JavaBean的属性值作为标签，而是用标签的属性名标识Bean的属性名，网上没找到现有的轮子，只好自己利用注解、反射来写一个（有朋友找到的话请告诉我）。


## 来简单地看一下代码，不是十分的完善，目前仅支持反射到第二层属性，以后有时间再慢慢完善：

### 先自定义几个注解，比较简单就不细说了

- head标签的注解

```
@Target({FIELD})
@Retention(RUNTIME)
public @interface Head {
	String text() default ""; //注释文本
}
```

- body标签的注解

```
@Target({FIELD})
@Retention(RUNTIME)
public @interface Body {
	String text() default ""; //注释文本
}
```

- field标签的注解

```
@Target({FIELD})
@Retention(RUNTIME)
public @interface Field {
	String name() default "";
	boolean pin() default false;
	boolean required() default false;
}
```

### 新建一个测试JavaBean

```
public class CCBBalanceResponse {
	@Head
	public CCBCommonHead header;
	@Body
	@Field(name = "MCH_NO")
	public String mchNo;
	@Body
	@Field(name = "MCH_NAME")
	public String mchName;
	@Body
	@Field(name="ARRAY_3FC006")
	public List<CCBBalanceItem> items;

	public class CCBBalanceItem {
		@Field(name="ACCT_NAME62")
		public String accName;
		@Field(name="SIT_NO")
		public String sitNo;
		@Field(name="CURR_COD")
		public String currCod;
		@Field(name="CURR_IDEN")
		public String currIden;
		@Field(name="ACCT_BAL")
		public double accBal;
		@Field(name="AVL_BAL")
		public double avlBal;
		@Field(name="STX_SVC_FRZ_BAL")
		public double stxSvcFrzBal;
		@Field(name="STX_GUAR_FRZ_BAL")
		public double stxGuarFrzBal;
		@Field(name="SUPP_GUAR_FRZ_BAL")
		public double suppGuarFrzBal;
		@Field(name="OTH_GUAR_FRZ_BAL")
		public double othGuarFrzBal;
	}
}


```

### 转换工具类，不懂的地方查下api应该就可以了

```
package com.qhee.trade.utils;

import java.lang.reflect.Field;
import java.util.Collection;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

import org.apache.commons.lang.StringUtils;

import com.qhee.trade.EtransException;

public class XmlUtils {
	private static final String NEW_LINE="\n";
	private static Map<Class<?>,Field[]> FIELD_CACHE = new ConcurrentHashMap<Class<?>,Field[]>();
	
	/****************************************Bean转Xml***************************************/
	
	/**
	 * Bean转Xml
	 * @param object
	 * @return
	 */
	public static String toXml(Object object) throws Exception{
		if(object == null){
			return "";
		}
		Field[] fields = object.getClass().getDeclaredFields();
		String message = "";
		try {
			message = buildMessageContent(object, fields);
		} catch (Exception e) {
			throw new EtransException(e);
		}
		StringBuilder sb = new StringBuilder();
		sb.append("<?xml version=\"1.0\" encoding=\"gb2312\"?>\n");
		sb.append("<message>\n");
		sb.append(message);
		sb.append("</message>");
		return sb.toString();
	}
	
	/**
	 * 构建message内容
	 * @param object
	 * @param fields
	 * @return
	 * @throws Exception
	 */
	private static String buildMessageContent(Object object,Field[] fields) throws Exception{
		StringBuilder headBuilder = new StringBuilder("    <head>\n");
		StringBuilder bodyBuilder = new StringBuilder("    <body>\n");
		for (Field field : fields) {
			if(field.isAnnotationPresent(com.qhee.etrans.annotation.Head.class)){
				buildField(object, field, headBuilder);
			}else if(field.isAnnotationPresent(com.qhee.etrans.annotation.Body.class)){
				buildField(object, field, bodyBuilder);
			}
		}
		headBuilder.append("    </head>\n");
		bodyBuilder.append("    </body>\n");
		return headBuilder.toString()+bodyBuilder.toString();
	}
	
	/**
	 * 构建field
	 * @param object
	 * @param field
	 * @param sb
	 * @throws Exception
	 */
	private static void buildField(Object object,Field field,StringBuilder sb) throws Exception{
		String className = field.getType().getName();
		if(Collection.class.isAssignableFrom(field.getType())){
			buildListField(object, field, sb);
		}else if(className.startsWith("com.qhee")){//此处是自己的自定义类，需修改成自己自定义类的包前缀
			Object customObject = field.get(object);
			if(customObject != null){
				Field[] fields = customObject.getClass().getDeclaredFields();
				for (Field field2 : fields) {
					buildField(customObject, field2, sb);
				}
			}
		}else{
			buildGenericField(object, field, sb);
		}
	}

	/**
	 * 构建feild标签内容
	 * @param sb
	 * @param field
	 * @param object
	 * @throws Exception
	 */
	private static void buildGenericField(Object object,Field field,StringBuilder sb) throws Exception{
		com.qhee.etrans.annotation.Field annoField = field.getAnnotation(com.qhee.etrans.annotation.Field.class);
		if(annoField != null){
			Object value = field.get(object);
			String fieldValue = value == null ? "" : StringUtils.trimToEmpty(value.toString());
			
			sb.append("        <field pin=\"");
			sb.append(annoField.pin());
			sb.append("\" name=\"");
			sb.append(annoField.name());
			sb.append("\">");
			sb.append(fieldValue);
			sb.append("</field>");
			sb.append(NEW_LINE);
		}
	}
	
	/**
	 * 构建field-list标签
	 * @param sb
	 * @param field
	 * @param object
	 * @throws Exception
	 */
	private static void buildListField(Object object,Field field,StringBuilder sb) throws Exception{
		com.qhee.etrans.annotation.Field annoField = field.getAnnotation(com.qhee.etrans.annotation.Field.class);
		if(annoField != null){
			sb.append("        <field-list name=\"");
			sb.append(annoField.name());
			sb.append("\">");
			sb.append(NEW_LINE);
			
			Object listObject = field.get(object);
			sb.append(bulidListContent(listObject));
			
			sb.append("        </field-list>");
			sb.append(NEW_LINE);
		}
	}
	
	/**
	 * 构建field-list标签内容
	 * @param listObject
	 * @return
	 * @throws Exception
	 */
	private static String bulidListContent(Object listObject) throws Exception{
		if(listObject == null){
			return "";
		}
		StringBuilder wrapper = new StringBuilder();
		Collection<?> collection = (Collection<?>) listObject;
		Object[] objects = collection.toArray();
		for (int i = 0; i < objects.length; i++) {
			wrapper.append("            <field-list name=\"");
			wrapper.append(i);
			wrapper.append("\">");
			wrapper.append(NEW_LINE);
			wrapper.append(buildFields(objects[i]));
			wrapper.append("            </field-list>");
			wrapper.append(NEW_LINE);
		}
		return wrapper.toString();
	}
	
	/**
	 * 构建field-list标签内容field
	 * @param object
	 * @return
	 * @throws Exception
	 */
	private static String buildFields(Object object) throws Exception{
		if(object == null){
			return "";
		}
		StringBuilder sb = new StringBuilder();
		Field[] fields = FIELD_CACHE.get(object.getClass());
		if(fields == null){
			fields = object.getClass().getDeclaredFields();
			FIELD_CACHE.put(object.getClass(), fields);
		}
		for (Field field : fields) {
			com.qhee.etrans.annotation.Field annoField = field.getAnnotation(com.qhee.etrans.annotation.Field.class);
			if(annoField != null){
				Object value = field.get(object);
				String fieldValue = value == null ? "" : StringUtils.trimToEmpty(value.toString());
				
				sb.append("                <field pin=\"");
				sb.append(annoField.pin());
				sb.append("\" name=\"");
				sb.append(annoField.name());
				sb.append("\">");
				sb.append(fieldValue);
				sb.append("</field>");
				sb.append(NEW_LINE);
			}
		}
		return sb.toString();
	}
	
}

```

### OK,下面随便写个main函数来测试一下

```
public static void main(String[] args) throws Exception{
	CCBCommonHead head = new CCBCommonHead();
	head.version = "100";
	head.type = "0200";
	head.chanlNo = "30";
	head.chanlSubNo = "3001";
	head.chanlDate = "20150919";
	head.chanlTime = "133030";
	head.chanlFlowNo = "11111";
	head.chanlTradNo = "111";
	
	CCBBalanceResponse br = new CCBBalanceResponse();
	br.header = head;
	br.mchNo = "1111";
	br.mchName = "ssss";
	
	List<CCBBalanceItem> items = new ArrayList<CCBBalanceResponse.CCBBalanceItem>();
	CCBBalanceItem item1 = br.new CCBBalanceItem();
	item1.accName = "111";
	item1.accName = "111";
	items.add(item1);
	
	CCBBalanceItem item2 = br.new CCBBalanceItem();
	item2.accName = "222";
	item2.accName = "222";
	items.add(item2);
	
	br.items = items;
	
	System.out.println(XmlUtils.toXml(br));
}
```

### 测试输出结果

```
<?xml version="1.0" encoding="gb2312"?>
<message>
    <head>
        <field pin="false" name="version">100</field>
        <field pin="false" name="type">0200</field>
        <field pin="false" name="chanl_no">30</field>
        <field pin="false" name="chanl_sub_no">3001</field>
        <field pin="false" name="chanl_date">20150919</field>
        <field pin="false" name="chanl_time">133030</field>
        <field pin="false" name="ectip_date"></field>
        <field pin="false" name="chanl_flow_no">11111</field>
        <field pin="false" name="ectip_flow_no"></field>
        <field pin="false" name="chanl_trad_no">111</field>
        <field pin="false" name="term_inf"></field>
        <field pin="false" name="resp_code"></field>
        <field pin="false" name="resp_msg"></field>
        <field pin="false" name="page"></field>
        <field pin="false" name="maxrow"></field>
        <field pin="false" name="locstr"></field>
        <field pin="false" name="tot_rec"></field>
    </head>
    <body>
        <field pin="false" name="MCH_NO">1111</field>
        <field pin="false" name="MCH_NAME">ssss</field>
        <field-list name="ARRAY_3FC006">
            <field-list name="0">
                <field pin="false" name="ACCT_NAME62">111</field>
                <field pin="false" name="SIT_NO"></field>
                <field pin="false" name="CURR_COD"></field>
                <field pin="false" name="CURR_IDEN"></field>
                <field pin="false" name="ACCT_BAL">0.0</field>
                <field pin="false" name="AVL_BAL">0.0</field>
                <field pin="false" name="STX_SVC_FRZ_BAL">0.0</field>
                <field pin="false" name="STX_GUAR_FRZ_BAL">0.0</field>
                <field pin="false" name="SUPP_GUAR_FRZ_BAL">0.0</field>
                <field pin="false" name="OTH_GUAR_FRZ_BAL">0.0</field>
            </field-list>
            <field-list name="1">
                <field pin="false" name="ACCT_NAME62">222</field>
                <field pin="false" name="SIT_NO"></field>
                <field pin="false" name="CURR_COD"></field>
                <field pin="false" name="CURR_IDEN"></field>
                <field pin="false" name="ACCT_BAL">0.0</field>
                <field pin="false" name="AVL_BAL">0.0</field>
                <field pin="false" name="STX_SVC_FRZ_BAL">0.0</field>
                <field pin="false" name="STX_GUAR_FRZ_BAL">0.0</field>
                <field pin="false" name="SUPP_GUAR_FRZ_BAL">0.0</field>
                <field pin="false" name="OTH_GUAR_FRZ_BAL">0.0</field>
            </field-list>
        </field-list>
    </body>
</message>
```

原创文章，欢迎转载，无需注明出处。
