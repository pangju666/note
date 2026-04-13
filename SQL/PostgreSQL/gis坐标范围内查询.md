```sql
SELECT id,
       name,
       point,
       remark,
       sphere,
       cover_image_md5,
       address,
       delete_time,
       create_time,
       update_time
FROM geo_point
WHERE geometry(geo_point.point) &&
      ST_SetSRID(ST_MakeBox2D(ST_Point(#{minX}, #{minY}), ST_Point(#{maxX}, #{maxY})), 4326)
  and delete_time is null
```