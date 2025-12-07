# Path Traversal Prevention (Safe File Paths)

> Using unsanitized user input in file paths—use validated path types to prevent directory traversal attacks.

## Problem

Path traversal (directory traversal) attacks occur when user-supplied input is used to construct file paths without validation. Attackers can use sequences like `../` to access files outside the intended directory. String-based paths provide no compile-time protection against these attacks.

## Example

### ❌ Before

```csharp
public class FileService
{
    private readonly string _uploadDirectory = "/var/app/uploads";
    
    public byte[] GetUserFile(string userId, string filename)
    {
        // Dangerous: path traversal vulnerability!
        // Attacker input: "../../../etc/passwd"
        var filePath = Path.Combine(_uploadDirectory, userId, filename);
        
        return File.ReadAllBytes(filePath);
        // Reads: /var/app/uploads/user123/../../../etc/passwd
        // Which resolves to: /etc/passwd
    }
    
    public void SaveUploadedFile(string filename, Stream content)
    {
        // Vulnerable: filename could be "../../malicious.exe"
        var filePath = Path.Combine(_uploadDirectory, filename);
        
        using var fileStream = File.Create(filePath);
        content.CopyTo(fileStream);
    }
    
    public string ReadTemplate(string templateName)
    {
        // Dangerous: templateName could include path traversal
        var templatePath = $"/var/app/templates/{templateName}.html";
        return File.ReadAllText(templatePath);
    }
}
```

**Problems:**
- User input directly used in file paths
- No validation of path traversal sequences
- `Path.Combine` doesn't prevent traversal
- No compile-time detection
- Can access sensitive system files

### ✅ After

```csharp
/// <summary>
/// A validated, safe file path that cannot traverse directories.
/// Cannot be constructed from untrusted strings without validation.
/// </summary>
public sealed record SafeFilePath
{
    public string FullPath { get; }
    public string FileName { get; }
    
    private SafeFilePath(string fullPath, string fileName)
    {
        FullPath = fullPath;
        FileName = fileName;
    }
    
    /// <summary>
    /// Creates a safe file path within a base directory.
    /// Validates that the resulting path doesn't escape the base directory.
    /// </summary>
    public static Result<SafeFilePath, PathValidationError> Create(
        string baseDirectory,
        params string[] pathSegments)
    {
        if (string.IsNullOrWhiteSpace(baseDirectory))
            return Result<SafeFilePath, PathValidationError>.Failure(
                new PathValidationError("Base directory cannot be empty"));
        
        if (pathSegments.Length == 0)
            return Result<SafeFilePath, PathValidationError>.Failure(
                new PathValidationError("Path segments required"));
        
        // Validate each segment
        foreach (var segment in pathSegments)
        {
            if (string.IsNullOrWhiteSpace(segment))
                return Result<SafeFilePath, PathValidationError>.Failure(
                    new PathValidationError("Path segment cannot be empty"));
            
            // Check for path traversal attempts
            if (segment.Contains(".."))
                return Result<SafeFilePath, PathValidationError>.Failure(
                    new PathValidationError("Path traversal detected: '..' not allowed"));
            
            // Check for absolute path attempts
            if (Path.IsPathRooted(segment))
                return Result<SafeFilePath, PathValidationError>.Failure(
                    new PathValidationError("Absolute paths not allowed"));
            
            // Check for invalid characters
            var invalidChars = Path.GetInvalidFileNameChars();
            if (segment.IndexOfAny(invalidChars) >= 0)
                return Result<SafeFilePath, PathValidationError>.Failure(
                    new PathValidationError($"Invalid characters in path segment: {segment}"));
        }
        
        // Combine path segments
        var combinedPath = Path.Combine(new[] { baseDirectory }.Concat(pathSegments).ToArray());
        
        // Get the full, normalized path
        var fullPath = Path.GetFullPath(combinedPath);
        var baseFullPath = Path.GetFullPath(baseDirectory);
        
        // Verify the resulting path is within the base directory
        if (!fullPath.StartsWith(baseFullPath, StringComparison.OrdinalIgnoreCase))
            return Result<SafeFilePath, PathValidationError>.Failure(
                new PathValidationError("Path escapes base directory"));
        
        var fileName = Path.GetFileName(fullPath);
        return Result<SafeFilePath, PathValidationError>.Success(
            new SafeFilePath(fullPath, fileName));
    }
    
    /// <summary>
    /// Creates a safe file path from a single filename in a base directory.
    /// </summary>
    public static Result<SafeFilePath, PathValidationError> FromFileName(
        string baseDirectory,
        string fileName)
    {
        return Create(baseDirectory, fileName);
    }
}

public sealed record PathValidationError(string Message);

/// <summary>
/// Represents a validated base directory for file operations.
/// </summary>
public sealed record BaseDirectory
{
    public string Path { get; }
    
    private BaseDirectory(string path) => Path = path;
    
    public static Result<BaseDirectory, string> Create(string path)
    {
        if (string.IsNullOrWhiteSpace(path))
            return Result<BaseDirectory, string>.Failure("Directory path cannot be empty");
        
        if (!Directory.Exists(path))
            return Result<BaseDirectory, string>.Failure($"Directory does not exist: {path}");
        
        var fullPath = Path.GetFullPath(path);
        return Result<BaseDirectory, string>.Success(new BaseDirectory(fullPath));
    }
    
    public Result<SafeFilePath, PathValidationError> GetFilePath(params string[] pathSegments)
    {
        return SafeFilePath.Create(Path, pathSegments);
    }
}

public class FileService
{
    private readonly BaseDirectory _uploadDirectory;
    
    public FileService(BaseDirectory uploadDirectory)
    {
        _uploadDirectory = uploadDirectory;
    }
    
    public Result<byte[], string> GetUserFile(UserId userId, string filename)
    {
        // Create safe path—traversal attempts will fail
        var pathResult = _uploadDirectory.GetFilePath(userId.Value.ToString(), filename);
        
        return pathResult.Match(
            onSuccess: safePath =>
            {
                if (!File.Exists(safePath.FullPath))
                    return Result<byte[], string>.Failure("File not found");
                
                var content = File.ReadAllBytes(safePath.FullPath);
                return Result<byte[], string>.Success(content);
            },
            onFailure: error => Result<byte[], string>.Failure(error.Message));
    }
    
    public Result<Unit, string> SaveUploadedFile(string filename, Stream content)
    {
        // Filename is validated—path traversal impossible
        var pathResult = SafeFilePath.FromFileName(_uploadDirectory.Path, filename);
        
        return pathResult.Match(
            onSuccess: safePath =>
            {
                using var fileStream = File.Create(safePath.FullPath);
                content.CopyTo(fileStream);
                return Result<Unit, string>.Success(Unit.Value);
            },
            onFailure: error => Result<Unit, string>.Failure(error.Message));
    }
}
```

## Advanced Patterns

### File Extension Validation

```csharp
/// <summary>
/// Allowed file extensions for uploads.
/// </summary>
public sealed record AllowedExtension
{
    public string Extension { get; }
    
    private AllowedExtension(string extension) => Extension = extension;
    
    public static readonly AllowedExtension Jpg = new(".jpg");
    public static readonly AllowedExtension Png = new(".png");
    public static readonly AllowedExtension Pdf = new(".pdf");
    public static readonly AllowedExtension Txt = new(".txt");
    
    private static readonly HashSet<string> ValidExtensions = new()
    {
        ".jpg", ".jpeg", ".png", ".gif", ".pdf", ".txt", ".doc", ".docx"
    };
    
    public static Result<AllowedExtension, string> Create(string extension)
    {
        if (string.IsNullOrWhiteSpace(extension))
            return Result<AllowedExtension, string>.Failure("Extension cannot be empty");
        
        var normalized = extension.ToLowerInvariant();
        if (!normalized.StartsWith("."))
            normalized = "." + normalized;
        
        if (!ValidExtensions.Contains(normalized))
            return Result<AllowedExtension, string>.Failure(
                $"Extension '{extension}' is not allowed");
        
        return Result<AllowedExtension, string>.Success(new AllowedExtension(normalized));
    }
}

/// <summary>
/// A validated filename with allowed extension.
/// </summary>
public sealed record ValidatedFileName
{
    public string Name { get; }
    public AllowedExtension Extension { get; }
    public string FullName => Name + Extension.Extension;
    
    private ValidatedFileName(string name, AllowedExtension extension)
    {
        Name = name;
        Extension = extension;
    }
    
    public static Result<ValidatedFileName, string> Create(string filename)
    {
        if (string.IsNullOrWhiteSpace(filename))
            return Result<ValidatedFileName, string>.Failure("Filename cannot be empty");
        
        // Remove any path components
        var nameOnly = Path.GetFileName(filename);
        
        // Check for path traversal
        if (nameOnly.Contains(".."))
            return Result<ValidatedFileName, string>.Failure("Path traversal not allowed");
        
        var extension = Path.GetExtension(nameOnly);
        var nameWithoutExtension = Path.GetFileNameWithoutExtension(nameOnly);
        
        if (string.IsNullOrWhiteSpace(nameWithoutExtension))
            return Result<ValidatedFileName, string>.Failure("Filename cannot be empty");
        
        var extensionResult = AllowedExtension.Create(extension);
        
        return extensionResult.Match(
            onSuccess: ext => Result<ValidatedFileName, string>.Success(
                new ValidatedFileName(nameWithoutExtension, ext)),
            onFailure: error => Result<ValidatedFileName, string>.Failure(error));
    }
}

public class SecureFileService
{
    public Result<Unit, string> SaveUpload(ValidatedFileName filename, Stream content)
    {
        var pathResult = _uploadDirectory.GetFilePath(filename.FullName);
        
        return pathResult.Match(
            onSuccess: safePath =>
            {
                // Additional MIME type validation
                var mimeType = GetMimeType(content);
                if (!IsAllowedMimeType(mimeType, filename.Extension))
                    return Result<Unit, string>.Failure("MIME type doesn't match extension");
                
                using var fileStream = File.Create(safePath.FullPath);
                content.CopyTo(fileStream);
                return Result<Unit, string>.Success(Unit.Value);
            },
            onFailure: error => Result<Unit, string>.Failure(error.Message));
    }
    
    private bool IsAllowedMimeType(string mimeType, AllowedExtension extension)
    {
        return extension.Extension switch
        {
            ".jpg" or ".jpeg" => mimeType == "image/jpeg",
            ".png" => mimeType == "image/png",
            ".pdf" => mimeType == "application/pdf",
            ".txt" => mimeType == "text/plain",
            _ => false
        };
    }
}
```

### Directory Jail Pattern

```csharp
/// <summary>
/// A "jailed" file system that restricts operations to a specific directory.
/// </summary>
public sealed class JailedFileSystem
{
    private readonly BaseDirectory _jailRoot;
    
    public JailedFileSystem(BaseDirectory jailRoot)
    {
        _jailRoot = jailRoot;
    }
    
    public Result<string, string> ReadFile(params string[] pathSegments)
    {
        var pathResult = _jailRoot.GetFilePath(pathSegments);
        
        return pathResult.Match(
            onSuccess: safePath =>
            {
                if (!File.Exists(safePath.FullPath))
                    return Result<string, string>.Failure("File not found");
                
                var content = File.ReadAllText(safePath.FullPath);
                return Result<string, string>.Success(content);
            },
            onFailure: error => Result<string, string>.Failure(error.Message));
    }
    
    public Result<Unit, string> WriteFile(string content, params string[] pathSegments)
    {
        var pathResult = _jailRoot.GetFilePath(pathSegments);
        
        return pathResult.Match(
            onSuccess: safePath =>
            {
                // Ensure parent directory exists
                var directory = Path.GetDirectoryName(safePath.FullPath);
                if (directory != null && !Directory.Exists(directory))
                    Directory.CreateDirectory(directory);
                
                File.WriteAllText(safePath.FullPath, content);
                return Result<Unit, string>.Success(Unit.Value);
            },
            onFailure: error => Result<Unit, string>.Failure(error.Message));
    }
    
    public Result<bool, string> FileExists(params string[] pathSegments)
    {
        var pathResult = _jailRoot.GetFilePath(pathSegments);
        
        return pathResult.Match(
            onSuccess: safePath => Result<bool, string>.Success(File.Exists(safePath.FullPath)),
            onFailure: error => Result<bool, string>.Failure(error.Message));
    }
    
    public Result<Unit, string> DeleteFile(params string[] pathSegments)
    {
        var pathResult = _jailRoot.GetFilePath(pathSegments);
        
        return pathResult.Match(
            onSuccess: safePath =>
            {
                if (!File.Exists(safePath.FullPath))
                    return Result<Unit, string>.Failure("File not found");
                
                File.Delete(safePath.FullPath);
                return Result<Unit, string>.Success(Unit.Value);
            },
            onFailure: error => Result<Unit, string>.Failure(error.Message));
    }
    
    public Result<List<string>, string> ListFiles(params string[] pathSegments)
    {
        var pathResult = pathSegments.Length == 0
            ? Result<SafeFilePath, PathValidationError>.Success(
                new SafeFilePath(_jailRoot.Path, ""))
            : _jailRoot.GetFilePath(pathSegments);
        
        return pathResult.Match(
            onSuccess: safePath =>
            {
                var directory = pathSegments.Length == 0 
                    ? _jailRoot.Path 
                    : Path.GetDirectoryName(safePath.FullPath);
                
                if (directory == null || !Directory.Exists(directory))
                    return Result<List<string>, string>.Failure("Directory not found");
                
                var files = Directory.GetFiles(directory)
                    .Select(Path.GetFileName)
                    .Where(f => f != null)
                    .Cast<string>()
                    .ToList();
                
                return Result<List<string>, string>.Success(files);
            },
            onFailure: error => Result<List<string>, string>.Failure(error.Message));
    }
}

// Usage
public class DocumentService
{
    private readonly JailedFileSystem _userDocuments;
    
    public DocumentService(BaseDirectory documentsRoot)
    {
        _userDocuments = new JailedFileSystem(documentsRoot);
    }
    
    public Result<string, string> GetUserDocument(UserId userId, string documentName)
    {
        // All operations are jailed to the documents root
        // Path traversal attempts will fail at the SafeFilePath creation
        return _userDocuments.ReadFile(userId.Value.ToString(), documentName);
    }
    
    public Result<Unit, string> SaveUserDocument(
        UserId userId, 
        string documentName, 
        string content)
    {
        return _userDocuments.WriteFile(content, userId.Value.ToString(), documentName);
    }
}
```

### Configuration-Based Base Directories

```csharp
public sealed class FileSystemConfiguration
{
    public Dictionary<string, BaseDirectory> Directories { get; } = new();
    
    public static Result<FileSystemConfiguration, string> Load(IConfiguration configuration)
    {
        var config = new FileSystemConfiguration();
        
        var uploadPath = configuration["FileSystem:UploadDirectory"];
        var uploadDirResult = BaseDirectory.Create(uploadPath ?? "/var/app/uploads");
        
        if (!uploadDirResult.IsSuccess)
            return Result<FileSystemConfiguration, string>.Failure(
                $"Invalid upload directory: {uploadDirResult.Error}");
        
        config.Directories["uploads"] = uploadDirResult.Value;
        
        var templatePath = configuration["FileSystem:TemplateDirectory"];
        var templateDirResult = BaseDirectory.Create(templatePath ?? "/var/app/templates");
        
        if (!templateDirResult.IsSuccess)
            return Result<FileSystemConfiguration, string>.Failure(
                $"Invalid template directory: {templateDirResult.Error}");
        
        config.Directories["templates"] = templateDirResult.Value;
        
        return Result<FileSystemConfiguration, string>.Success(config);
    }
    
    public Result<BaseDirectory, string> GetDirectory(string name)
    {
        if (!Directories.TryGetValue(name, out var directory))
            return Result<BaseDirectory, string>.Failure($"Directory '{name}' not configured");
        
        return Result<BaseDirectory, string>.Success(directory);
    }
}

// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    var fsConfigResult = FileSystemConfiguration.Load(Configuration);
    
    if (!fsConfigResult.IsSuccess)
        throw new InvalidOperationException(
            $"Failed to load file system configuration: {fsConfigResult.Error}");
    
    services.AddSingleton(fsConfigResult.Value);
}
```

## Testing

```csharp
public class PathTraversalTests
{
    private readonly BaseDirectory _testDirectory;
    
    public PathTraversalTests()
    {
        var tempPath = Path.Combine(Path.GetTempPath(), Guid.NewGuid().ToString());
        Directory.CreateDirectory(tempPath);
        _testDirectory = BaseDirectory.Create(tempPath).Value;
    }
    
    [Fact]
    public void SafeFilePath_WithTraversalAttempt_Fails()
    {
        var result = SafeFilePath.Create(_testDirectory.Path, "../../../etc/passwd");
        
        Assert.False(result.IsSuccess);
        Assert.Contains("traversal", result.Error!.Message.ToLower());
    }
    
    [Fact]
    public void SafeFilePath_WithAbsolutePath_Fails()
    {
        var result = SafeFilePath.Create(_testDirectory.Path, "/etc/passwd");
        
        Assert.False(result.IsSuccess);
        Assert.Contains("absolute", result.Error!.Message.ToLower());
    }
    
    [Fact]
    public void SafeFilePath_WithValidPath_Succeeds()
    {
        var result = SafeFilePath.Create(_testDirectory.Path, "user123", "document.pdf");
        
        Assert.True(result.IsSuccess);
        Assert.StartsWith(_testDirectory.Path, result.Value!.FullPath);
    }
    
    [Fact]
    public void SafeFilePath_WithEncodedTraversal_Fails()
    {
        // URL-encoded ../ is %2E%2E%2F
        var result = SafeFilePath.Create(_testDirectory.Path, "%2E%2E%2F", "file.txt");
        
        // Should fail because it contains ".."
        Assert.False(result.IsSuccess);
    }
    
    [Fact]
    public void JailedFileSystem_CannotEscapeJail()
    {
        var fs = new JailedFileSystem(_testDirectory);
        
        var result = fs.ReadFile("../../etc/passwd");
        
        Assert.False(result.IsSuccess);
    }
}
```

## Why It's a Problem

1. **Unauthorized file access**: Attackers can read sensitive system files
2. **Data exfiltration**: Access to configuration files, credentials, source code
3. **System compromise**: Reading `/etc/passwd` or SSH keys
4. **File overwrite**: Writing to arbitrary locations
5. **No compile-time detection**: String paths look innocent

## Symptoms

- User input directly used in `Path.Combine()` or string concatenation
- No validation of `..` sequences in file paths
- File operations using unsanitized filenames
- Security vulnerabilities in file upload/download features
- Access to files outside intended directories

## Benefits

- **Traversal-proof**: Path validation prevents directory traversal attacks
- **Compile-time safety**: Type system enforces validated paths
- **Self-documenting**: `SafeFilePath` makes security constraints explicit
- **Jailed operations**: Restrict all file operations to specific directories
- **Extension validation**: Prevent upload of dangerous file types

## Trade-offs

- **More verbose**: Path creation requires explicit validation
- **Result types**: Must handle validation failures
- **Directory setup**: Base directories must be configured at startup
- **Testing complexity**: Tests need temporary directories

## See Also

- [Input Sanitization](./input-sanitization.md) — trusted types at boundaries
- [Validated Configuration](./validated-configuration.md) — startup validation
- [Honest Functions](./honest-functions.md) — explicit error handling
- [Command Injection Prevention](./command-injection-prevention.md) — similar validation pattern
