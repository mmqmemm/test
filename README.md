### 创建配件实体类

```java
@Data
@TableName(value = "yjx_parts")  // 指定数据库表名
@Builder  // 构造器
@AllArgsConstructor  // 全参构造函数
@NoArgsConstructor
public class Parts {
    private Integer partId; //配件id
    private String partName; //配件名称
    private String partDescription;//配件描述
    private Double partPrice;//配件价格
    private Integer stockQuantity;//配件数量
    private Integer supplierId;//配件供应商id
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```



### 查询配件(分页查询和条件搜索)

#### 创建Controller

```java
@RestController
@RequestMapping("/parts")
public class PartsController {
    @Autowired
    private PartsService partsService;
    @GetMapping("/list")
    public Result<Map<String, Object>> list(
            //使用RequestParam注解,接收前端传递的参数
            @RequestParam(value = "userId",required = false) Integer userId,
            //required = false 前端可以不是必须传递，后端可以设置默认值
            @RequestParam(value = "searchKeyword",required = false) String searchKeyword,
            @RequestParam(value ="pageNum",required = false) Integer pageNum,
            @RequestParam(value = "pageSize",required = false) Integer pageSize,
            @RequestParam(value =  "sortField",required = false) String sortField,
            @RequestParam(value =  "sortOrder",required = false) String sortOrder
    )
    {
        Map<String,Object> data = partsService.list(userId,searchKeyword,pageNum,pageSize,sortField,sortOrder);
        return Result.success(data);
    }
}
```

#### 创建Service接口

```java
//查询配件
Map<String, Object> list(Integer userId, String searchKeyword, Integer pageNum, Integer pageSize, String sortField, String sortOrder);
```

#### 创建Service实现类

```java
@Autowired
private PartsMapper partsMapper;
@Override
public Map<String, Object> list(Integer userId, String searchKeyword, Integer pageNum, Integer pageSize, String sortField, String sortOrder) {
    // 1. 参数校验与默认值设置（避免null异常）
    pageNum = (pageNum == null || pageNum < 1) ? 1 : pageNum;
    pageSize = (pageSize == null || pageSize < 1) ? 10 : pageSize;
    searchKeyword = (searchKeyword == null) ? "" : searchKeyword.trim();
    // 默认排序：按创建时间降序
    if (sortField == null || sortField.trim().isEmpty()) {
        sortField = "createdAt";
    }
    if (sortOrder == null || (!"asc".equalsIgnoreCase(sortOrder) && !"desc".equalsIgnoreCase(sortOrder))) {
        sortOrder = "desc";
    }

    // 2. 分页查询
    Page<Parts> page = new Page<>(pageNum, pageSize);

    IPage<Parts> partsIPage = partsMapper.list(
            page, userId, searchKeyword, sortField, sortOrder
    );

    // 3. 封装响应格式（与你之前的 getAllRepair 一致，前端可复用逻辑）
    Map<String, Object> responseMap = new HashMap<>();
    responseMap.put("pageResult", partsIPage.getRecords());  // 维修管理列表
    responseMap.put("count", partsIPage.getTotal());                  // 总条数（用于分页组件）
    return responseMap;
}
```

#### 创建mapper层

```java
//查询所有配件记录
/**
 * @param page 分页对象（页码、每页条数）
 * @param userId 当前登录用户ID（用于权限过滤：如管理员查全部，普通用户查自己）
 * @param searchKeyword 搜索关键词（模糊匹配手机型号、维修描述、用户名）
 * @param sortField 排序字段（如 createdAt、repairId）
 * @param sortOrder 排序方向（asc/desc）
 * @return 分页结果（包含 Management 列表和总条数）
 */
@Select("""
    SELECT 
        ya.*
    FROM yjx_parts ya
    WHERE 1=1
    -- 权限过滤：与你之前的逻辑一致（userId=1/5/8查全部，其他查自己关联的）
    AND ( #{userId} IN (1,5,8) )
    -- 搜索关键词：模糊匹配手机型号、维修描述、用户名（前端可能按这些字段搜索）
    AND ( 
        ya.part_name LIKE CONCAT('%', #{searchKeyword}, '%') 
        OR ya.part_description LIKE CONCAT('%', #{searchKeyword}, '%') 
    )
    -- 排序：与你之前的逻辑一致（支持按创建时间、维修ID排序）
    ORDER BY 
        CASE WHEN #{sortField} IS NOT NULL AND #{sortOrder} IS NOT NULL 
             THEN CASE #{sortField} 
                  WHEN 'createdAt' THEN ya.created_at 
                  WHEN 'part_id' THEN ya.part_id 
                  ELSE ya.created_at END 
        ELSE ya.created_at END 
        ${sortOrder == 'desc' ? 'DESC' : 'ASC'}
""")
IPage<Parts> list(Page<Parts> page, Integer userId, String searchKeyword, String sortField, String sortOrder);
```



### 增加配件

#### 创建Controller

```java
//增加配件
@PostMapping("/addPart")
public Result<String> addPart(@RequestBody Parts parts) {
   return partsService.addPart(parts);
}
```

#### 创建Service接口

```java
//增加配件
    Result<String> addPart(Parts parts);
```

#### 创建Service实现类

```java
//创建配件
@Override
public Result<String> addPart(Parts parts) {
    //判断参数（名字，价格，数量，供应商id均不为空）
    if (parts.getPartName() == null || parts.getPartName().trim().isEmpty())
        return Result.fail("配件名称不能为空",404);
    if (parts.getPartPrice() == null || parts.getPartPrice() <= 0)
        return Result.fail("配件价格不能小于0",404);
    if (parts.getStockQuantity() == null || parts.getStockQuantity() <= 0)
        return Result.fail("配件数量不能小于0",404);
    if (parts.getSupplierId() == null || parts.getSupplierId() <= 0)
        return Result.fail("供应商id不能小于0",404);

    // 设置默认值
    parts.setPartDescription("测试");
    //插入数据库
    Integer result = partsMapper.addParts(parts);
    if (result != null && result > 0) {
        return Result.success("创建配件成功");
    } else {
        return Result.fail("创建配件失败",404);
    }
}
```

#### 创建mapper层

```java
//添加组件
@Insert("INSERT INTO yjx_parts(part_name, part_description, part_price, stock_quantity, supplier_id  ) " +
        "VALUES (#{partName}, #{partDescription}, #{partPrice}, #{stockQuantity}, #{supplierId}) ")
Integer addParts(Parts parts);
```



### 更新配件

#### 创建Controller

```java
//修改配件
@PostMapping("/updatePart")
public Result<String> updatePart(@RequestBody Parts parts) {
    return partsService.updatePart(parts);
}
```

#### 创建Service接口

```java
//更新配件
Result<String> updatePart(Parts parts);
```



#### 创建Service实现类

```java
@Override
public Result<String> updatePart(Parts parts) {
    // 参数校验，确保partId存在
    if (parts.getPartId() == null) {
        return Result.fail("配件ID不能为空",404);
    }
    // 执行更新操作，MyBatis - Plus的updateById方法会根据主键更新实体类中非null的字段
    int result = partsMapper.updateParts(parts);
    if (result >0) {
        return Result.success("配件更新成功");
    } else {
        return Result.fail("配件更新失败，可能配件不存在",404);
    }
}
```



#### 创建mapper层

```java
// 更新配件
@Update("UPDATE yjx_parts " +
        "SET part_name = #{partName}, " +
        "part_description = #{partDescription}, " +
        "part_price = #{partPrice}, " +
        "stock_quantity = #{stockQuantity}, " +
        "supplier_id = #{supplierId} " +
        "WHERE part_id = #{partId}")
int updateParts(Parts parts);
```



### 删除配件

#### 创建Controller

```java
//删除配件
@DeleteMapping("/delete/{partId}")
public Result<?> delete(@PathVariable Integer partId) {
    int result = partsService.deletePartsByPartId(partId);
    if (result > 0) {
        return Result.success("删除成功");
    } else {
        return Result.fail("删除失败，配件不存在",404);
    }
}
```

#### 创建Service接口

```java
    //删除配件
    int deletePartsByPartId(Integer partId);

```

#### 创建Service实现类

```java
@Override
@Transactional(propagation = Propagation.REQUIRED)
public int deletePartsByPartId(Integer partId) {
    if(partId == null) {
        Result.fail("配件id不能为空",404);
    }
    Parts parts = partsMapper.queryPartsByPartId(partId);
    if(parts == null) {
        Result.fail("配件记录不存在",404);
    }
    int rows = partsMapper.deletePartsByPartId(partId);
    if(rows <= 0) {
        Result.fail("删除失败",404);
    }
    return rows;
}
```

#### 创建mapper层

```java
//根据配件id查询配件记录
@Select("select * from yjx_parts where part_id = #{partId}")
Parts queryPartsByPartId(Integer partId);
```

```java
//删除配件
@Delete("delete from yjx_parts where part_id =#{partId}")
int deletePartsByPartId(Integer partId);
```





























































