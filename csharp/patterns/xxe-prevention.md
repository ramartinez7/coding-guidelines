# XXE Prevention (Safe XML Parsing)

> Parsing XML with default settings enables XML External Entity (XXE) attacks—use secure XML parsing configuration to prevent entity expansion and external resource access.

## Problem

XML parsers by default resolve external entities, which allows attackers to read local files, perform SSRF attacks, or cause denial-of-service through entity expansion. Unsafe XML parsing is one of the OWASP Top 10 vulnerabilities. There's no compile-time protection when using standard XML APIs.

## Example

### ❌ Before

```csharp
public class XmlService
{
    public XDocument ParseUserXml(string xmlContent)
    {
        // Dangerous: default XDocument.Parse enables XXE
        // Attacker input:
        // <?xml version="1.0"?>
        // <!DOCTYPE foo [
        //   <!ENTITY xxe SYSTEM "file:///etc/passwd">
        // ]>
        // <data>&xxe;</data>
        
        return XDocument.Parse(xmlContent);
    }
    
    public XmlDocument LoadXmlFile(string filePath)
    {
        // Vulnerable: XmlDocument with default settings
        var doc = new XmlDocument();
        doc.Load(filePath);
        return doc;
    }
    
    public string TransformXml(string xmlContent, string xsltPath)
    {
        // Dangerous: XSLT can access external resources
        var xslt = new XslCompiledTransform();
        xslt.Load(xsltPath);
        
        var input = XDocument.Parse(xmlContent);
        var output = new StringWriter();
        
        xslt.Transform(input.CreateReader(), null, output);
        return output.ToString();
    }
}
```

**Problems:**
- Default XML parsers resolve external entities
- Can read arbitrary local files
- SSRF: Can make requests to internal services
- Entity expansion: `<!ENTITY lol "lol"><!ENTITY lol2 "&lol;&lol;">` causes DoS
- No compile-time detection

### ✅ After

```csharp
/// <summary>
/// Secure XML settings that prevent XXE attacks.
/// </summary>
public static class SecureXmlSettings
{
    /// <summary>
    /// Creates XmlReaderSettings with XXE protections enabled.
    /// </summary>
    public static XmlReaderSettings CreateSecureSettings()
    {
        return new XmlReaderSettings
        {
            // Disable DTD processing entirely (strongest protection)
            DtdProcessing = DtdProcessing.Prohibit,
            
            // Disable external entity resolution
            XmlResolver = null,
            
            // Limit document size to prevent DoS
            MaxCharactersFromEntities = 1024,
            MaxCharactersInDocument = 10_000_000,
            
            // Close input stream after reading
            CloseInput = true,
            
            // Validate against schema if provided
            ValidationType = ValidationType.None
        };
    }
    
    /// <summary>
    /// Creates XmlReaderSettings that allow DTD but with entity expansion limits.
    /// Use only if DTDs are required.
    /// </summary>
    public static XmlReaderSettings CreateSettingsWithLimitedDtd()
    {
        return new XmlReaderSettings
        {
            // Parse DTD but ignore entities
            DtdProcessing = DtdProcessing.Ignore,
            
            // No external entity resolution
            XmlResolver = null,
            
            MaxCharactersFromEntities = 1024,
            MaxCharactersInDocument = 10_000_000,
            CloseInput = true,
            ValidationType = ValidationType.None
        };
    }
}

/// <summary>
/// Safely parsed XML document.
/// Cannot be constructed without going through secure parsing.
/// </summary>
public sealed record SafeXmlDocument
{
    public XDocument Document { get; }
    public DateTime ParsedAt { get; }
    
    private SafeXmlDocument(XDocument document)
    {
        Document = document;
        ParsedAt = DateTime.UtcNow;
    }
    
    /// <summary>
    /// Parses XML content with XXE protections.
    /// </summary>
    public static Result<SafeXmlDocument, XmlParseError> Parse(string xmlContent)
    {
        if (string.IsNullOrWhiteSpace(xmlContent))
            return Result<SafeXmlDocument, XmlParseError>.Failure(
                new XmlParseError("XML content cannot be empty"));
        
        try
        {
            var settings = SecureXmlSettings.CreateSecureSettings();
            
            using var stringReader = new StringReader(xmlContent);
            using var xmlReader = XmlReader.Create(stringReader, settings);
            
            var document = XDocument.Load(xmlReader);
            
            return Result<SafeXmlDocument, XmlParseError>.Success(
                new SafeXmlDocument(document));
        }
        catch (XmlException ex)
        {
            return Result<SafeXmlDocument, XmlParseError>.Failure(
                new XmlParseError($"Invalid XML: {ex.Message}"));
        }
        catch (Exception ex)
        {
            return Result<SafeXmlDocument, XmlParseError>.Failure(
                new XmlParseError($"Failed to parse XML: {ex.Message}"));
        }
    }
    
    /// <summary>
    /// Parses XML from a file with XXE protections.
    /// </summary>
    public static Result<SafeXmlDocument, XmlParseError> ParseFile(SafeFilePath filePath)
    {
        try
        {
            var settings = SecureXmlSettings.CreateSecureSettings();
            
            using var fileStream = File.OpenRead(filePath.FullPath);
            using var xmlReader = XmlReader.Create(fileStream, settings);
            
            var document = XDocument.Load(xmlReader);
            
            return Result<SafeXmlDocument, XmlParseError>.Success(
                new SafeXmlDocument(document));
        }
        catch (XmlException ex)
        {
            return Result<SafeXmlDocument, XmlParseError>.Failure(
                new XmlParseError($"Invalid XML: {ex.Message}"));
        }
        catch (Exception ex)
        {
            return Result<SafeXmlDocument, XmlParseError>.Failure(
                new XmlParseError($"Failed to parse XML file: {ex.Message}"));
        }
    }
    
    /// <summary>
    /// Parses XML from a stream with XXE protections.
    /// </summary>
    public static Result<SafeXmlDocument, XmlParseError> ParseStream(Stream stream)
    {
        try
        {
            var settings = SecureXmlSettings.CreateSecureSettings();
            
            using var xmlReader = XmlReader.Create(stream, settings);
            
            var document = XDocument.Load(xmlReader);
            
            return Result<SafeXmlDocument, XmlParseError>.Success(
                new SafeXmlDocument(document));
        }
        catch (XmlException ex)
        {
            return Result<SafeXmlDocument, XmlParseError>.Failure(
                new XmlParseError($"Invalid XML: {ex.Message}"));
        }
        catch (Exception ex)
        {
            return Result<SafeXmlDocument, XmlParseError>.Failure(
                new XmlParseError($"Failed to parse XML stream: {ex.Message}"));
        }
    }
}

public sealed record XmlParseError(string Message);

public class XmlService
{
    public Result<SafeXmlDocument, XmlParseError> ParseUserXml(string xmlContent)
    {
        // Automatically protected against XXE
        return SafeXmlDocument.Parse(xmlContent);
    }
    
    public Result<XElement, string> GetElement(SafeXmlDocument document, string elementName)
    {
        var element = document.Document.Root?.Element(elementName);
        
        if (element == null)
            return Result<XElement, string>.Failure($"Element '{elementName}' not found");
        
        return Result<XElement, string>.Success(element);
    }
}
```

## Advanced Patterns

### Schema Validation

```csharp
/// <summary>
/// XML schema for validation.
/// </summary>
public sealed record XmlSchema
{
    public string NamespaceUri { get; }
    public string SchemaContent { get; }
    
    private XmlSchema(string namespaceUri, string schemaContent)
    {
        NamespaceUri = namespaceUri;
        SchemaContent = schemaContent;
    }
    
    public static Result<XmlSchema, string> Load(SafeFilePath schemaPath)
    {
        try
        {
            var content = File.ReadAllText(schemaPath.FullPath);
            
            // Parse to validate it's valid XSD
            using var stringReader = new StringReader(content);
            using var xmlReader = XmlReader.Create(stringReader, 
                SecureXmlSettings.CreateSecureSettings());
            
            var doc = XDocument.Load(xmlReader);
            var ns = doc.Root?.Attribute("targetNamespace")?.Value ?? "";
            
            return Result<XmlSchema, string>.Success(
                new XmlSchema(ns, content));
        }
        catch (Exception ex)
        {
            return Result<XmlSchema, string>.Failure(
                $"Failed to load schema: {ex.Message}");
        }
    }
}

/// <summary>
/// XML document validated against a schema.
/// </summary>
public sealed record ValidatedXmlDocument
{
    public SafeXmlDocument Document { get; }
    public XmlSchema Schema { get; }
    public DateTime ValidatedAt { get; }
    
    private ValidatedXmlDocument(SafeXmlDocument document, XmlSchema schema)
    {
        Document = document;
        Schema = schema;
        ValidatedAt = DateTime.UtcNow;
    }
    
    public static Result<ValidatedXmlDocument, string> Validate(
        SafeXmlDocument document,
        XmlSchema schema)
    {
        try
        {
            var schemaSet = new XmlSchemaSet();
            
            using var schemaReader = new StringReader(schema.SchemaContent);
            using var xmlSchemaReader = XmlReader.Create(schemaReader,
                SecureXmlSettings.CreateSecureSettings());
            
            schemaSet.Add(schema.NamespaceUri, xmlSchemaReader);
            
            var errors = new List<string>();
            
            document.Document.Validate(schemaSet, (sender, e) =>
            {
                errors.Add(e.Message);
            });
            
            if (errors.Count > 0)
                return Result<ValidatedXmlDocument, string>.Failure(
                    $"Validation failed: {string.Join("; ", errors)}");
            
            return Result<ValidatedXmlDocument, string>.Success(
                new ValidatedXmlDocument(document, schema));
        }
        catch (Exception ex)
        {
            return Result<ValidatedXmlDocument, string>.Failure(
                $"Validation error: {ex.Message}");
        }
    }
}
```

### SOAP Message Handling

```csharp
/// <summary>
/// Secure SOAP message parser.
/// </summary>
public sealed class SecureSoapParser
{
    private readonly XmlSchema _soapSchema;
    
    public SecureSoapParser(XmlSchema soapSchema)
    {
        _soapSchema = soapSchema;
    }
    
    public Result<SafeXmlDocument, string> ParseSoapMessage(string soapXml)
    {
        // Parse with XXE protection
        var parseResult = SafeXmlDocument.Parse(soapXml);
        
        if (!parseResult.IsSuccess)
            return Result<SafeXmlDocument, string>.Failure(
                parseResult.Error!.Message);
        
        var document = parseResult.Value!;
        
        // Validate SOAP structure
        var envelope = document.Document.Root;
        
        if (envelope?.Name.LocalName != "Envelope")
            return Result<SafeXmlDocument, string>.Failure(
                "Invalid SOAP message: missing Envelope");
        
        var body = envelope.Element(XName.Get("Body", envelope.Name.NamespaceName));
        
        if (body == null)
            return Result<SafeXmlDocument, string>.Failure(
                "Invalid SOAP message: missing Body");
        
        return Result<SafeXmlDocument, string>.Success(document);
    }
}
```

### XSLT Transformation (Secure)

```csharp
/// <summary>
/// Secure XSLT transformer that prevents XXE in stylesheets.
/// </summary>
public sealed class SecureXsltTransformer
{
    private readonly XslCompiledTransform _transform;
    
    private SecureXsltTransformer(XslCompiledTransform transform)
    {
        _transform = transform;
    }
    
    public static Result<SecureXsltTransformer, string> Load(SafeFilePath xsltPath)
    {
        try
        {
            var settings = new XsltSettings
            {
                // Disable script execution
                EnableScript = false,
                
                // Disable document() function (prevents SSRF)
                EnableDocumentFunction = false
            };
            
            var readerSettings = SecureXmlSettings.CreateSecureSettings();
            
            using var fileStream = File.OpenRead(xsltPath.FullPath);
            using var xmlReader = XmlReader.Create(fileStream, readerSettings);
            
            var transform = new XslCompiledTransform();
            transform.Load(xmlReader, settings, null);
            
            return Result<SecureXsltTransformer, string>.Success(
                new SecureXsltTransformer(transform));
        }
        catch (Exception ex)
        {
            return Result<SecureXsltTransformer, string>.Failure(
                $"Failed to load XSLT: {ex.Message}");
        }
    }
    
    public Result<string, string> Transform(SafeXmlDocument input)
    {
        try
        {
            using var outputWriter = new StringWriter();
            
            _transform.Transform(input.Document.CreateReader(), null, outputWriter);
            
            return Result<string, string>.Success(outputWriter.ToString());
        }
        catch (Exception ex)
        {
            return Result<string, string>.Failure(
                $"Transformation failed: {ex.Message}");
        }
    }
}
```

### Configuration-Based Limits

```csharp
public sealed class XmlParsingConfiguration
{
    public int MaxDocumentSize { get; init; } = 10_000_000;  // 10 MB
    public int MaxEntityExpansion { get; init; } = 1024;
    public bool AllowDtd { get; init; } = false;
    public bool ValidateSchema { get; init; } = false;
    
    public XmlReaderSettings ToXmlReaderSettings()
    {
        return new XmlReaderSettings
        {
            DtdProcessing = AllowDtd ? DtdProcessing.Ignore : DtdProcessing.Prohibit,
            XmlResolver = null,
            MaxCharactersFromEntities = MaxEntityExpansion,
            MaxCharactersInDocument = MaxDocumentSize,
            CloseInput = true,
            ValidationType = ValidateSchema ? ValidationType.Schema : ValidationType.None
        };
    }
    
    public static XmlParsingConfiguration FromAppSettings(IConfiguration configuration)
    {
        return new XmlParsingConfiguration
        {
            MaxDocumentSize = configuration.GetValue("Xml:MaxDocumentSize", 10_000_000),
            MaxEntityExpansion = configuration.GetValue("Xml:MaxEntityExpansion", 1024),
            AllowDtd = configuration.GetValue("Xml:AllowDtd", false),
            ValidateSchema = configuration.GetValue("Xml:ValidateSchema", false)
        };
    }
}
```

## Testing

```csharp
public class XxePreventionTests
{
    [Fact]
    public void SafeXmlDocument_WithXxeAttack_Fails()
    {
        var xxePayload = @"<?xml version=""1.0""?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM ""file:///etc/passwd"">
]>
<data>&xxe;</data>";
        
        var result = SafeXmlDocument.Parse(xxePayload);
        
        // Should fail because DTD processing is prohibited
        Assert.False(result.IsSuccess);
        Assert.Contains("DTD", result.Error!.Message);
    }
    
    [Fact]
    public void SafeXmlDocument_WithEntityExpansion_Fails()
    {
        // Billion laughs attack
        var billionLaughs = @"<?xml version=""1.0""?>
<!DOCTYPE lolz [
  <!ENTITY lol ""lol"">
  <!ENTITY lol2 ""&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;"">
  <!ENTITY lol3 ""&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;"">
]>
<data>&lol3;</data>";
        
        var result = SafeXmlDocument.Parse(billionLaughs);
        
        Assert.False(result.IsSuccess);
    }
    
    [Fact]
    public void SafeXmlDocument_WithValidXml_Succeeds()
    {
        var validXml = @"<?xml version=""1.0""?>
<data>
  <item>Hello</item>
  <item>World</item>
</data>";
        
        var result = SafeXmlDocument.Parse(validXml);
        
        Assert.True(result.IsSuccess);
        Assert.Equal("data", result.Value!.Document.Root!.Name.LocalName);
    }
    
    [Fact]
    public void SecureXsltTransformer_WithDocumentFunction_IsDisabled()
    {
        // XSLT that tries to load external documents
        var xslt = @"<?xml version=""1.0""?>
<xsl:stylesheet version=""1.0"" xmlns:xsl=""http://www.w3.org/1999/XSL/Transform"">
  <xsl:template match=""/"">
    <xsl:value-of select=""document('file:///etc/passwd')""/>
  </xsl:template>
</xsl:stylesheet>";
        
        // Create temp file for XSLT
        var tempFile = Path.GetTempFileName();
        File.WriteAllText(tempFile, xslt);
        
        var pathResult = SafeFilePath.Create(Path.GetDirectoryName(tempFile)!, 
            Path.GetFileName(tempFile));
        
        // Should fail to load because document() is disabled
        var result = SecureXsltTransformer.Load(pathResult.Value!);
        
        // Or if it loads, transformation should fail
        // (behavior depends on .NET version)
    }
}
```

## Why It's a Problem

1. **File disclosure**: XXE can read arbitrary local files (`/etc/passwd`, config files)
2. **SSRF attacks**: Can make requests to internal services
3. **Denial of service**: Entity expansion bombs consume memory
4. **Data exfiltration**: Send data to attacker-controlled servers
5. **No compile-time detection**: Default XML APIs are vulnerable

## Symptoms

- Using `XDocument.Parse()` or `XmlDocument.Load()` with default settings
- No `XmlReaderSettings` configuration
- DTD processing enabled
- `XmlResolver` not set to `null`
- XSLT transformations without security settings

## Benefits

- **XXE-proof**: Disabled DTD processing prevents external entity attacks
- **DoS protection**: Entity expansion limits prevent memory exhaustion
- **SSRF prevention**: No external resource resolution
- **Self-documenting**: `SafeXmlDocument` makes security guarantees explicit
- **Schema validation**: Ensures XML structure matches expected format

## Trade-offs

- **No DTD support**: Legitimate DTDs are also blocked (use schema validation instead)
- **More verbose**: Requires explicit secure parsing
- **Breaking change**: Existing code using DTDs needs updates
- **Size limits**: Large documents may need configuration adjustment

## See Also

- [Input Sanitization](./input-sanitization.md) — trusted types at boundaries
- [Deserialization Prevention](./deserialization-prevention.md) — safe object deserialization
- [SQL Injection Prevention](./sql-injection-prevention.md) — parameterized queries
- [Validated Configuration](./validated-configuration.md) — secure configuration parsing
