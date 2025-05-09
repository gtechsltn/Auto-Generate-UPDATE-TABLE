# Script Auto-Generate UPDATE TABLE (non-PK columns only)
```
DECLARE @TableName NVARCHAR(128) = 'YourTableName'; -- 👈 change table name here
DECLARE @SchemaName NVARCHAR(128) = 'dbo';          -- 👈 change schema if needed

DECLARE @sql NVARCHAR(MAX);
DECLARE @cols NVARCHAR(MAX);
DECLARE @pkCols NVARCHAR(MAX);

-- Get list of primary key columns
SELECT @pkCols = STRING_AGG(QUOTENAME(c.COLUMN_NAME), ',')
FROM INFORMATION_SCHEMA.TABLE_CONSTRAINTS tc
JOIN INFORMATION_SCHEMA.KEY_COLUMN_USAGE c
  ON c.CONSTRAINT_NAME = tc.CONSTRAINT_NAME
WHERE tc.TABLE_NAME = @TableName
  AND tc.TABLE_SCHEMA = @SchemaName
  AND tc.CONSTRAINT_TYPE = 'PRIMARY KEY';

-- Generate SET clause for non-PK columns
SELECT @cols = STRING_AGG(
    QUOTENAME(c.COLUMN_NAME) + ' = @' + c.COLUMN_NAME,
    ', '
)
FROM INFORMATION_SCHEMA.COLUMNS c
WHERE c.TABLE_NAME = @TableName
  AND c.TABLE_SCHEMA = @SchemaName
  AND c.COLUMN_NAME NOT IN (
    SELECT k.COLUMN_NAME
    FROM INFORMATION_SCHEMA.TABLE_CONSTRAINTS t
    JOIN INFORMATION_SCHEMA.KEY_COLUMN_USAGE k
      ON k.CONSTRAINT_NAME = t.CONSTRAINT_NAME
    WHERE t.TABLE_NAME = @TableName
      AND t.TABLE_SCHEMA = @SchemaName
      AND t.CONSTRAINT_TYPE = 'PRIMARY KEY'
  );

SET @sql = 'UPDATE ' + QUOTENAME(@SchemaName) + '.' + QUOTENAME(@TableName) + CHAR(13) +
           'SET ' + @cols + CHAR(13) +
           'WHERE ' + ISNULL(REPLACE(@pkCols, ',', ' = @ AND ') + ' = @', '-- No PK found, please specify WHERE manually');

PRINT @sql;
```
