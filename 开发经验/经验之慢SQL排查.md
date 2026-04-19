> 问题概述：sql监控发现慢sql，如下，问题在于merchant_id被拼接了cast函数，导致无法使用索引
```
select item0_.item_no as item_no1_17_, item0_.user_start_price as user_st36_17_, ... from items item0_ where cast(item0_.merchant_id as char)=108018 and item0_.is_deleted=? order by item0_.create_dt desc limit ?
```

### 解决方式

错误构建了CriteriaBuilder cb：`cb.equal(root.get("merchantId").as(String.class), itemSpecification.getMerchantId())`
正确：`cb.equal(root.get("merchantId").as(Long.class), itemSpecification.getMerchantId())`

解释：在构建CriteriaBuilder时，需要正确判断Root对象和Domain对象的数据类型，如果Domain对象中字段是Long类型的，则root需要使用对应的数据类型`root.get("merchantId").as(Long.class)`，否则CriteriaBuilder发现数据类型不一致，会使用`cast函数`去进行转换

### 为什么会多cast函数的拼接

源码分析：
看看`root.get("merchantId").as(Long.class)`中`as`做了什么操作
```java
public <X> Expression<X> as(Class<X> type) {
    return (Expression)(type.equals(this.getJavaType()) ? this : new CastFunction(this.criteriaBuilder(), type, this));
}
```
其实这一步就已经判断是否一致了，不一致就使用`CastFunction`做转换操作了。后面在渲染sql语句的时候会调用`CastFunction`方法：
```java
public String render(RenderingContext renderingContext) {
        return "cast(" + this.castSource.render(renderingContext) + " as " + renderingContext.getCastType(this.getJavaType()) + ')';
    }
```
