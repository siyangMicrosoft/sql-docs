---
title: "Tutorial: Creating and using indexes on enclave-enabled columns using randomized encryption | Microsoft Docs"
ms.custom: ""
ms.date: 06/26/2019
ms.prod: sql
ms.prod_service: "database-engine, sql-database"
ms.reviewer: vanto
ms.suite: "sql"
ms.technology: security
ms.tgt_pltfrm: ""
ms.topic: tutorial
author: jaszymas
ms.author: jaszymas
monikerRange: ">= sql-server-ver15 || = sqlallproducts-allversions"
---
# Tutorial: Creating and using indexes on enclave-enabled columns using randomized encryption
[!INCLUDE [tsql-appliesto-ssver15-xxxx-xxxx-xxx](../../includes/tsql-appliesto-ssver15-xxxx-xxxx-xxx.md)]

This tutorial teaches you how to create and use indexes on enclave-enabled columns using randomized encryption supported in [Always Encrypted with secure enclaves](encryption/always-encrypted-enclaves.md). It will show you:

- How to create an index when you have access to the keys (the column master key and the column encryption key) protecting the column.
- How to create an index when you don't have access to the keys protecting the column.

## Prerequisites

This tutorial is the continuation of [Tutorial: Getting Started with Always Encrypted with secure enclaves using SSMS](./tutorial-getting-started-with-always-encrypted-enclaves.md). Make sure you've completed it, before following the below steps.

## Step 1: Enable Accelerated Database Recovery (ADR) in your database

Microsoft strongly recommends you enable ADR in your database before creating the first index on an enclave-enabled column using randomized encryption. See the [Database Recovery](./encryption/always-encrypted-enclaves.md##database-recovery) section in [Always Encrypted with Secure Enclaves](./encryption/always-encrypted-enclaves.md).

1. Close any SSMS instances, you used in the previous tutorial. This will close database connections you've opened, which is required to enable ADR.
1. Open a new instance of SSMS and connect to your SQL Server instance as sysadmin **without** Always Encrypted enabled for the database connection.
    1. Start SSMS.
    1. In the **Connect to Server** dialog, specify your server name, select an authentication method, and specify your credentials.
    1. Click **Options >>** and select the **Always Encrypted** tab.
    1. Make sure the **Enable Always Encrypted (column encryption)** checkbox is **not** selected.
    1. Select **Connect**.
1. Open a new query window and execute the below statement to enable ADR.

   ```sql
   ALTER DATABASE ContosoHR SET ACCELERATED_DATABASE_RECOVERY = ON;
   ```

## Step 2: Create and test an index without role separation

In this step, you'll create and test an index on an encrypted column. You'll be acting as a single user who is assuming the roles of both a DBA, who manages the database, and the data owner who has access to the keys, protecting the data.

1. Open a new SSMS instance and connect to your SQL Server instance **with** Always Encrypted enabled for the database connection.
   1. Start a new instance of SSMS.
   1. In the **Connect to Server** dialog, specify your server name, select an   authentication method, and specify your credentials.
   1. Click **Options >>** and select the **Always Encrypted** tab.
   1. Select the **Enable Always Encrypted (column encryption)** checkbox and   specify your enclave attestation URL (for example, ht<span>tp://<   span>hgs.bastion.local/Attestation).
   1. Select **Connect**.
   1. If prompted to enable parameterization for Always Encrypted queries, click **Enable**.
1. If you weren't prompted to enable Parameterization for Always Encrypted, verify it's enabled.
   1. Select **Tools** from the main menu of SSMS.
   2. Select **Options...**.
   3. Navigate to **Query Execution** > **SQL Server** > **Advanced**.
   4. Ensure that **Enable Parameterization for Always Encrypted** is checked.
   5. Select **OK**.
1. Open a query window and execute the below statements to encrypt the **LastName** column in the **Employees** table. You'll create and use an index on that column in later steps.

   ```sql
   USE [ContosoHR];
   GO   
   ALTER TABLE [dbo].[Employees]
   ALTER COLUMN [LastName] [nvarchar](50) COLLATE Latin1_General_BIN2 
   ENCRYPTED WITH (COLUMN_ENCRYPTION_KEY = [CEK1], ENCRYPTION_TYPE = Randomized    ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256') NOT NULL;
   GO   
   ALTER DATABASE SCOPED CONFIGURATION CLEAR PROCEDURE_CACHE;
   GO
   ```

1. Create an index on the **LastName** column. Since you're connected to the database with Always Encrypted enabled, the client driver inside SSMS transparently provides **CEK1** (the column encryption key, protecting the **LastName** column) to the enclave, which is needed to create the index.

   ```sql
   USE [ContosoHR];
   GO

   CREATE INDEX IX_LastName ON [Employees] ([LastName])
   INCLUDE ([EmployeeID], [FirstName], [SSN], [Salary]);
   GO
   ```

1. Run a rich query on the **LastName** column and verify SQL Server uses the index when executing the query.
   1. In the same or a new query window, make sure the **Include Live Query Statistics** button on the toolbar is on.
   1. Execute the below query.

       ```sql
       USE [ContosoHR];
       GO

       DECLARE @LastNamePrefix NVARCHAR(50) = 'Aber%';
       SELECT * FROM [dbo].[Employees] WHERE [LastName] LIKE @LastNamePrefix;
       GO
      ```

   1. In the **Live Query Statistics** tab (in the bottom part of the query window), observe that the query uses the index.

## Step 3: Create an index with role separation

In this step, you'll create an index on an encrypted column, pretending to be two different users. One user is a DBA, who needs to create an index, but doesn't have access to the keys. The other user is a data owner, who has access to the keys.

1. Using the SSMS instance **without** Always Encrypted enabled, execute the below statement to drop the index on the **LastName** column.

   ```sql
   USE [ContosoHR];
   GO

   DROP INDEX IX_LastName ON [Employees]; 
   GO
   ```

1. Acting as a data owner (or an application that has access to the keys), populate the cache inside the enclave with **CEK1**.

   > [!NOTE]
   > Unless you have restarted your SQL Server instance after **Step 2: Create and test an index without role separation**, this step is redundant as the **CEK1** is already present in the cache. We have added it to demonstrate how a data owner can provide a key to the enclave, if it is not already present in the enclave.

   1. In the SSMS instance **with** Always Encrypted enabled, execute the below statements in a query window. The statement sends all enclave-enabled column encryption keys to the enclave. See [sp_enclave_send_keys](../system-stored-procedures/sp-enclave-send-keys-sql.md) for details.

        ```sql
        USE [ContosoHR];
        GO

        EXEC sp_enclave_send_keys;
        GO
        ```

   1. As an alternative to executing the above stored procedure, you can run a DML query that uses the enclave against the **LastName** column. This will populate the enclave only with **CEK1**.

        ```sql
        USE [ContosoHR];
        GO

        DECLARE @LastNamePrefix NVARCHAR(50) = 'Aber%';
        SELECT * FROM [dbo].[Employees] WHERE [LastName] LIKE @LastNamePrefix;
        GO
        ```

1. Acting as a DBA, create the index.
    1. In the SSMS instance **without** Always Encrypted enabled, execute the below statements in a query window.

        ```sql
        USE [ContosoHR];
        GO

        CREATE INDEX IX_LastName ON [Employees] ([LastName])
        INCLUDE ([EmployeeID], [FirstName], [SSN], [Salary]);
        GO
        ```

1. As a data owner, run a rich query on the **LastName** column and verify SQL Server uses the index when executing the query.
   1. In the SSMS instance **with** Always Encrypted enabled, select an existing query window or open a new query window, and make sure the **Include Live Query Statistics** button on the toolbar is on.
   1. Execute the below query. 

        ```sql
        USE [ContosoHR];
        GO

        DECLARE @LastNamePrefix NVARCHAR(50) = 'Aber%';
        SELECT * FROM [dbo].[Employees] WHERE [LastName] LIKE @LastNamePrefix;
        GO
        ```

   1. In the **Live Query Statistics** (in the bottom part of the query window), observe that the query uses the index.

## Next steps

- See [Configure Always Encrypted with secure enclaves](encryption/configure-always-encrypted-enclaves.md) for information on other use cases for Always Encrypted with secure enclaves.
