# Command Injection Prevention (Safe Process Execution)

> Executing shell commands with untrusted input—use validated command types to prevent command injection attacks.

## Problem

Command injection occurs when untrusted input is passed to shell commands without proper validation. Attackers can inject shell metacharacters to execute arbitrary commands. String-based process execution provides no compile-time protection against these attacks.

## Example

### ❌ Before

```csharp
public class ImageProcessingService
{
    public void ConvertImage(string inputFile, string outputFile)
    {
        // Dangerous: command injection vulnerability!
        // Attacker input: "input.jpg; rm -rf /"
        var command = $"convert {inputFile} {outputFile}";
        
        var process = new Process
        {
            StartInfo = new ProcessStartInfo
            {
                FileName = "/bin/bash",
                Arguments = $"-c \"{command}\"",
                RedirectStandardOutput = true,
                UseShellExecute = false
            }
        };
        
        process.Start();
        process.WaitForExit();
    }
    
    public void CompressFile(string filename)
    {
        // Vulnerable: filename could be "file.txt && cat /etc/passwd"
        var command = $"gzip {filename}";
        
        Process.Start("/bin/sh", $"-c \"{command}\"");
    }
    
    public string ListDirectory(string directory)
    {
        // Dangerous: directory could be ". && rm -rf /"
        var process = new Process
        {
            StartInfo = new ProcessStartInfo("cmd.exe", $"/c dir {directory}")
            {
                RedirectStandardOutput = true,
                UseShellExecute = false
            }
        };
        
        process.Start();
        return process.StandardOutput.ReadToEnd();
    }
}
```

**Problems:**
- Shell metacharacters enable arbitrary command execution
- No validation of command arguments
- Using shell (`/bin/bash`, `cmd.exe`) expands attack surface
- No compile-time detection
- Can execute destructive commands

### ✅ After

```csharp
/// <summary>
/// A validated command argument that cannot contain shell metacharacters.
/// Cannot be constructed from untrusted strings without validation.
/// </summary>
public sealed record SafeCommandArgument
{
    public string Value { get; }
    
    private SafeCommandArgument(string value) => Value = value;
    
    // Shell metacharacters that enable command injection
    private static readonly char[] DangerousCharacters = 
    { 
        '|', '&', ';', '\n', '\r', '$', '`', '\\', '"', '\'', 
        '<', '>', '!', '*', '?', '{', '}', '[', ']', '(', ')', '~'
    };
    
    public static Result<SafeCommandArgument, string> Create(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            return Result<SafeCommandArgument, string>.Failure(
                "Command argument cannot be empty");
        
        // Check for dangerous characters
        if (value.IndexOfAny(DangerousCharacters) >= 0)
            return Result<SafeCommandArgument, string>.Failure(
                "Command argument contains dangerous shell metacharacters");
        
        // Check for null bytes (can truncate arguments)
        if (value.Contains('\0'))
            return Result<SafeCommandArgument, string>.Failure(
                "Command argument contains null bytes");
        
        return Result<SafeCommandArgument, string>.Success(
            new SafeCommandArgument(value));
    }
    
    /// <summary>
    /// Creates an argument from a file path that has already been validated.
    /// </summary>
    public static SafeCommandArgument FromSafePath(SafeFilePath path)
    {
        // SafeFilePath is already validated, so we trust it
        return new SafeCommandArgument(path.FullPath);
    }
}

/// <summary>
/// An executable command with validated arguments.
/// Executes directly without using a shell.
/// </summary>
public sealed record SafeCommand
{
    public string ExecutablePath { get; }
    public IReadOnlyList<SafeCommandArgument> Arguments { get; }
    
    private SafeCommand(string executablePath, IReadOnlyList<SafeCommandArgument> arguments)
    {
        ExecutablePath = executablePath;
        Arguments = arguments;
    }
    
    public static Result<SafeCommand, string> Create(
        string executablePath,
        params SafeCommandArgument[] arguments)
    {
        if (string.IsNullOrWhiteSpace(executablePath))
            return Result<SafeCommand, string>.Failure("Executable path cannot be empty");
        
        if (!File.Exists(executablePath))
            return Result<SafeCommand, string>.Failure(
                $"Executable not found: {executablePath}");
        
        return Result<SafeCommand, string>.Success(
            new SafeCommand(executablePath, arguments.ToList().AsReadOnly()));
    }
    
    /// <summary>
    /// Creates a command from a known executable name (searches PATH).
    /// </summary>
    public static Result<SafeCommand, string> FromExecutable(
        string executableName,
        params SafeCommandArgument[] arguments)
    {
        var executablePath = FindExecutable(executableName);
        
        if (executablePath == null)
            return Result<SafeCommand, string>.Failure(
                $"Executable '{executableName}' not found in PATH");
        
        return Create(executablePath, arguments);
    }
    
    private static string? FindExecutable(string name)
    {
        var pathEnv = Environment.GetEnvironmentVariable("PATH");
        if (pathEnv == null) return null;
        
        var paths = pathEnv.Split(Path.PathSeparator);
        
        foreach (var path in paths)
        {
            var fullPath = Path.Combine(path, name);
            if (File.Exists(fullPath))
                return fullPath;
            
            // Windows: try with .exe extension
            if (OperatingSystem.IsWindows())
            {
                var withExe = fullPath + ".exe";
                if (File.Exists(withExe))
                    return withExe;
            }
        }
        
        return null;
    }
}

/// <summary>
/// Result of executing a command.
/// </summary>
public sealed record CommandResult
{
    public int ExitCode { get; }
    public string StandardOutput { get; }
    public string StandardError { get; }
    public bool IsSuccess => ExitCode == 0;
    
    internal CommandResult(int exitCode, string stdout, string stderr)
    {
        ExitCode = exitCode;
        StandardOutput = stdout;
        StandardError = stderr;
    }
}

/// <summary>
/// Service for safely executing external commands.
/// </summary>
public class SafeProcessExecutor
{
    public Result<CommandResult, string> Execute(
        SafeCommand command,
        TimeSpan? timeout = null)
    {
        try
        {
            var startInfo = new ProcessStartInfo
            {
                FileName = command.ExecutablePath,
                RedirectStandardOutput = true,
                RedirectStandardError = true,
                UseShellExecute = false,  // Critical: don't use shell!
                CreateNoWindow = true
            };
            
            // Add arguments without shell expansion
            foreach (var arg in command.Arguments)
            {
                startInfo.ArgumentList.Add(arg.Value);
            }
            
            using var process = new Process { StartInfo = startInfo };
            
            var outputBuilder = new StringBuilder();
            var errorBuilder = new StringBuilder();
            
            process.OutputDataReceived += (_, e) =>
            {
                if (e.Data != null)
                    outputBuilder.AppendLine(e.Data);
            };
            
            process.ErrorDataReceived += (_, e) =>
            {
                if (e.Data != null)
                    errorBuilder.AppendLine(e.Data);
            };
            
            process.Start();
            process.BeginOutputReadLine();
            process.BeginErrorReadLine();
            
            var timeoutMs = timeout.HasValue 
                ? (int)timeout.Value.TotalMilliseconds 
                : Timeout.Infinite;
            
            if (!process.WaitForExit(timeoutMs))
            {
                process.Kill();
                return Result<CommandResult, string>.Failure(
                    "Command execution timed out");
            }
            
            var result = new CommandResult(
                process.ExitCode,
                outputBuilder.ToString(),
                errorBuilder.ToString());
            
            return Result<CommandResult, string>.Success(result);
        }
        catch (Exception ex)
        {
            return Result<CommandResult, string>.Failure(
                $"Failed to execute command: {ex.Message}");
        }
    }
    
    public async Task<Result<CommandResult, string>> ExecuteAsync(
        SafeCommand command,
        TimeSpan? timeout = null,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var startInfo = new ProcessStartInfo
            {
                FileName = command.ExecutablePath,
                RedirectStandardOutput = true,
                RedirectStandardError = true,
                UseShellExecute = false,
                CreateNoWindow = true
            };
            
            foreach (var arg in command.Arguments)
            {
                startInfo.ArgumentList.Add(arg.Value);
            }
            
            using var process = new Process { StartInfo = startInfo };
            
            var outputTask = new TaskCompletionSource<string>();
            var errorTask = new TaskCompletionSource<string>();
            
            var outputBuilder = new StringBuilder();
            var errorBuilder = new StringBuilder();
            
            process.OutputDataReceived += (_, e) =>
            {
                if (e.Data != null)
                    outputBuilder.AppendLine(e.Data);
                else
                    outputTask.TrySetResult(outputBuilder.ToString());
            };
            
            process.ErrorDataReceived += (_, e) =>
            {
                if (e.Data != null)
                    errorBuilder.AppendLine(e.Data);
                else
                    errorTask.TrySetResult(errorBuilder.ToString());
            };
            
            process.Start();
            process.BeginOutputReadLine();
            process.BeginErrorReadLine();
            
            var timeoutTask = timeout.HasValue
                ? Task.Delay(timeout.Value, cancellationToken)
                : Task.Delay(Timeout.Infinite, cancellationToken);
            
            var processTask = process.WaitForExitAsync(cancellationToken);
            
            var completedTask = await Task.WhenAny(processTask, timeoutTask);
            
            if (completedTask == timeoutTask)
            {
                process.Kill();
                return Result<CommandResult, string>.Failure(
                    "Command execution timed out");
            }
            
            await Task.WhenAll(outputTask.Task, errorTask.Task);
            
            var result = new CommandResult(
                process.ExitCode,
                outputBuilder.ToString(),
                errorBuilder.ToString());
            
            return Result<CommandResult, string>.Success(result);
        }
        catch (OperationCanceledException)
        {
            return Result<CommandResult, string>.Failure("Command execution cancelled");
        }
        catch (Exception ex)
        {
            return Result<CommandResult, string>.Failure(
                $"Failed to execute command: {ex.Message}");
        }
    }
}

public class ImageProcessingService
{
    private readonly SafeProcessExecutor _executor;
    private readonly string _convertExecutable;
    
    public ImageProcessingService(SafeProcessExecutor executor)
    {
        _executor = executor;
        
        // Find ImageMagick convert executable at startup
        var convertPath = SafeCommand.FindExecutable("convert");
        _convertExecutable = convertPath 
            ?? throw new InvalidOperationException("ImageMagick convert not found");
    }
    
    public Result<Unit, string> ConvertImage(SafeFilePath inputFile, SafeFilePath outputFile)
    {
        // Create validated arguments—injection impossible
        var inputArg = SafeCommandArgument.FromSafePath(inputFile);
        var outputArg = SafeCommandArgument.FromSafePath(outputFile);
        
        var commandResult = SafeCommand.Create(
            _convertExecutable,
            inputArg,
            outputArg);
        
        if (!commandResult.IsSuccess)
            return Result<Unit, string>.Failure(commandResult.Error!);
        
        var execResult = _executor.Execute(commandResult.Value!, TimeSpan.FromMinutes(5));
        
        return execResult.Match(
            onSuccess: result => result.IsSuccess
                ? Result<Unit, string>.Success(Unit.Value)
                : Result<Unit, string>.Failure($"Convert failed: {result.StandardError}"),
            onFailure: error => Result<Unit, string>.Failure(error));
    }
}
```

## Advanced Patterns

### Allowlist of Known Commands

```csharp
/// <summary>
/// Registry of allowed commands with their executable paths.
/// </summary>
public sealed class CommandRegistry
{
    private readonly Dictionary<string, string> _allowedCommands = new();
    
    public Result<Unit, string> Register(string name, string executablePath)
    {
        if (string.IsNullOrWhiteSpace(name))
            return Result<Unit, string>.Failure("Command name cannot be empty");
        
        if (!File.Exists(executablePath))
            return Result<Unit, string>.Failure(
                $"Executable not found: {executablePath}");
        
        _allowedCommands[name] = executablePath;
        return Result<Unit, string>.Success(Unit.Value);
    }
    
    public Result<string, string> GetExecutablePath(string name)
    {
        if (!_allowedCommands.TryGetValue(name, out var path))
            return Result<string, string>.Failure(
                $"Command '{name}' is not registered");
        
        return Result<string, string>.Success(path);
    }
    
    public Result<SafeCommand, string> CreateCommand(
        string commandName,
        params SafeCommandArgument[] arguments)
    {
        var pathResult = GetExecutablePath(commandName);
        
        return pathResult.Match(
            onSuccess: path => SafeCommand.Create(path, arguments),
            onFailure: error => Result<SafeCommand, string>.Failure(error));
    }
}

// Startup configuration
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        var registry = new CommandRegistry();
        
        // Register allowed commands
        registry.Register("convert", "/usr/bin/convert");
        registry.Register("ffmpeg", "/usr/bin/ffmpeg");
        registry.Register("gzip", "/usr/bin/gzip");
        
        services.AddSingleton(registry);
        services.AddSingleton<SafeProcessExecutor>();
    }
}
```

### Predefined Command Templates

```csharp
/// <summary>
/// Predefined command template with fixed structure.
/// </summary>
public abstract record CommandTemplate
{
    public abstract string CommandName { get; }
    public abstract Result<SafeCommand, string> Build(CommandRegistry registry);
}

public sealed record ConvertImageCommand(
    SafeFilePath InputFile,
    SafeFilePath OutputFile,
    ImageFormat Format) : CommandTemplate
{
    public override string CommandName => "convert";
    
    public override Result<SafeCommand, string> Build(CommandRegistry registry)
    {
        var inputArg = SafeCommandArgument.FromSafePath(InputFile);
        var outputArg = SafeCommandArgument.FromSafePath(OutputFile);
        
        // Additional format-specific arguments
        var formatArg = SafeCommandArgument.Create($"-format {Format}");
        
        if (!formatArg.IsSuccess)
            return Result<SafeCommand, string>.Failure(formatArg.Error!);
        
        return registry.CreateCommand(CommandName, inputArg, formatArg.Value!, outputArg);
    }
}

public enum ImageFormat
{
    Jpeg,
    Png,
    Gif
}

public sealed record CompressFileCommand(SafeFilePath InputFile) : CommandTemplate
{
    public override string CommandName => "gzip";
    
    public override Result<SafeCommand, string> Build(CommandRegistry registry)
    {
        var fileArg = SafeCommandArgument.FromSafePath(InputFile);
        return registry.CreateCommand(CommandName, fileArg);
    }
}

public class CommandExecutor
{
    private readonly SafeProcessExecutor _executor;
    private readonly CommandRegistry _registry;
    
    public async Task<Result<CommandResult, string>> ExecuteAsync(
        CommandTemplate template,
        CancellationToken cancellationToken = default)
    {
        var commandResult = template.Build(_registry);
        
        if (!commandResult.IsSuccess)
            return Result<CommandResult, string>.Failure(commandResult.Error!);
        
        return await _executor.ExecuteAsync(
            commandResult.Value!,
            TimeSpan.FromMinutes(10),
            cancellationToken);
    }
}
```

### Docker/Container Execution

```csharp
/// <summary>
/// Executes commands safely inside a Docker container.
/// </summary>
public sealed class DockerCommand
{
    public string Image { get; }
    public SafeCommand Command { get; }
    public Dictionary<string, string> EnvironmentVariables { get; }
    
    private DockerCommand(
        string image,
        SafeCommand command,
        Dictionary<string, string> envVars)
    {
        Image = image;
        Command = command;
        EnvironmentVariables = envVars;
    }
    
    public static Result<DockerCommand, string> Create(
        string image,
        SafeCommand command,
        Dictionary<string, string>? envVars = null)
    {
        if (string.IsNullOrWhiteSpace(image))
            return Result<DockerCommand, string>.Failure("Docker image cannot be empty");
        
        return Result<DockerCommand, string>.Success(
            new DockerCommand(image, command, envVars ?? new()));
    }
    
    public SafeCommand ToDockerCommand()
    {
        var args = new List<SafeCommandArgument>
        {
            SafeCommandArgument.Create("run").Value!,
            SafeCommandArgument.Create("--rm").Value!,
            SafeCommandArgument.Create("--network=none").Value!  // Isolate network
        };
        
        // Add environment variables
        foreach (var (key, value) in EnvironmentVariables)
        {
            var envArg = SafeCommandArgument.Create($"-e").Value!;
            var envValue = SafeCommandArgument.Create($"{key}={value}").Value!;
            args.Add(envArg);
            args.Add(envValue);
        }
        
        // Add image
        args.Add(SafeCommandArgument.Create(Image).Value!);
        
        // Add command arguments
        args.AddRange(Command.Arguments);
        
        return SafeCommand.Create("/usr/bin/docker", args.ToArray()).Value!;
    }
}

public class IsolatedExecutor
{
    private readonly SafeProcessExecutor _executor;
    
    public async Task<Result<CommandResult, string>> ExecuteInContainerAsync(
        DockerCommand dockerCommand,
        CancellationToken cancellationToken = default)
    {
        var command = dockerCommand.ToDockerCommand();
        
        return await _executor.ExecuteAsync(
            command,
            TimeSpan.FromMinutes(30),
            cancellationToken);
    }
}
```

## Testing

```csharp
public class CommandInjectionTests
{
    [Fact]
    public void SafeCommandArgument_WithShellMetacharacters_Fails()
    {
        var maliciousInput = "file.txt; rm -rf /";
        
        var result = SafeCommandArgument.Create(maliciousInput);
        
        Assert.False(result.IsSuccess);
        Assert.Contains("metacharacters", result.Error!.ToLower());
    }
    
    [Fact]
    public void SafeCommandArgument_WithValidInput_Succeeds()
    {
        var validInput = "myfile.txt";
        
        var result = SafeCommandArgument.Create(validInput);
        
        Assert.True(result.IsSuccess);
        Assert.Equal(validInput, result.Value!.Value);
    }
    
    [Fact]
    public void SafeProcessExecutor_DoesNotUseShell()
    {
        var executor = new SafeProcessExecutor();
        var command = SafeCommand.FromExecutable("echo", 
            SafeCommandArgument.Create("hello").Value!).Value!;
        
        var result = executor.Execute(command);
        
        Assert.True(result.IsSuccess);
        Assert.Contains("hello", result.Value!.StandardOutput);
    }
    
    [Fact]
    public void SafeProcessExecutor_WithTimeout_TimesOut()
    {
        var executor = new SafeProcessExecutor();
        var command = SafeCommand.FromExecutable("sleep",
            SafeCommandArgument.Create("10").Value!).Value!;
        
        var result = executor.Execute(command, TimeSpan.FromSeconds(1));
        
        Assert.False(result.IsSuccess);
        Assert.Contains("timeout", result.Error!.ToLower());
    }
}
```

## Why It's a Problem

1. **Arbitrary command execution**: Attackers can run any command on the system
2. **Shell metacharacters**: `;`, `|`, `&&`, etc. enable command chaining
3. **System compromise**: Can read files, install malware, create backdoors
4. **Data destruction**: `rm -rf` commands can destroy data
5. **No compile-time detection**: String-based process execution looks innocent

## Symptoms

- Using `/bin/sh`, `/bin/bash`, or `cmd.exe` to execute commands
- String concatenation or interpolation in command arguments
- `Process.Start()` with unsanitized user input
- No validation of command arguments
- Shell metacharacters in command strings

## Benefits

- **Injection-proof**: Direct execution without shell prevents injection
- **Compile-time safety**: Type system enforces argument validation
- **Self-documenting**: `SafeCommand` makes security constraints explicit
- **Timeout support**: Prevents denial-of-service via long-running commands
- **Allowlist enforcement**: Only registered commands can be executed

## Trade-offs

- **More verbose**: Command construction requires explicit validation
- **Limited shell features**: Cannot use pipes, redirects, or other shell features
- **Command registration**: Allowed commands must be configured at startup
- **Path dependencies**: Executable paths may differ across environments

## See Also

- [Input Sanitization](./input-sanitization.md) — trusted types at boundaries
- [Path Traversal Prevention](./path-traversal-prevention.md) — safe file path handling
- [SQL Injection Prevention](./sql-injection-prevention.md) — parameterized queries
- [Type-Safe String Interpolation](./type-safe-string-interpolation.md) — preventing injection
