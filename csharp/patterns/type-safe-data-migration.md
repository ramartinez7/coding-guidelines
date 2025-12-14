# Type-Safe Data Migration (Schema Versioning)

> Database migrations with version numbers and SQL scripts fail silently—use typed migrations to enforce schema evolution at compile time.

## Problem

Database schema changes tracked with version numbers and SQL scripts can be applied out of order, skipped, or incompatible. The compiler can't verify that migrations are complete or that schema changes are compatible with code changes.

## Example

### ❌ Before

```csharp
public class Migration001_CreateUsersTable : Migration
{
    public override void Up()
    {
        Create.Table("Users")
            .WithColumn("Id").AsInt32().PrimaryKey()
            .WithColumn("Name").AsString();
    }
}

// Later migration can break assumptions
public class Migration002_DropNameColumn : Migration
{
    public override void Up()
    {
        Delete.Column("Name").FromTable("Users");  // Code still expects Name!
    }
}
```

### ✅ After

```csharp
public interface ISchemaVersion { }
public interface IV1 : ISchemaVersion { }
public interface IV2 : ISchemaVersion { }

public sealed record UserTableV1(int Id, string Name);
public sealed record UserTableV2(int Id, string FirstName, string LastName);

public interface IMigration<TFrom, TTo>
    where TFrom : ISchemaVersion
    where TTo : ISchemaVersion
{
    Task MigrateAsync();
}

public sealed class UserTableMigrationV1ToV2 : IMigration<IV1, IV2>
{
    public async Task MigrateAsync()
    {
        // Type-safe migration logic
        var users = await _db.Query<UserTableV1>("SELECT * FROM Users");
        
        foreach (var user in users)
        {
            var names = user.Name.Split(' ');
            await _db.Execute(
                "UPDATE Users SET FirstName = @first, LastName = @last WHERE Id = @id",
                new { first = names[0], last = names.Length > 1 ? names[1] : "", id = user.Id });
        }
    }
}
```

## See Also

- [Type-Safe Serialization Contracts](./type-safe-serialization-contracts.md)
- [Phantom Types](./phantom-types.md)
