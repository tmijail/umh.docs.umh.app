---
title: "Delete Assets from the Database"
content_type: task
description: |
    This task shows you how to delete assets from the database.
weight: 51
---

<!-- overview -->

This is useful if you have created assets by mistake, or to delete
the ones that are no longer needed.

{{% notice warning %}} This task deletes data from the database. Make sure you
have a backup of the database before you proceed. {{% /notice %}}

## {{% heading "prerequisites" %}}

{{< include "task-aftinst-prereqs.md" >}}

Also, make sure to backup the database before you proceed. For more information,
see [Backing Up and Restoring the Database](/docs/production-guide/backup_recovery/backup-timescale).

<!-- steps -->

## Delete assets from factoryinsight
If you want to delete assets from the `umh_v2` database, go to [this section](/docs/production-guide/administration/delete-assets/#delete-assets-from-umh_v2).
### Open the database shell

{{< include "open-database-shell.md" >}}

Connect to the `factoryinsight` database:

```bash
\c factoryinsight
```

### Choose the assets to delete

You have multiple choices to delete assets, like deleting a single asset, or
deleting all assets in a location, or deleting all assets with a specific name.

To do so, you can customize the SQL command using different filters. Specifically,
a combination of the following filters:

- `assetid`
- `location`
- `customer`

To filter an SQL command, you can use the `WHERE` clause. For example, using all
of the filters:

```sql
WHERE assetid = '<asset-id>' AND location = '<location>' AND customer = '<customer>';
```

You can use any combination of the filters, even just one of them.

Here are some examples:

- Delete all assets with the same name  from any location and any customer:

  ```sql
  WHERE assetid = '<asset-id>'
  ```

- Delete all assets in a specific location:

  ```sql
   WHERE location = '<location>'
   ```

- Delete all assets with the same name in a specific location:

  ```sql
  WHERE assetid = '<asset-id>' AND location = '<location>'
  ```

- Delete all assets with the same name in a specific location for a single customer:

  ```sql
  WHERE assetid = '<asset-id>' AND location = '<location>' AND customer = '<customer>'
  ```

### Delete the assets

Once you know the filters you want to use, you can use the following SQL commands
to delete assets:

```sql
BEGIN;

WITH assets_to_be_deleted AS (SELECT id FROM assettable <filter>)
DELETE FROM shifttable WHERE asset_id IN (SELECT id FROM assets_to_be_deleted);

WITH assets_to_be_deleted AS (SELECT id FROM assettable <filter>)
DELETE FROM counttable WHERE asset_id IN (SELECT id FROM assets_to_be_deleted);

WITH assets_to_be_deleted AS (SELECT id FROM assettable <filter>)
DELETE FROM ordertable WHERE asset_id IN (SELECT id FROM assets_to_be_deleted);

WITH assets_to_be_deleted AS (SELECT id FROM assettable <filter>)
DELETE FROM processvaluestringtable WHERE asset_id IN (SELECT id FROM assets_to_be_deleted);

WITH assets_to_be_deleted AS (SELECT id FROM assettable <filter>)
DELETE FROM processvaluetable WHERE asset_id IN (SELECT id FROM assets_to_be_deleted);

WITH assets_to_be_deleted AS (SELECT id FROM assettable <filter>)
DELETE FROM producttable WHERE asset_id IN (SELECT id FROM assets_to_be_deleted);

WITH assets_to_be_deleted AS (SELECT id FROM assettable <filter>)
DELETE FROM shifttable WHERE asset_id IN (SELECT id FROM assets_to_be_deleted);

WITH assets_to_be_deleted AS (SELECT id FROM assettable <filter>)
DELETE FROM statetable WHERE asset_id IN (SELECT id FROM assets_to_be_deleted);

WITH assets_to_be_deleted AS (SELECT id FROM assettable <filter>)
DELETE FROM assettable WHERE id IN (SELECT id FROM assets_to_be_deleted);

COMMIT;
```

Optionally, you can add the following code before the last `WITH` statement if
you used the track&trace feature:

   ```sql
   WITH assets_to_be_deleted AS (SELECT id FROM assettable <filter>), uniqueproducts_to_be_deleted AS (SELECT uniqueproductid FROM uniqueproducttable WHERE asset_id IN (SELECT id FROM assets_to_be_deleted))
   DELETE FROM producttagtable WHERE product_uid IN (SELECT uniqueproductid FROM uniqueproducts_to_be_deleted);

   WITH assets_to_be_deleted AS (SELECT id FROM assettable <filter>), uniqueproducts_to_be_deleted AS (SELECT uniqueproductid FROM uniqueproducttable WHERE asset_id IN (SELECT id FROM assets_to_be_deleted))
   DELETE FROM producttagstringtable WHERE product_uid IN (SELECT uniqueproductid FROM uniqueproducts_to_be_deleted);

   WITH assets_to_be_deleted AS (SELECT id FROM assettable <filter>), uniqueproducts_to_be_deleted AS (SELECT uniqueproductid FROM uniqueproducttable WHERE asset_id IN (SELECT id FROM assets_to_be_deleted))
   DELETE FROM productinheritancetable WHERE parent_uid IN (SELECT uniqueproductid FROM uniqueproducts_to_be_deleted) OR child_uid IN (SELECT uniqueproductid FROM uniqueproducts_to_be_deleted);

   WITH assets_to_be_deleted AS (SELECT id FROM assettable <filter>)
   DELETE FROM uniqueproducttable WHERE asset_id IN (SELECT id FROM assets_to_be_deleted);
   ```

<!-- umh_v2 -->
## Delete assets from umh_v2

### Open the database shell

{{< include "open-database-shell.md" >}}

Connect to the `umh_v2` database:

```bash
\c umh_v2
```

### Choose the assets to delete
You have multiple choices to delete assets, like deleting a single asset, or 
deleting all assets in a location, or deleting all assets with a specific name.

To do so, you can customize the SQL command using different filters. Specifically, 
a combination of the following filters:

- `enterprise`
- `site`
- `area`
- `line`
- `workcell`
- `origin_id`

To filter an SQL command, you can use the WHERE clause. For example, you can filter 
by `enterprise`, `site`, and `area`:

```sql
WHERE enterprise = '<your-enterprise>' AND site = '<your-site>' AND area = '<your-area>';
```

You can use any combination of the filters, even just one of them.

### Delete the assets
Once you know the filters you want to use, you can use the following SQL commands 
to delete assets:

```sql
BEGIN;
WITH assets_to_be_deleted AS (SELECT id FROM asset <filter>)
DELETE FROM tag WHERE asset_id IN (SELECT id FROM assets_to_be_deleted);

WITH assets_to_be_deleted AS (SELECT id FROM asset <filter>)
DELETE FROM tag_string WHERE asset_id IN (SELECT id FROM assets_to_be_deleted);

WITH assets_to_be_deleted AS (SELECT id FROM asset <filter>)
DELETE FROM asset WHERE id IN (SELECT id FROM assets_to_be_deleted);
COMMIT;
```

<!-- discussion -->

<!-- Optional section; add links to information related to this topic. -->
## {{% heading "whatsnext" %}}

- See [Backing Up and Restoring the Database](/docs/production-guide/backup_recovery/backup-restore-database)
