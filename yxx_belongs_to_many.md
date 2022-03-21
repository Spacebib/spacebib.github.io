# laravel Orm 多对多 id 错乱

## Model
```
    WasherTeam
	id
	name

    VehicleLocationArea
	id
	name
	lat_lng

    WasherTeamArea（中间表）
	id,
	washer_team_id,
	area_id
```

## 问题复现

```php

class WasherTeam extends Model
{

  public function areas(): BelongsToMany
  {  
     return $this->belongsToMany(VehicleLocationArea::class, WasherTeamArea::class, 'washer_team_id', 'area_id');
  }

}
```

当执行
```php
$areas = WasherTeam::query()->find(1)->areas->pluck('name', 'id');
```
`$areas` 获取的 id 一直是中间表 `WasherTeamArea` 的 id。无法获得实际想获取的 `VehicleLocationArea` id。

## 问题原因
通过追踪执行的 sql 语句，执行结果如下
```sql
select
  *,
  ST_AsText(lat_lng) as lat_lng,
  `washer_team_areas`.`washer_team_id` as `pivot_washer_team_id`,
  `washer_team_areas`.`area_id` as `pivot_area_id`
from
  `vehicle_location_areas`
  inner join `washer_team_areas` on `vehicle_location_areas`.`id` = `washer_team_areas`.`area_id`
where
  `washer_team_areas`.`washer_team_id` = 1
```

开始我怀疑是 laravel orm 自身内连查询 `select *  ...` 导致查询表的 id ，被中间表覆盖。后面发现  Model `VehicleLocationArea` 存在如下代码导致 id 被覆盖：
```php
public function newQuery()
{
  $raw = 'ST_AsText(lat_lng) as lat_lng ';
  return parent::newQuery()->addSelect('*', DB::raw($raw));
}
```

## 解决办法
加上表名前缀
```php
public function newQuery()
{
  $raw = 'ST_AsText(lat_lng) as lat_lng';
  return parent::newQuery()->addSelect("{$this->getTable()}.*", DB::raw($raw));
}
```
