# 🛠️ Script Auto-Generate UPDATE TABLE (non-PK columns only)
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

# 🛠️ Script — Auto-generate column copy SET list (excluding PK)
```
DECLARE @TableName NVARCHAR(128) = 'YourTableName'; -- 👈 your table name
DECLARE @SchemaName NVARCHAR(128) = 'dbo';          -- 👈 your schema

DECLARE @sql NVARCHAR(MAX);
DECLARE @setCols NVARCHAR(MAX);

-- Get list of primary key columns
WITH PK_Cols AS (
    SELECT k.COLUMN_NAME
    FROM INFORMATION_SCHEMA.TABLE_CONSTRAINTS t
    JOIN INFORMATION_SCHEMA.KEY_COLUMN_USAGE k
      ON k.CONSTRAINT_NAME = t.CONSTRAINT_NAME
    WHERE t.TABLE_NAME = @TableName
      AND t.TABLE_SCHEMA = @SchemaName
      AND t.CONSTRAINT_TYPE = 'PRIMARY KEY'
)

-- Generate SET clause for non-PK columns
SELECT @setCols = STRING_AGG(
    'target.' + QUOTENAME(c.COLUMN_NAME) + ' = source.' + QUOTENAME(c.COLUMN_NAME),
    ', ' + CHAR(13)
)
FROM INFORMATION_SCHEMA.COLUMNS c
WHERE c.TABLE_NAME = @TableName
  AND c.TABLE_SCHEMA = @SchemaName
  AND c.COLUMN_NAME NOT IN (SELECT COLUMN_NAME FROM PK_Cols);

-- Output result
SET @sql = 'SET ' + CHAR(13) + @setCols;
PRINT @sql;
```

## For Example
```
UPDATE target
SET
    target.Col1 = source.Col1,
    target.Col2 = source.Col2,
    target.Col3 = source.Col3
FROM YourTable AS target
CROSS JOIN YourTable AS source
WHERE target.Id = 2
  AND source.Id = 1;
```

## Real Example

### Folder Structure
```
--272	02 ERMS
--273		-> Documents
--274		-> Source
```

### T-SQL Copy all columns' value of ID = 272 to ID = 273
```
UPDATE target
SET
target.[vcName] = source.[vcName], 
--target.[iParentID] = 272, 
target.[iMode] = source.[iMode], 
target.[iAcl] = source.[iAcl], 
target.[iType] = source.[iType], 
target.[dtCreation] = source.[dtCreation], 
target.[uidOwner] = source.[uidOwner], 
target.[vcDescription] = source.[vcDescription], 
target.[dtDelete] = source.[dtDelete], 
target.[uidDeleteBy] = source.[uidDeleteBy], 
target.[cust1] = source.[cust1], 
target.[MailListBroadcast] = source.[MailListBroadcast], 
target.[iAccType] = source.[iAccType], 
target.[vcUrl] = source.[vcUrl], 
target.[Flags] = source.[Flags], 
target.[dtModify] = source.[dtModify], 
target.[dtModify2] = source.[dtModify2], 
target.[uidModifiedBy2] = source.[uidModifiedBy2], 
target.[DispositionAction] = source.[DispositionAction], 
target.[RetentionPeriodStartDate] = source.[RetentionPeriodStartDate], 
target.[Source_WkSpcID] = source.[Source_WkSpcID], 
target.[NASTransfer] = source.[NASTransfer], 
target.[DispositionReason] = source.[DispositionReason], 
target.[DispositionReasonID] = source.[DispositionReasonID], 
target.[InternalAgencyID] = source.[InternalAgencyID], 
target.[AutoCloseDate] = source.[AutoCloseDate], 
target.[FRPeriod] = source.[FRPeriod], 
target.[FRPeriodType] = source.[FRPeriodType], 
target.[StubFlag] = source.[StubFlag], 
target.[uidPersonal] = source.[uidPersonal], 
target.[uidOldPersonal] = source.[uidOldPersonal], 
target.[cust2] = source.[cust2], 
target.[cust3] = source.[cust3], 
target.[cust4] = source.[cust4], 
target.[custID1] = source.[custID1], 
target.[CreatedBy] = source.[CreatedBy], 
target.[CreatedDateTime] = source.[CreatedDateTime], 
target.[CreatedOnBehalfOf] = source.[CreatedOnBehalfOf], 
target.[FolderRefSeq] = source.[FolderRefSeq], 
target.[DisposedFlag] = source.[DisposedFlag], 
target.[DisposedRMJobID] = source.[DisposedRMJobID], 
target.[RelativeUrl] = source.[RelativeUrl], 
target.[iRMWkSpcID] = source.[iRMWkSpcID], 
target.[AutoCreateUponClose] = source.[AutoCreateUponClose]
FROM workspaces AS target
CROSS JOIN workspaces AS source
WHERE target.iWkSpcID = 273
  AND source.iWkSpcID = 272;
```

# 🛠️ Full Auto-generate T-SQL — Copy all columns (excluding PK)
```
DECLARE @TableName NVARCHAR(128) = 'YourTableName'; -- 👈 your table
DECLARE @SchemaName NVARCHAR(128) = 'dbo';          -- 👈 your schema
DECLARE @SourceId INT = 1;                           -- 👈 copy FROM Id
DECLARE @TargetId INT = 2;                           -- 👈 copy TO Id

DECLARE @sql NVARCHAR(MAX);
DECLARE @setCols NVARCHAR(MAX);
DECLARE @pkCol NVARCHAR(128);

-- Get PK column (assumes single-column PK)
SELECT TOP 1 @pkCol = k.COLUMN_NAME
FROM INFORMATION_SCHEMA.TABLE_CONSTRAINTS t
JOIN INFORMATION_SCHEMA.KEY_COLUMN_USAGE k
  ON k.CONSTRAINT_NAME = t.CONSTRAINT_NAME
WHERE t.TABLE_NAME = @TableName
  AND t.TABLE_SCHEMA = @SchemaName
  AND t.CONSTRAINT_TYPE = 'PRIMARY KEY';

-- Generate SET clause for all non-PK columns
SELECT @setCols = STRING_AGG(
    '    target.' + QUOTENAME(c.COLUMN_NAME) + ' = source.' + QUOTENAME(c.COLUMN_NAME),
    ',' + CHAR(13)
)
FROM INFORMATION_SCHEMA.COLUMNS c
WHERE c.TABLE_NAME = @TableName
  AND c.TABLE_SCHEMA = @SchemaName
  AND c.COLUMN_NAME <> @pkCol;

-- Build full UPDATE statement
SET @sql = 'UPDATE target' + CHAR(13) +
           'SET' + CHAR(13) +
           @setCols + CHAR(13) +
           'FROM ' + QUOTENAME(@SchemaName) + '.' + QUOTENAME(@TableName) + ' AS target' + CHAR(13) +
           'CROSS JOIN ' + QUOTENAME(@SchemaName) + '.' + QUOTENAME(@TableName) + ' AS source' + CHAR(13) +
           'WHERE target.' + QUOTENAME(@pkCol) + ' = ' + CAST(@TargetId AS NVARCHAR) + CHAR(13) +
           '  AND source.' + QUOTENAME(@pkCol) + ' = ' + CAST(@SourceId AS NVARCHAR) + ';';

PRINT @sql;
```
