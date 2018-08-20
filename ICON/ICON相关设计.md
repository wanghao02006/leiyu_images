## Redis存储设计
### 1.版本对比key
格式(String)：
- Key：固定前缀:数据库名:表名:fil-业务主键。如：las_im_data_sync:db1:table11:fil-10001
- Value: 数据变更时间
- 有效期： 30天
### 2.更新专有Key
格式（String）：
- Key：固定前缀:数据库名:表名:kid-业务主键。如：las_im_data_sync:db1:table1:kid-10001
- Value:对应查询Key数组序列化信息
### 3.打标查询存储
格式（hash)
- Key:固定前缀:数据库名:表名:条件1:条件2:...如：las_im_data_sync:db1:table1:las-72:cat-870:bra-3987
- Value：{业务主键:对象属相}，如：{10001:{"id":1,"provinceId":1,"cityId":72,"lastAddrId":72,"categoryId":870}}
## 数据同步过程
1. 接收报文
2. 根据时间戳过滤是否需要更新数据
3. 判断是否是逻辑删除，如果是，标记为逻辑删除
```java
boolean logicDelete = false;
for (ChangeOfColumn column : changeOfRow.getAfterChangeOfColumns()) {
	if (column.getName().equals("yn")) {
		if (column.getValue() == null || column.getValue().equals("0")) {
			logicDelete = true;
			break;
		}
	}
}
```
4. 通过反射将报文转为操作实体
```java
protected <T>  T createRoObjectbyChangeRow(Class<T> cls,ChangeOfRow changeOfRow){
	if (logger.isDebugEnabled()){
		logger.debug("start to create {} object",cls.getName());
	}
	T instanceObj = null;
	try {
		instanceObj = cls.newInstance();
		Field[] fields = cls.getDeclaredFields();
		for (Field field:fields){
			ImRoColumn roAnnotation = field.getAnnotation(ImRoColumn.class);
			inner:
			for (ChangeOfColumn column:changeOfRow.getAfterChangeOfColumns()){
				boolean isThisField = false;
				if (roAnnotation!=null && roAnnotation.name()!=null && roAnnotation.name().equals(column.getName())){//如果注解的name属性与数据库列名相同
					isThisField = true;
				} else if (field.getName().equals(column.getName())){
					isThisField = true;
				}

				if (isThisField){
					setFieldFromString(instanceObj,field,column);
					break inner;
				}
			}
		}
	} catch (Exception e) {
		logger.error("JimdbSyncAbstract!createRoObjectbyChangeRow->ERROR:",e);
		throw new RuntimeException("JimdbSyncAbstract!createRoObjectbyChangeRow has Exception",e);
	}
	if (logger.isDebugEnabled()){
		logger.debug("{} object has be created,value is {}", JsonUtil.toJsonByGoogle(instanceObj));
	}
	return instanceObj;
}


private void setFieldFromString(Object instanceObj,Field field,ChangeOfColumn column) throws Exception{
	if (column.isNull()){
		logger.debug("{} need not set value, column value is null",field.getName());
		return;
	}
	String value = column.getValue();
	field.setAccessible(true);
	if (field.getType() == String.class){
		field.set(instanceObj,value);
	}else{
		if (field.getType()==byte.class){
			field.setByte(instanceObj,Byte.parseByte(value));

		}else if (field.getType()==char.class){
			field.setChar(instanceObj,value.charAt(0));

		}else if (field.getType()==short.class){
			field.setShort(instanceObj,Short.parseShort(value));

		}else if (field.getType()==int.class){
			field.setInt(instanceObj,Integer.parseInt(value));

		}else if (field.getType()==long.class){
			field.setLong(instanceObj,Long.parseLong(value));

		}else if (field.getType()==float.class){
			field.setFloat(instanceObj,Float.parseFloat(value));

		}else if (field.getType()==long.class){
			field.setLong(instanceObj,Long.parseLong(value));

		}else if (field.getType()==double.class){
			field.setDouble(instanceObj,Double.parseDouble(value));

		}else if (field.getType()==Byte.class){
			field.set(instanceObj,Byte.parseByte(value));

		}else if (field.getType()==Character.class){
			field.set(instanceObj,value.charAt(0));

		}else if (field.getType()==Short.class){
			field.set(instanceObj,Short.parseShort(value));

		}else if (field.getType()==Integer.class){
			field.set(instanceObj,Integer.parseInt(value));

		}else if (field.getType()==Long.class){
			field.set(instanceObj,Long.parseLong(value));

		}else if (field.getType()==Float.class){
			field.set(instanceObj,Float.parseFloat(value));

		}else if (field.getType()==Long.class){
			field.set(instanceObj,Long.parseLong(value));

		}else if (field.getType()==Double.class){
			field.set(instanceObj,Double.parseDouble(value));

		}else{
			throw new IllegalArgumentException("unspport class type of "+field.getType());
		}
	}

}
```
5. 根据业务唯一主键，获取需要更新（删除）的Redis Key。
```java
String relationKeyId = infoRo.getRoRelationKeyId(this.property.getJimdbKeyPrefix(),id);//根据业务主键生成
String[] roRelationIdKeys = infoRo.getRoRelationIdKeys(this.property.getJimdbKeyPrefix());
Set<String> needDeleteOnDeleteActionKeys = new HashSet<>();//删除操作时要删除的key
Set<String> needDeleteOnUpdateActionKeys = new HashSet<>();//更新操作时，要删除的key，redis中保存的key多余要更新的key

for (String roRelationId:roRelationIdKeys){//默认删除业务唯一主键对应所有key
	needDeleteOnDeleteActionKeys.add(roRelationId);
}

byte[] needDeleteKeysByte = this.property.getCluster().get(RoSerializer.serialize(relationKeyId));
if (needDeleteKeysByte!=null && needDeleteKeysByte.length>0){
	String[] existRoRelationIdKeys = (String[])(RoSerializer.deSerialize(needDeleteKeysByte));//根据业务主键获取业务key
	if (existRoRelationIdKeys!=null){
		for (String existKey:existRoRelationIdKeys){
			needDeleteOnDeleteActionKeys.add(existKey);//删除操作时删除该业务主键对应所有业务key数据
			boolean notExist = true;
			for (String ridKey:roRelationIdKeys){
				if (ridKey.equals(existKey)){//redis与生成key对应
					notExist = false;
				}
			}

			if (notExist){//redis与生成key不对应，更新时需要删除redis中多余key
				needDeleteOnUpdateActionKeys.add(existKey);
			}
		}
	}
}
```
6. 如果是删除操作，将所有信息(包括用于查询的key与更新的key)
```java
if (changeOfRow.getEventType().equals(ChangeOfRow.EventType.DELETE) || logicDelete){
            for (String idKey:needDeleteOnDeleteActionKeys){
                pipelineClient.hDel(RoSerializer.serialize(idKey),RoSerializer.serialize(changeOfRow.getBusinessId()));
            }
            pipelineClient.del(RoSerializer.serialize(relationKeyId));
        }
```
7. 如果是更新操作，首先将redis中比要更新查询key多的部分删除，然后更新查询key信息.并将新的查询key 与业务唯一标识id关联
```java
for (String idKey:needDeleteOnUpdateActionKeys){
	pipelineClient.hDel(RoSerializer.serialize(idKey),RoSerializer.serialize(changeOfRow.getBusinessId()));
}

for (String idKey:roRelationIdKeys){
	pipelineClient.hSet(RoSerializer.serialize(idKey),RoSerializer.serialize(changeOfRow.getBusinessId()),RoSerializer.serialize(infoRo));
}
pipelineClient.set(RoSerializer.serialize(relationKeyId),RoSerializer.serialize(roRelationIdKeys));
```
## 打标标记查询流程
1. 根据业务不同，创建对应查询实体
```java
ImMultishopBrandRo roBrandQuery = new ImMultishopBrandRo();
roBrandQuery.setBrandId(this.getInstallRequest().getBrandId());
roBrandQuery.setCategoryId(this.getInstallRequest().getCategoryThreeId());
```
2. 根据实体生成查询key
```java
public String[] getRoRelationIdKeys(String jimdbKeyPrefix){
	if (relationIdKeys == null){

		List<String[]> keyList = this.getKeyList();
		relationIdKeys = new String[keyList.size()];

		for (int i = 0;i<keyList.size();i++){
			String[] roKey = keyList.get(i);
			StringBuffer sb = new StringBuffer();
			sb.append(jimdbKeyPrefix)
					.append(":")
					.append(this.getDbName())
					.append(":")
					.append(this.getTableName());
			for(int j=0;j<roKey.length;j++){
				sb.append(":");
				sb.append(roKey[j]);
			}
			relationIdKeys[i] = sb.toString();
		}
	}

	return this.relationIdKeys;
}
```
3. 查询redis，进行反序列化并进行业务操作