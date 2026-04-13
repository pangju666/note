匹配 JSON 数组中**同时包含某个值**
```SQL
'标签1' MEMBER OF(tags)
```

空数组查询
```SQL
tags like '[]'
```

匹配 JSON 数组中**包含任意一个值**（OR 条件）
```SQL
JSON_OVERLAPS(tags, '["标签1", "标签2"]')
```

匹配 JSON 数组中**同时包含多个值**（AND 条件）
```SQL
JSON_CONTAINS(tags, '["标签1", "标签2"]')
```

JSON数组列建立**多值索引**
```SQL
CREATE INDEX idx_tags ON table_name (delete_status, (CAST(tags AS CHAR(20) ARRAY)));
```