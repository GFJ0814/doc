### API升级技术方案

#### 整体方案思考：

​	整体思路是改造标签受众，进而直接生成一份用户参数取值配置，供调用方传值。综合两个方案，一是继续用受众，由我方维护标签受众关系 二是标签暴露到创建实验，由用户组合。两个方案对比，我选择方案二，原因如下：

1. 历史逻辑改动小

2. 对应用者清楚

3. 标签受众调整实时生效，无需数据搅动

4. 以后无需维护受众，也减少了初始化实验时候多表的关联查询


#### 方案步骤

1. 标签整理

   - 将原来的标签，每个标签增加一个对应的分类，如下图

   | 类别                | 取值                                              | 中文名 |
   | ------------------- | ------------------------------------------------- | ------ |
   | clientType          | PC、M、APP                                        |        |
   | channelType         | beike,lianjia                                     |        |
   | app_version         | 选择APP、以及channelType后，级联展示，举例：8.0.2 |        |
   | os                  | safari                                            |        |
   | device              | huawei、xiaomi                                    |        |
   | ciryCode            | 城市id：11000                                     |        |
   | URL                 |                                                   |        |
   | phone（配合白名单） |                                                   |        |
   |                     |                                                   |        |

   - 创建实验，受众部分选择人群包受众和AB受众，人群包受众不变，当选择AB类型的受众时候，由用户根据标签组合。组合类型包括且（+）、或（?）、非（-）

   - 设置白名单

   - 创建实验完成，在原来的基础上显示一下参数示例，用于调用方参考

2. 标签组合以及运算规则

   - 标签组合方式：类似于DMP平台使用标签组与标签组的逻辑关系来做

     ![img](https://preview.lianjia.com/yunpan/5f3a4ca4-692a-4476-8469-d213e0bd72bf!m_simple,f_%E6%A0%87%E7%AD%BE%E7%BB%84%E5%90%88.png)

   - 举例子：

     如果是（PC北京）or（APP西安），则第一个标签组为

     端类型 = pc  且  城市 = 北京

     第二个标签组为：

     端类型 = APP 且 城市 = 西安

     两个标签组之间是或者的关系，每个参数的值允许多选，比如城市=北京、成都、上海，之间默认是或者的关系

   - 标签计算思路：

     1、将标签组之间的逻辑规则，转换成json存储，并转化对象维护

     如图所示：

```json
[
    {
        "logicOp":"",
        "tagParams":[
            {
                "field":"cityCode",
                "op":"=",
                "values":[
                    "11000" //北京的城市编码
                ]
            },
            {
                "field":"clientType",
                "logicOp":"and",
                "op":"=",
                "values":[
                    "pc"
                ]
            }
        ]
    },
    {
        "logicOp":"or",
        "tagParams":[
            {
                "field":"cityCode",
                "op":"=",
                "values":[
                    "21000" //假设是西安的城市编码
                ]
            },
            {
                "field":"clientType",
                "logicOp":"and",
                "op":"=",
                "values":[
                    "app"
                ]
            }
        ]
    }
]
```
​	

​	每一个标签组，为一个标签组对象，属性有记录与其他标签组的运算关系的logicOP，以及标签组里面包含的所有标签自身取值关系的集合
```java
package com.lianjia.platform.sampling.bean.params;
import java.util.*;
/**
 * @program: sampling
 * @description: 参数组
 * @author: guofangjun001
 * @create: 2019-01-09 20:59
 **/
public class ParamGroup {
    private  String logicOp; //记录与下一个标签组之间的运算关系，如果是第一个标签组，取值为空串
    private  List<TagParam>  tagParams;
}


```

标签取值对象：维护标签取值关系以及标签与标签之间的逻辑关系
```java
public class TagParam {
    private String field; //属性名 比如citycode
    private String op; //属性取值  = 或者 ！=
    private List<String> values; //属性取值 北京、上海的citycode
    private String logicOp; //与下一个标签之间的运算关系，如果是第一个标签，则为空串
}
```

​	2.计算规则：
遍历所有的标签组，针对每一个标签组的所有标签，根据标签名，从参数中获取对应参数的值，将返回得到的结果boolean值用于和下一个标签的的运算结果进行逻辑运算，得到整个标签组最终计算结果true或者false。同理遍历第二个标签组，将两个标签组的结果根据两个标签组之间的逻辑关系值，映射为相应Boolean值的逻辑运算。最终，得到判断结果。
> 为了避免与或逻辑的从前往后运算，导致的计算结果不清楚，不好理解，规定了标签组内的逻辑运算都是与运算，标签组之间的运算都是或运算。这样方便与或组合。原来的非标签转换为标签的不等于计算。
demo逻辑如下：
```java
package com.lianjia.platform.sampling.service;

import com.alibaba.fastjson.JSON;
import com.lianjia.platform.sampling.bean.params.NewServerParam;
import com.lianjia.platform.sampling.bean.params.ParamGroup;
import com.lianjia.platform.sampling.bean.params.TagParam;
import org.apache.commons.lang3.StringUtils;

import java.util.*;

/**
 * @program: sampling
 * @description: 标签组合运算方法
 * @author: guofangjun001
 * @create: 2019-01-09 21:03
 **/
public class TagCalculateService {

    public boolean istrue(List<ParamGroup> paramGroups, NewServerParam newServerParam) {
        boolean result = true;
        for (ParamGroup paramGroup : paramGroups) {
            if (StringUtils.isNotEmpty(paramGroup.getLogicOp())) {
                switch (paramGroup.getLogicOp()) {
                    case "and":
                        result = result && getParamGroupResult(paramGroup, newServerParam);
                        break;
                    case "or":
                        result = result || getParamGroupResult(paramGroup, newServerParam);
                        break;
                }
            } else {
                result = getParamGroupResult(paramGroup, newServerParam);
            }

        }
        return result;
    }

    private boolean filter(String paramValue, String operate, List<String> values) {
        boolean r = false;
        switch (operate) {
            case "=":
                r = values.contains(paramValue);
                break;
            case "!=":
                r = !(values.contains(paramValue));
                break;
        }
        return r;
    }

    private boolean getTagParamResult(TagParam tagParam, NewServerParam newServerParam) {
        String op = tagParam.getOp();
        List<String> list = tagParam.getValues();
        boolean re = false; //第一个表达式的运算结果
        switch (tagParam.getField()) {
            case "cityCode":
                re = filter(newServerParam.getCityCode(), op, list);
                break;
            case "appType":
                re = filter(newServerParam.getAppType(), op, list);
                break;
            case "clientType":
                re = filter(newServerParam.getClientType(), op, list);
                break;
        }
        return re;
    }

    private boolean getParamGroupResult(ParamGroup paramGroup, NewServerParam newServerParam) {
        boolean result = true;
        for (TagParam tagParam : paramGroup.getTagParams()) {
            if (StringUtils.isNotEmpty(tagParam.getLogicOp())) {
                switch (tagParam.getLogicOp()) {
                    case "and":
                        result = result && getTagParamResult(tagParam, newServerParam);
                        break;
                    case "or":
                        result = result || getTagParamResult(tagParam, newServerParam);
                        break;
                }
            } else {
                result = getTagParamResult(tagParam, newServerParam);
            }
        }
        return result;
    }
}
```

单元测试：
```java
 public static void main(String[] args) {
        //测试PC北京（11000） or APP西安（citycode假设为：21000）
        List<ParamGroup> list = new ArrayList<>();
        List values1 = new ArrayList();
        List tagParams = new ArrayList();
        ParamGroup paramGroup1 = new ParamGroup();
        TagParam tagParam1 = new TagParam();
        tagParam1.setField("cityCode");
        tagParam1.setOp("=");
        values1.add("11000");
        tagParam1.setValues(new ArrayList<String>(values1));

        TagParam tagParam2 = new TagParam();
        tagParam2.setLogicOp("and");
        tagParam2.setField("clientType");
        List values2 = new ArrayList();
        values2.add("pc");
        tagParam2.setValues(values2);
        tagParam2.setOp("=");
        


        tagParams.add(tagParam1);
        tagParams.add(tagParam2);

        paramGroup1.setLogicOp("");
        paramGroup1.setTagParams(tagParams);


        ParamGroup paramGroup2 = new ParamGroup();
        List tagParams2 = new ArrayList();
        TagParam tagParam21 = new TagParam();
        tagParam21.setField("cityCode");
        tagParam21.setOp("=");
        List values21 = new ArrayList();
        values21.add("21000");
        tagParam21.setValues(new ArrayList<String>(values21));

        TagParam tagParam22 = new TagParam();
        tagParam22.setLogicOp("and");
        tagParam22.setField("clientType");
        List values22 = new ArrayList();
        values22.add("app");
        tagParam22.setValues(values22);
        tagParam22.setOp("=");

        tagParams2.add(tagParam21);
        tagParams2.add(tagParam22);
        paramGroup2.setLogicOp("or");
        paramGroup2.setTagParams(tagParams2);


        list.add(paramGroup1);
        list.add(paramGroup2);
        System.out.println(JSON.toJSONString(list));


        NewServerParam newServerParam = new NewServerParam();
        newServerParam.setCityCode("11000");
        newServerParam.setAppType("beike");
        newServerParam.setClientType("PC");

        System.out.println(new TagCalculateService().istrue(list, newServerParam));
```
输出：true


3. 特殊处理

   - 白名单配置phone,怎么处理？判断如果是phone类型的白名单，就在生成的参数中加上phone

     逻辑：

     ​	（1）增加一个新的白名单类型：phone

     ​	（2）判断白名单类型，取值，如果没有传phone值，则走分流逻辑配置。

   - 原来所有的标签都重新增加一个分类吗，再增加一个标志比较好，对于原来的，只作为展示。

     增加一列叫param：

     ​	或者把category用上

#### 实现过程：

1. 数据库修改：
   - tag表增加type（区别新版（取值=1）和旧版（取值=0），便于管理）、param（参数：比如clientType），value（参数取值：比如app、pc），新增标签，根据选择的版本，确定要写入的参数，比如选择新版，才会显示新版参数的输入框，修改则不能变更标签类型。修改原有标签接口：【新增】【修改】
   - 标签列表页面增加一个版本选择框
   - exp_group表和experimentation表增加一列：custom_audience（自定义受众）,用于保存用户圈定的标签组合配置

2. 老版本受众改造：

  - 受众管理新增受众和修改受众时候，只能遍历旧版本的标签，所以要提供根据类型获取标签列表接口【接口1】

3. 创建实验，设置受众

   - 前端提供三种选择：自定义受众（custom）、DMP受众（dmp）、AB受众（ab）
   - 选择自定义受众，请求后端接口，查询所有新版标签，按照参数归类【接口2】，前端映射成各个维度，标签取值提供sug。分流id类型同AB受众，其他类型受众不用改
   - 如果是全部用户：通过标签组合形式，比如appType=lianjia,beike;
   - 将传入的配置以json串格式保存到每个experimentation的custom_audience字段以及exp_group表的该字段。

   - 实验详情展示：判断customer_audience是否有值，如果有值，则翻译json，如果为空，则按照原有逻辑。

4. 加载实验，初始化配置

   - 遍历实验，判断实验的custom_audience字段是否为空，如果不为空，则将json转换为List结构存储在pipeline对象中。如果为空，则按照原来的方式存储过滤器。

5. 受众分流

   - 新建一个服务端接口API，获取请求参数包含各个标签参数的具体值以及分流id、实验key，是否灰度。
   - 根据实验key，获取每个实验的分流配置以及受众配置。
   - 根据获取到的受众配置，调用标签计算方法，判断是否在当前自定义受众中，满足则分流，不满足，则退出。

6. 白名单支持电话号码

   - 白名单增加phone类型
   - 自定义受众的白名单，增加uuid、ucid以及phone类型展示
   - 创建实验判断，如果是phone类型白名单，则显示参数加上phone

7. phone类型白名单分流

   - 获取到白名单id，判断如果是phone类型，获取phone值，然后按照正常白名单判断走

#### 设计问题

1. 要满足的需求老的受众标签存在，还要再来一版本新的标签和受众产生的矛盾

2. 设计的目的：摆脱对OP的重依赖

3. 设计想法：

   - 操作方式不用变，受众和标签中都增加两个版本的内容，新增受众时候，选择先选择版本，然后，才会去获取标签列表。创建实验的时候，接入API版本为必填项，确定接入了API版本之后，才会在对应选择下去选择DMP受众或者AB对应版本的受众。并且新版本受众，才会支持电话号的白名单。每一个版本上面加上一个？提示，关联到API文档上。实验创建完成之后， 显示一行必传参数。













