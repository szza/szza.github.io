# CONFIG

由于配置文件广泛应用在REDIS中，,因此这一节主要讲述下REDIS的配置文件。

配置文件中共有五种配置类型数据:`bool`、`string`、`enum`和`numeric`，

### Bool

#### boolConfigData

```cpp
typedef struct boolConfigData {
    int* config;					  			   // 指向变量          
    const int default_value; 		   				// 该变量的默认值 
    int (*is_valid_fn)(int val, char **err);  		 // 校验该变量是否有效的函数指针。返回1则有效，0则无效
    int (*update_fn)(int val, int prev, char **err);  // 更新该变量值的函数指针。返回1则有效，0则无效
} boolConfigData;
```

#### typeData

`typeData`是个联合体，表示某种数据类型

```cpp
typedef union typeData {
    boolConfigData      yesno;	// bool类型的数据
    stringConfigData    string;        
    enumConfigData      enumd;
    numericConfigData   numeric;
} typeData;
```

#### typeInterface

而 `typeInterface` 是操作`typeData`对象的接口，

```cpp
// 操作类型的接口
typedef struct typeInterface {
    void (*init)(typeData data);
    int  (*load)(typeData data, sds *argc, int argv, char **err);
    int  (*set)(typeData data, sds value, int update, char **err);
    void (*get)(client *c, typeData data);
    void (*rewrite)(typeData data, const char *name, struct rewriteConfigState *state);
} typeInterface;
```

##### boolConfigInit

`boolConfigInit`函数，即 `typeInterface::init` 字段，用来初始化的BOOL类型数据实体的默认值 `data.yesno.default_value`

```cpp
static void boolConfigInit(typeData data) {
    *data.yesno.config = data.yesno.default_value;
}
```

##### boolConfigSet

`boolConfigSet`函数，即`typeInterface::set`字段的函数指针，用于将输入参数`value`赋值给`data.yesno.config`。如果发生错误，`data`中的值不变。

```cpp
///@brief 将 @c value 转为后赋值给 data中的数据实体
static int boolConfigSet(typeData data, sds value, int update, char **err) {
    int yn = yesnotoi(value);
    if (yn == -1) {
        *err = "argument must be 'yes' or 'no'";
        return 0;
    }

    if (data.yesno.is_valid_fn && !data.yesno.is_valid_fn(yn, err))
        return 0;

    int prev = *(data.yesno.config);    // 之前的值
    *(data.yesno.config) = yn; 

    if (update && data.yesno.update_fn && !data.yesno.update_fn(yn, prev, err)) {
        // 如果发生错误则回复
        *(data.yesno.config) = prev;
        return 0;
    }
    return 1;
}
```

##### boolConfigGet

将`data`的数据实体回复给客户端

```cpp
static void boolConfigGet(client *c, typeData data) {
    addReplyBulkCString(c, *data.yesno.config ? "yes" : "no");
}
```

##### boolConfigRewrite

重写配置文件

```cpp
static void boolConfigRewrite(typeData data, const char *name, struct rewriteConfigState *state) {
    rewriteConfigYesNoOption(state, name,*(data.yesno.config), data.yesno.default_value);
}
```

#### standardConfig

`standardConfig`结构定义了一个完整的对象

```cpp
// 配置数据
typedef struct standardConfig {
    const char*   name;         // 这个变量在配置文件中的名字
    const char*   alias;        // 别名
    const int     modifiable;   // 这个变量是否可修改
    typeInterface interface;    // 操作这个变量的接口
    typeData      data;         // 数据实体
} standardConfig;
```

REDIS定义了一个 `standardConfig`类型的数组，存储四种不同的变量的配置数据

```cpp
standardConfig configs[];
```

##### createBoolConfig

REDIS使用了一个宏定义`createBoolConfig` 来创建 `standardConfig`对象：

```cpp
#define createBoolConfig(name, alias, modifiable, config_addr, default, is_valid, update) { \
    embedCommonConfig(name, alias, modifiable) \
    embedConfigInterface(boolConfigInit, boolConfigSet, boolConfigGet, boolConfigRewrite) \
    .data.yesno = { \
        .config        = &(config_addr), \
        .default_value = (default), \
        .is_valid_fn   = (is_valid), \
        .update_fn     = (update), \
    } \
}
```

该宏定义展开如下：

```cpp
 //  createBoolConfig(name, alias, modifiable, config_addr, default, is_valid, update)
 //  等价于 
  {
    .name       = (config_name),  
    .alias      = (config_alias), 
    .modifiable = (is_modifiable),
    .interface  = { 
                    .init       = (initfn),    
                    .set        = (setfn),     
                    .get        = (getfn),     
                    .rewrite    = (rewritefn)  
                  }
   .data.yesno = { 
                    .config        = &(config_addr), 
                    .default_value = (default), 
                    .is_valid_fn   = (is_valid), 
                    .update_fn     = (update), 
                } 
    }
```

因此，通过宏定义 `createBoolConfig` 即使创建BOOL类型的对象、给予初始默认值以及校验函数和更新函数。

#### config_set_bool_field

`config_set_bool_field`宏，将名为`_name`的变量值修改为`_var`

```CPP
/// @brief 将 @c _name 对应的值赋给 @c _var 
// CONFIG SET NAME VALUE
#define config_set_bool_field(_name,_var)            \
    } else if (!strcasecmp(c->argv[2]->ptr,_name)) { \
        int yn = yesnotoi(o->ptr);                   \
        if (yn == -1) goto badfmt;                   \
        _var = yn;
```

配置其余类型的变量类似，不再赘叙





