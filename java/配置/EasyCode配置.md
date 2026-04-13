文件名：EasyCodeConfig.json

```json
{
  "author" : "majiale",
  "version" : "1.2.8",
  "userSecure" : "",
  "currTypeMapperGroupName" : "Default",
  "currTemplateGroupName" : "ChangTech",
  "currColumnConfigGroupName" : "Default",
  "currGlobalConfigGroupName" : "Default",
  "typeMapper" : {
    "Default" : {
      "name" : "Default",
      "elementList" : [ {
        "matchType" : "REGEX",
        "columnType" : "varchar(\\(\\d+\\))?",
        "javaType" : "java.lang.String"
      }, {
        "matchType" : "REGEX",
        "columnType" : "char(\\(\\d+\\))?",
        "javaType" : "java.lang.String"
      }, {
        "matchType" : "REGEX",
        "columnType" : "(tiny|medium|long)*text",
        "javaType" : "java.lang.String"
      }, {
        "matchType" : "REGEX",
        "columnType" : "decimal(\\(\\d+,\\d+\\))?",
        "javaType" : "java.math.BigDecimal"
      }, {
        "matchType" : "ORDINARY",
        "columnType" : "integer",
        "javaType" : "java.lang.Integer"
      }, {
        "matchType" : "ORDINARY",
        "columnType" : "tinyint(1)",
        "javaType" : "java.lang.Boolean"
      }, {
        "matchType" : "REGEX",
        "columnType" : "(tiny|small|medium)*int(\\(\\d+\\))?",
        "javaType" : "java.lang.Integer"
      }, {
        "matchType" : "ORDINARY",
        "columnType" : "int4",
        "javaType" : "java.lang.Integer"
      }, {
        "matchType" : "ORDINARY",
        "columnType" : "int8",
        "javaType" : "java.lang.Long"
      }, {
        "matchType" : "ORDINARY",
        "columnType" : "bigint unsigned",
        "javaType" : "java.lang.Long"
      }, {
        "matchType" : "REGEX",
        "columnType" : "bigint(\\(\\d+\\))?",
        "javaType" : "java.lang.Long"
      }, {
        "matchType" : "ORDINARY",
        "columnType" : "date",
        "javaType" : "java.util.Date"
      }, {
        "matchType" : "ORDINARY",
        "columnType" : "datetime",
        "javaType" : "java.util.Date"
      }, {
        "matchType" : "ORDINARY",
        "columnType" : "timestamp",
        "javaType" : "java.util.Date"
      }, {
        "matchType" : "ORDINARY",
        "columnType" : "time",
        "javaType" : "java.time.LocalTime"
      }, {
        "matchType" : "ORDINARY",
        "columnType" : "boolean",
        "javaType" : "java.lang.Boolean"
      }, {
        "matchType" : "ORDINARY",
        "columnType" : "json",
        "javaType" : "java.util.List"
      } ]
    }
  },
  "template" : {
    "ChangTech" : {
      "name" : "ChangTech",
      "elementList" : [ {
        "name" : "entity.java.vm",
        "code" : "##导入宏定义\n$!{define.vm}\n\n##设置表后缀（宏定义）\n#setTableSuffix(\"DO\")\n\n##保存文件（宏定义）\n#save(\"/model/entity\", \"DO.java\")\n\n##包路径（宏定义）\n#setPackageSuffix(\"model.entity\")\n\n##自动导入包（全局变量）\n$!{autoImport.vm}\nimport com.baomidou.mybatisplus.extension.activerecord.Model;\nimport com.baomidou.mybatisplus.annotation.TableName;\nimport lombok.Data;\n\n@Data\n@TableName(value = \"$!{tool.hump2Underline($tableInfo.name)}\")\npublic class $!{tableInfo.name}DO {\n#foreach($column in $tableInfo.fullColumn)\n#if(${column.comment})    \n    /** ${column.comment} */\n#end\n    private $!{tool.getClsNameByFullName($column.type)} $!{column.name};\n#end\n}\n"
      }, {
        "name" : "mapper.java.vm",
        "code" : "##导入宏定义\n$!{define.vm}\n\n##设置表后缀（宏定义）\n#setTableSuffix(\"Mapper\")\n\n##保存文件（宏定义）\n#save(\"/mapper\", \"Mapper.java\")\n\n##包路径（宏定义）\n#setPackageSuffix(\"mapper\")\n\nimport com.baomidou.mybatisplus.core.mapper.BaseMapper;\nimport org.apache.ibatis.annotations.Mapper;\nimport $!{tableInfo.savePackageName}.model.entity.$!{tableInfo.name}DO;\n\n@Mapper\npublic interface $!{tableName} extends BaseMapper<$!{tableInfo.name}DO> {\n\n}\n"
      }, {
        "name" : "repository.java.vm",
        "code" : "##导入宏定义\n$!{define.vm}\n\n##设置表后缀（宏定义）\n#setTableSuffix(\"Repository\")\n\n##保存文件（宏定义）\n#save(\"/repository\", \"Repository.java\")\n\n##包路径（宏定义）\n#setPackageSuffix(\"repository\")\n\nimport cn.changtech.framework.data.mybatisplus.repository.BaseRepository;\nimport $!{tableInfo.savePackageName}.model.entity.$!{tableInfo.name}DO;\nimport $!{tableInfo.savePackageName}.mapper.$!{tableInfo.name}Mapper;\nimport org.springframework.stereotype.Repository;\n\n@Repository\npublic class $!{tableName} extends BaseRepository<$!{tableInfo.name}Mapper, $!{tableInfo.name}DO> {\n}\n"
      } ]
    }
  },
  "columnConfig" : { },
  "globalConfig" : { }
}
```